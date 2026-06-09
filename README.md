-- ============================================================
-- 06A - AI summaries
-- Adds AI-generated 1-3 paragraph summaries for digest items
-- Version v3: headline-locked summaries
--
-- Purpose:
-- - Prevents summaries from mixing multiple newsletter stories
-- - Focuses only on the selected headline
-- - Keeps the feedback wording from senior analyst
-- ============================================================

CREATE TABLE IF NOT EXISTS PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES (
    SUMMARY_KEY STRING,
    SUMMARY_VERSION STRING,
    ARTICLE_TITLE STRING,
    ARTICLE_URL STRING,
    EMAIL_SOURCE_TYPE STRING,
    PUBLISH_DATE DATE,
    PUBLISH_DATE_SOURCE STRING,
    AI_SUMMARY STRING,
    CREATED_AT TIMESTAMP_TZ DEFAULT CURRENT_TIMESTAMP()
);

ALTER TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES
ADD COLUMN IF NOT EXISTS SUMMARY_VERSION STRING;

ALTER TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES
ADD COLUMN IF NOT EXISTS PUBLISH_DATE DATE;

ALTER TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES
ADD COLUMN IF NOT EXISTS PUBLISH_DATE_SOURCE STRING;


CREATE OR REPLACE PROCEDURE PHARMA_NEWS_SANDBOX.NEWS.SP_REFRESH_AI_SUMMARIES()
RETURNS STRING
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
    V_NEW_SUMMARIES NUMBER DEFAULT 0;
