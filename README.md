-- ============================================================
-- 07 - Snowflake email output view
-- Creates one email-ready row for Snowflake email sending
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_ALTERYX_DAILY_PHARMA_NEWS_EMAIL AS
WITH ITEMS AS (
    SELECT
        RECEIVED_TS_PARSED,
        EMAIL_SOURCE_TYPE,
        ARTICLE_TITLE,
        ARTICLE_URL,
        PRIORITY_TIER,
        SIGNAL_SCORE,
        SIGNAL_REASONS,
        MATCHED_COMPANIES,
        MATCHED_COMPANY_CATEGORIES,

        CASE
            WHEN PRIORITY_TIER = 'VERY_IMPORTANT' THEN 1
            WHEN PRIORITY_TIER = 'IMPORTANT' THEN 2
            ELSE 3
        END AS PRIORITY_SORT

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2

    WHERE RECEIVED_TS_PARSED >= DATEADD('day', -30, CURRENT_TIMESTAMP())
      AND PRIORITY_TIER IN ('VERY_IMPORTANT', 'IMPORTANT')
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

COUNTS AS (
    SELECT
        COUNT(*) AS TOTAL_ITEMS,
        COUNT_IF(PRIORITY_TIER = 'VERY_IMPORTANT') AS VERY_IMPORTANT_ITEMS,
        COUNT_IF(PRIORITY_TIER = 'IMPORTANT') AS IMPORTANT_ITEMS
    FROM ITEMS
),

EMAIL_TEXT AS (
    SELECT
        LISTAGG(
            CASE
                WHEN PRIORITY_TIER = 'VERY_IMPORTANT' THEN '[VERY IMPORTANT] '
                ELSE '[IMPORTANT] '
            END ||
            ARTICLE_TITLE || CHR(10) ||
            'Source: ' || COALESCE(EMAIL_SOURCE_TYPE, '') || CHR(10) ||
            'Priority score: ' || COALESCE(TO_VARCHAR(SIGNAL_SCORE), '') || CHR(10) ||
            'Why selected: ' || COALESCE(SIGNAL_REASONS, '') || CHR(10) ||
            'Matched companies: ' || COALESCE(NULLIF(MATCHED_COMPANIES, ''), 'None') || CHR(10) ||
            'Company categories: ' || COALESCE(NULLIF(MATCHED_COMPANY_CATEGORIES, ''), 'None') || CHR(10) ||
            'URL: ' || COALESCE(ARTICLE_URL, '') || CHR(10) ||
            CHR(10) || '------------------------' || CHR(10),
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

    'Daily Pharma News Digest' || CHR(10) ||
    'Generated and sent from Snowflake' || CHR(10) ||
    'Lookback window: last 30 days' || CHR(10) ||
    CHR(10) ||
    'Total relevant items found: ' || COALESCE(C.TOTAL_ITEMS, 0) || CHR(10) ||
    'Very important items: ' || COALESCE(C.VERY_IMPORTANT_ITEMS, 0) || CHR(10) ||
    'Important items: ' || COALESCE(C.IMPORTANT_ITEMS, 0) || CHR(10) ||
    CHR(10) ||
    'NEWS ITEMS' || CHR(10) ||
    '==========' || CHR(10) ||
    COALESCE(E.BODY_ITEMS, 'No important pharma news items found in the selected period.') AS EMAIL_BODY

FROM COUNTS C
CROSS JOIN EMAIL_TEXT E;
