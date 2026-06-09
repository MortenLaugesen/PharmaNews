-- ============================================================
-- 06A - AI summaries
-- Adds AI-generated 1-3 paragraph summaries for VERY_IMPORTANT items
-- Run after Step 06 exists.
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

        'v2_1_3_paragraphs' AS SUMMARY_VERSION,

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
       AND S.SUMMARY_VERSION = 'v2_1_3_paragraphs'

    WHERE D.PRIORITY_TIER = 'VERY_IMPORTANT'
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

                'Summarize this article in 1-3 paragraphs. ',
                'The summary should focus on specifics related to monetary value, capability expansion or reduction, and/or how the change is expected to shape the industry. ',

                'Use the provided article text as the primary source. ',
                'Do not simply restate the headline. ',
                'Do not invent facts, figures, manufacturing relevance, CDMO relevance, or strategic implications that are not supported by the article text. ',

                'When available, include specific company names, deal values, investment amounts, facility/site names, geography, modality, capacity, product/asset names, disease area, and timeline. ',

                'If no monetary value is stated, explicitly say that no specific monetary value was stated. ',
                'If no capability expansion or reduction is stated, explicitly say that no specific capability change was stated. ',

                'Write in a professional, concise style suitable for senior stakeholders. ',
                'No bullet points. No quotation marks. ',

                'Headline: ', COALESCE(ARTICLE_TITLE, ''), '. ',
                'Source: ', COALESCE(EMAIL_SOURCE_TYPE, ''), '. ',
                'Publish date: ', COALESCE(TO_VARCHAR(PUBLISH_DATE), 'Unknown'), '. ',
                'Matched companies: ', COALESCE(MATCHED_COMPANIES, ''), '. ',
                'Company categories: ', COALESCE(MATCHED_COMPANY_CATEGORIES, ''), '. ',
                'Signal reasons: ', COALESCE(SIGNAL_REASONS, ''), '. ',

                'Article text: ',
                COALESCE(LEFT(ARTICLE_SUMMARY_INPUT, 12000), '')
            ),
            {
                'temperature': 0,
                'max_tokens': 900
            }
        ) AS AI_SUMMARY,

        CURRENT_TIMESTAMP() AS CREATED_AT

    FROM TMP_AI_SUMMARY_CANDIDATES;


    RETURN 'AI summaries refreshed. New v2 summaries created: ' || V_NEW_SUMMARIES;

END;
$$;