BEGIN

    CREATE OR REPLACE TEMP TABLE TMP_AI_SUMMARY_CANDIDATES AS
    SELECT
        SHA2(
            LOWER(
                COALESCE(D.ARTICLE_URL, '') || '|' ||
                COALESCE(D.ARTICLE_TITLE, '')
            ),
            256
        ) AS SUMMARY_KEY,

        'v3_headline_locked_1_3_paragraphs' AS SUMMARY_VERSION,

        D.ARTICLE_TITLE,
        D.ARTICLE_URL,
        D.EMAIL_SOURCE_TYPE,
        D.PUBLISH_DATE,
        D.PUBLISH_DATE_SOURCE,
        D.PRIORITY_TIER,
        D.SIGNAL_SCORE,
        D.SIGNAL_REASONS,
        D.MATCHED_COMPANIES,
        D.MATCHED_COMPANY_CATEGORIES,
        D.ARTICLE_SUMMARY_INPUT

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2 D

    LEFT JOIN PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES S
        ON SHA2(
            LOWER(
                COALESCE(D.ARTICLE_URL, '') || '|' ||
                COALESCE(D.ARTICLE_TITLE, '')
            ),
            256
        ) = S.SUMMARY_KEY
       AND S.SUMMARY_VERSION = 'v3_headline_locked_1_3_paragraphs'

    WHERE D.PRIORITY_TIER IN ('VERY_IMPORTANT', 'IMPORTANT')
      AND D.RECEIVED_TS_PARSED >= DATEADD('day', -7, CURRENT_TIMESTAMP())
      AND S.SUMMARY_KEY IS NULL
      AND D.ARTICLE_TITLE IS NOT NULL;


    SELECT COUNT(*)
    INTO :V_NEW_SUMMARIES
    FROM TMP_AI_SUMMARY_CANDIDATES;


    INSERT INTO PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES (
        SUMMARY_KEY,
        SUMMARY_VERSION,
        ARTICLE_TITLE,
        ARTICLE_URL,
        EMAIL_SOURCE_TYPE,
        PUBLISH_DATE,
        PUBLISH_DATE_SOURCE,
        AI_SUMMARY,
        CREATED_AT
    )
    SELECT
        SUMMARY_KEY,
        SUMMARY_VERSION,
        ARTICLE_TITLE,
        ARTICLE_URL,
        EMAIL_SOURCE_TYPE,
        PUBLISH_DATE,
        PUBLISH_DATE_SOURCE,

        AI_COMPLETE(
            'llama3.3-70b',
            CONCAT(
                'You are summarizing pharmaceutical and biopharmaceutical industry news for a Business Intelligence & Insights team at a biologics/CDMO company. ',

                'Summarize only the news item that matches the selected headline below in 1-3 paragraphs. ',
                'The source text may contain multiple newsletter items, article teasers, links, advertisements, author bios, or unrelated stories. ',
                'Ignore all unrelated stories, even if they mention the same company, same disease area, same event type, or same industry. ',
                'Do not include information from other headlines, other article teasers, related news sections, or surrounding newsletter content. ',

                'The summary should focus only on specifics related to the selected headline: monetary value, capability expansion or reduction, and how the change is expected to shape the industry. ',

                'Use the provided article text as the primary source, but only use details that clearly belong to the selected headline. ',
                'Do not simply restate the headline. ',
                'Do not invent facts, figures, manufacturing relevance, CDMO relevance, competitive implications, or strategic implications that are not supported by the source text. ',

                'When available for the selected headline, include specific company names, deal values, investment amounts, facility or site names, geography, modality, capacity, product or asset names, disease area, and timeline. ',

                'If no monetary value is stated for the selected headline, explicitly say that no specific monetary value was stated. ',
                'If no capability expansion or reduction is stated for the selected headline, explicitly say that no specific capability change was stated. ',
                'If the source text does not provide enough detail beyond the headline, say that limited detail was available in the provided source text rather than adding context from other newsletter items. ',

                'Write in a professional, concise style suitable for senior stakeholders. ',
                'No bullet points. No quotation marks. ',
                'Do not use phrases such as In related news, Additionally, Separately, or In other industry news unless that information clearly belongs to the selected headline. ',

                'Selected headline: ', COALESCE(ARTICLE_TITLE, ''), '. ',
                'Source: ', COALESCE(EMAIL_SOURCE_TYPE, ''), '. ',
                'Publish date: ', COALESCE(TO_VARCHAR(PUBLISH_DATE), 'Unknown'), '. ',
                'Matched companies: ', COALESCE(MATCHED_COMPANIES, ''), '. ',
                'Company categories: ', COALESCE(MATCHED_COMPANY_CATEGORIES, ''), '. ',
                'Signal reasons: ', COALESCE(SIGNAL_REASONS, ''), '. ',

                'Source text: ',
                COALESCE(LEFT(ARTICLE_SUMMARY_INPUT, 12000), '')
            ),
            {
                'temperature': 0,
                'max_tokens': 800
            }
        ) AS AI_SUMMARY,

        CURRENT_TIMESTAMP() AS CREATED_AT

    FROM TMP_AI_SUMMARY_CANDIDATES;


    RETURN 'AI summaries refreshed. New v3 headline-locked summaries created: ' || V_NEW_SUMMARIES;

END;
$$;

