-- ============================================================
-- 07 - Professional HTML email output view
-- Uses AI summaries + AI story keys
-- Result: Top 10 unique stories
--
-- Robust version:
-- - Uses v3 headline-locked summaries
-- - Joins summaries via one shared ITEM_KEY
-- - Uses latest summary per article
-- - Shows Email date / Publish date
-- - Includes debug-friendly Summary blocks
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_ALTERYX_DAILY_PHARMA_NEWS_EMAIL AS
WITH DIGEST AS (
    SELECT
        D.*,

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
    WHERE D.RECEIVED_TS_PARSED >= DATEADD('day', -7, CURRENT_TIMESTAMP())
      AND D.PRIORITY_TIER IN ('VERY_IMPORTANT', 'IMPORTANT')
),

LATEST_SUMMARIES AS (
    SELECT
        SUMMARY_KEY,
        SUMMARY_VERSION,
        ARTICLE_TITLE,
        ARTICLE_URL,
        AI_SUMMARY,
        CREATED_AT
    FROM PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES
    WHERE SUMMARY_VERSION = 'v3_headline_locked_1_3_paragraphs'
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY SUMMARY_KEY
        ORDER BY CREATED_AT DESC
    ) = 1
),

LATEST_STORY_KEYS AS (
    SELECT
        STORY_ITEM_KEY,
        AI_STORY_KEY,
        CREATED_AT
    FROM PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_STORY_KEYS
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY STORY_ITEM_KEY
        ORDER BY CREATED_AT DESC
    ) = 1
),

ITEMS_RAW AS (
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
        D.ITEM_KEY,
        D.PRIORITY_SORT,

        S.AI_SUMMARY,
        K.AI_STORY_KEY

    FROM DIGEST D

    LEFT JOIN LATEST_SUMMARIES S
        ON D.ITEM_KEY = S.SUMMARY_KEY

    LEFT JOIN LATEST_STORY_KEYS K
        ON D.ITEM_KEY = K.STORY_ITEM_KEY
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
                WHEN COALESCE(NULLIF(AI_SUMMARY, ''), '') <> '' THEN 1
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

        REGEXP_REPLACE(
            REPLACE(REPLACE(REPLACE(COALESCE(AI_SUMMARY, ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;'),
            '\\n+',
            '<br><br>'
        ) AS AI_SUMMARY_HTML,

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
        COUNT_IF(PRIORITY_TIER = 'IMPORTANT') AS RAW_IMPORTANT_ITEMS,
        COUNT_IF(COALESCE(NULLIF(AI_SUMMARY, ''), '') <> '') AS RAW_ITEMS_WITH_SUMMARY
    FROM ITEMS_RAW
),

DEDUPED_COUNTS AS (
    SELECT
        COUNT(*) AS UNIQUE_STORIES,
        COUNT_IF(PRIORITY_TIER = 'VERY_IMPORTANT') AS UNIQUE_VERY_IMPORTANT_ITEMS,
        COUNT_IF(PRIORITY_TIER = 'IMPORTANT') AS UNIQUE_IMPORTANT_ITEMS,
        COUNT_IF(COALESCE(NULLIF(AI_SUMMARY, ''), '') <> '') AS UNIQUE_STORIES_WITH_SUMMARY
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
                        ELSE
                            '<div style="font-size:14px; color:#999999; line-height:1.5; margin-bottom:10px;">
                                <b>Summary:</b><br>No AI summary available for this item.
                            </div>'
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
                                                <b>Important:</b> ' || COALESCE(C.UNIQUE_IMPORTANT_ITEMS, 0) || ' &nbsp; | &nbsp;
                                                <b>Summaries:</b> ' || COALESCE(C.UNIQUE_STORIES_WITH_SUMMARY, 0) || '
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