-- ============================================================
-- 07 - Professional HTML email output view
-- Uses AI summaries + AI story keys
-- Result: Top 10 unique stories
--
-- Version update:
-- - Uses v3 headline-locked summaries
-- - Shows date field
-- - If true article publish date exists, it says Publish date
-- - If only email fallback exists, it says Email date
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_ALTERYX_DAILY_PHARMA_NEWS_EMAIL AS
WITH ITEMS_RAW AS (
    SELECT
        D.RECEIVED_TS_PARSED,
        D.PUBLISH_DATE,
        D.PUBLISH_DATE_SOURCE,
        D.EMAIL_SOURCE_TYPE,
        D.ARTICLE_TITLE,
        D.ARTICLE_URL,
        D.PRIORITY_TIER,
        D.SIGNAL_SCORE,
        D.SIGNAL_REASONS,
        D.MATCHED_COMPANIES,
        D.MATCHED_COMPANY_CATEGORIES,

        S.AI_SUMMARY,
        K.AI_STORY_KEY,

        SHA2(
            LOWER(
                COALESCE(D.ARTICLE_URL, '') || '|' ||
                COALESCE(D.ARTICLE_TITLE, '')
            ),
            256
        ) AS ITEM_KEY,

        CASE
            WHEN D.PRIORITY_TIER = 'VERY_IMPORTANT' THEN 1
            WHEN D.PRIORITY_TIER = 'IMPORTANT' THEN 2
            ELSE 3
        END AS PRIORITY_SORT

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2 D

    LEFT JOIN PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES S
        ON SHA2(
            LOWER(
                COALESCE(D.ARTICLE_URL, '') || '|' ||
                COALESCE(D.ARTICLE_TITLE, '')
            ),
            256
        ) = S.SUMMARY_KEY
       AND S.SUMMARY_VERSION = 'v3_headline_locked_1_3_paragraphs'

    LEFT JOIN PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_STORY_KEYS K
        ON SHA2(
            LOWER(
                COALESCE(D.ARTICLE_URL, '') || '|' ||
                COALESCE(D.ARTICLE_TITLE, '')
            ),
            256
        ) = K.STORY_ITEM_KEY

    WHERE D.RECEIVED_TS_PARSED >= DATEADD('day', -7, CURRENT_TIMESTAMP())
      AND D.PRIORITY_TIER IN ('VERY_IMPORTANT', 'IMPORTANT')
),

ITEMS_WITH_DEDUPE_KEY AS (
    SELECT
        *,
        COALESCE(NULLIF(AI_STORY_KEY, ''), ITEM_KEY) AS FINAL_STORY_KEY
    FROM ITEMS_RAW
),

ITEMS_DEDUPED AS (
    SELECT *
    FROM ITEMS_WITH_DEDUPE_KEY
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY FINAL_STORY_KEY
        ORDER BY
            PRIORITY_SORT,
            SIGNAL_SCORE DESC,
            RECEIVED_TS_PARSED DESC NULLS LAST,

            CASE
                WHEN AI_SUMMARY IS NOT NULL THEN 1
                ELSE 2
            END,

            CASE
                WHEN LENGTH(ARTICLE_TITLE) BETWEEN 45 AND 150 THEN 1
                WHEN LENGTH(ARTICLE_TITLE) BETWEEN 30 AND 170 THEN 2
                ELSE 3
            END,

            LENGTH(ARTICLE_TITLE) DESC
    ) = 1
),

ITEMS AS (
    SELECT
        *,

        REPLACE(REPLACE(REPLACE(COALESCE(ARTICLE_TITLE, ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;') AS ARTICLE_TITLE_HTML,
        REPLACE(REPLACE(REPLACE(COALESCE(EMAIL_SOURCE_TYPE, ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;') AS EMAIL_SOURCE_TYPE_HTML,
        REPLACE(REPLACE(REPLACE(COALESCE(ARTICLE_URL, ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;') AS ARTICLE_URL_HTML,
        REPLACE(REPLACE(REPLACE(COALESCE(AI_SUMMARY, ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;') AS AI_SUMMARY_HTML,

        COALESCE(TO_VARCHAR(PUBLISH_DATE, 'YYYY-MM-DD'), 'Unknown') AS DATE_HTML,

        CASE
            WHEN PUBLISH_DATE_SOURCE = 'Article publish date'
                THEN 'Publish date'
            WHEN PUBLISH_DATE_SOURCE = 'Email received date fallback'
                THEN 'Email date'
            ELSE 'Date'
        END AS DATE_LABEL_HTML,

        REPLACE(REPLACE(REPLACE(COALESCE(PUBLISH_DATE_SOURCE, ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;') AS DATE_SOURCE_HTML,

        CASE
            WHEN COALESCE(NULLIF(MATCHED_COMPANIES, ''), '') <> ''
              OR COALESCE(NULLIF(MATCHED_COMPANY_CATEGORIES, ''), '') <> ''
            THEN
                CASE
                    WHEN COALESCE(NULLIF(MATCHED_COMPANIES, ''), '') <> ''
                    THEN REPLACE(REPLACE(REPLACE(MATCHED_COMPANIES, '&', '&amp;'), '<', '&lt;'), '>', '&gt;')
                    ELSE ''
                END ||
                CASE
                    WHEN COALESCE(NULLIF(MATCHED_COMPANIES, ''), '') <> ''
                     AND COALESCE(NULLIF(MATCHED_COMPANY_CATEGORIES, ''), '') <> ''
                    THEN ' · '
                    ELSE ''
                END ||
                CASE
                    WHEN COALESCE(NULLIF(MATCHED_COMPANY_CATEGORIES, ''), '') <> ''
                    THEN REPLACE(REPLACE(REPLACE(MATCHED_COMPANY_CATEGORIES, '&', '&amp;'), '<', '&lt;'), '>', '&gt;')
                    ELSE ''
                END
            ELSE ''
        END AS TAGS_TEXT_HTML

    FROM ITEMS_DEDUPED
),

TOP_ITEMS AS (
    SELECT *
    FROM ITEMS
    ORDER BY
        PRIORITY_SORT,
        SIGNAL_SCORE DESC,
        RECEIVED_TS_PARSED DESC NULLS LAST
    LIMIT 10
),

RAW_COUNTS AS (
    SELECT
        COUNT(*) AS RAW_TOTAL_ITEMS,
        COUNT_IF(PRIORITY_TIER = 'VERY_IMPORTANT') AS RAW_VERY_IMPORTANT_ITEMS,
        COUNT_IF(PRIORITY_TIER = 'IMPORTANT') AS RAW_IMPORTANT_ITEMS
    FROM ITEMS_RAW
),

DEDUPED_COUNTS AS (
    SELECT
        COUNT(*) AS UNIQUE_STORIES,
        COUNT_IF(PRIORITY_TIER = 'VERY_IMPORTANT') AS UNIQUE_VERY_IMPORTANT_ITEMS,
        COUNT_IF(PRIORITY_TIER = 'IMPORTANT') AS UNIQUE_IMPORTANT_ITEMS
    FROM ITEMS_DEDUPED
),

EMAIL_TEXT AS (
    SELECT
        LISTAGG(
            '<tr>
                <td style="padding:16px 0; border-bottom:1px solid #e5e5e5;">

                    <div style="font-size:12px; font-weight:bold; letter-spacing:0.3px; margin-bottom:6px;">' ||
                        CASE
                            WHEN PRIORITY_TIER = 'VERY_IMPORTANT'
                            THEN '<span style="color:#b00020;">VERY IMPORTANT</span>'
                            ELSE '<span style="color:#555555;">IMPORTANT</span>'
                        END ||
                    '</div>

                    <div style="font-size:16px; font-weight:bold; line-height:1.35; margin-bottom:8px;">
                        <a href="' || ARTICLE_URL_HTML || '" style="color:#1a1a1a; text-decoration:none;">' ||
                            ARTICLE_TITLE_HTML ||
                        '</a>
                    </div>

                    <div style="font-size:13px; color:#666666; margin-bottom:8px;">
                        <b>' || DATE_LABEL_HTML || ':</b> ' || DATE_HTML || '
                        <span style="color:#999999;">(' || DATE_SOURCE_HTML || ')</span>
                    </div>' ||

                    CASE
                        WHEN COALESCE(NULLIF(AI_SUMMARY_HTML, ''), '') <> ''
                        THEN
                            '<div style="font-size:14px; color:#333333; line-height:1.5; margin-bottom:10px;">
                                <b>Summary:</b><br>' || AI_SUMMARY_HTML || '
                            </div>'
                        ELSE ''
                    END ||

                    CASE
                        WHEN COALESCE(NULLIF(TAGS_TEXT_HTML, ''), '') <> ''
                        THEN
                            '<div style="font-size:13px; color:#666666; margin-bottom:8px;">
                                <b>Tags:</b> ' || TAGS_TEXT_HTML || ' · ' || EMAIL_SOURCE_TYPE_HTML || '
                            </div>'
                        ELSE
                            '<div style="font-size:13px; color:#666666; margin-bottom:8px;">
                                <b>Source:</b> ' || EMAIL_SOURCE_TYPE_HTML || '
                            </div>'
                    END ||

                    '<div style="font-size:13px;">
                        <a href="' || ARTICLE_URL_HTML || '" style="color:#0b57d0; text-decoration:none;">Read article →</a>
                    </div>

                </td>
            </tr>',
            ''
        ) WITHIN GROUP (
            ORDER BY
                PRIORITY_SORT,
                SIGNAL_SCORE DESC,
                RECEIVED_TS_PARSED DESC NULLS LAST
        ) AS BODY_ITEMS
    FROM TOP_ITEMS
)

SELECT
    'morten.laugesen@fujifilm.com' AS TO_EMAIL,

    'Daily Pharma News Digest - ' || TO_VARCHAR(CURRENT_DATE()) AS EMAIL_SUBJECT,

    '<html>
        <body style="margin:0; padding:0; background-color:#f6f6f6; font-family:Arial, sans-serif;">
            <table width="100%" cellpadding="0" cellspacing="0" style="background-color:#f6f6f6; padding:24px 0;">
                <tr>
                    <td align="center">
                        <table width="720" cellpadding="0" cellspacing="0" style="background-color:#ffffff; border:1px solid #e5e5e5;">
                            <tr>
                                <td style="padding:24px 28px 14px 28px;">
                                    <div style="font-size:22px; font-weight:bold; color:#1a1a1a; margin-bottom:6px;">
                                        Daily Pharma News Digest
                                    </div>
                                    <div style="font-size:13px; color:#666666; line-height:1.5;">
                                        Generated and sent from Snowflake<br>
                                        Lookback window: last 7 days · Top 10 unique stories
                                    </div>
                                </td>
                            </tr>

                            <tr>
                                <td style="padding:0 28px 18px 28px;">
                                    <table width="100%" cellpadding="0" cellspacing="0" style="background-color:#f3f5f7; border:1px solid #e2e6ea;">
                                        <tr>
                                            <td style="padding:12px 14px; font-size:13px; color:#333333;">
                                                <b>Raw relevant items:</b> ' || COALESCE(R.RAW_TOTAL_ITEMS, 0) || ' &nbsp; | &nbsp;
                                                <b>Unique stories:</b> ' || COALESCE(C.UNIQUE_STORIES, 0) || ' &nbsp; | &nbsp;
                                                <b>Very important:</b> ' || COALESCE(C.UNIQUE_VERY_IMPORTANT_ITEMS, 0) || ' &nbsp; | &nbsp;
                                                <b>Important:</b> ' || COALESCE(C.UNIQUE_IMPORTANT_ITEMS, 0) || '
                                            </td>
                                        </tr>
                                    </table>
                                </td>
                            </tr>

                            <tr>
                                <td style="padding:0 28px 28px 28px;">
                                    <table width="100%" cellpadding="0" cellspacing="0">' ||

                                        COALESCE(E.BODY_ITEMS, '<tr><td style="padding:16px 0;">No important pharma news items found in the selected period.</td></tr>') ||

                                    '</table>
                                </td>
                            </tr>
                        </table>
                    </td>
                </tr>
            </table>
        </body>
    </html>' AS EMAIL_BODY

FROM RAW_COUNTS R
CROSS JOIN DEDUPED_COUNTS C
CROSS JOIN EMAIL_TEXT E;
