-- FAST FIX SCRIPT
-- Run this when PHARMA_NEWS_CLASSIFIED already exists and you only want to update filtering/scoring/email output.

USE ROLE SANDBOX_DEVELOPER;
USE WAREHOUSE SANDBOX_WH;
USE DATABASE PHARMA_NEWS_SANDBOX;
USE SCHEMA NEWS;


-- ============================================================
-- 05 - Priority tier
-- Scoring-based priority logic
--
-- Key design:
-- Company matching only uses article title + subject.
-- It does NOT use full newsletter body.
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2 AS
WITH BASE AS (
    SELECT
        C.*,

        LOWER(
            COALESCE(C.ARTICLE_TITLE_CLEAN, '') || ' ' ||
            COALESCE(C.SUBJECT_RAW, '')
        ) AS TITLE_CONTEXT_LC,

        LOWER(
            COALESCE(C.ARTICLE_TITLE_CLEAN, '') || ' ' ||
            COALESCE(C.SUBJECT_RAW, '') || ' ' ||
            COALESCE(LEFT(C.ARTICLE_LLM_INPUT, 1000), '')
        ) AS FULL_TEXT_LC,

        LOWER(COALESCE(TO_JSON(C.CATEGORY_RESULT), '')) AS CATEGORY_TEXT_LC

    FROM PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_CLASSIFIED C
    WHERE C.IS_RELEVANT = TRUE
),

COMPANY_MATCHES AS (
    SELECT
        B.MESSAGE_ID,
        B.ARTICLE_TITLE_CLEAN,

        LISTAGG(DISTINCT TC.COMPANY_NAME, ', ')
            WITHIN GROUP (ORDER BY TC.COMPANY_NAME) AS MATCHED_COMPANIES,

        LISTAGG(DISTINCT TC.COMPANY_CATEGORY, ', ')
            WITHIN GROUP (ORDER BY TC.COMPANY_CATEGORY) AS MATCHED_COMPANY_CATEGORIES

    FROM BASE B
    LEFT JOIN PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_TRACKED_COMPANIES TC
        ON B.TITLE_CONTEXT_LC ILIKE '%' || LOWER(TC.COMPANY_NAME) || '%'

    GROUP BY
        B.MESSAGE_ID,
        B.ARTICLE_TITLE_CLEAN
),

ENRICHED AS (
    SELECT
        B.*,

        COALESCE(CM.MATCHED_COMPANIES, '') AS MATCHED_COMPANIES,
        COALESCE(CM.MATCHED_COMPANY_CATEGORIES, '') AS MATCHED_COMPANY_CATEGORIES,

        REGEXP_SUBSTR(
            B.TITLE_CONTEXT_LC,
            '(\\$|€|£)[0-9]+(\\.[0-9]+)?\\s?(b|m)|[0-9]+(\\.[0-9]+)?\\s?(billion|million)'
        ) AS DEAL_VALUE_KEY,

        CASE
            WHEN B.TITLE_CONTEXT_LC RLIKE '(\\$|€|£)[0-9]+(\\.[0-9]+)?\\s?b'
              OR B.TITLE_CONTEXT_LC RLIKE '[0-9]+(\\.[0-9]+)?\\s?billion'
            THEN TRUE ELSE FALSE
        END AS VALUE_ABOVE_1B,

        CASE
            WHEN B.TITLE_CONTEXT_LC RLIKE '(\\$|€|£)[5-9][0-9]{2}\\s?m'
              OR B.TITLE_CONTEXT_LC RLIKE '(\\$|€|£)[0-9]+(\\.[0-9]+)?\\s?b'
              OR B.TITLE_CONTEXT_LC RLIKE '[5-9][0-9]{2}\\s?million'
              OR B.TITLE_CONTEXT_LC RLIKE '[0-9]+(\\.[0-9]+)?\\s?billion'
            THEN TRUE ELSE FALSE
        END AS VALUE_ABOVE_500M,

        CASE
            WHEN B.TITLE_CONTEXT_LC RLIKE '(\\$|€|£)[2-9][0-9]{2}\\s?m'
              OR B.TITLE_CONTEXT_LC RLIKE '(\\$|€|£)[0-9]+(\\.[0-9]+)?\\s?b'
              OR B.TITLE_CONTEXT_LC RLIKE '[2-9][0-9]{2}\\s?million'
              OR B.TITLE_CONTEXT_LC RLIKE '[0-9]+(\\.[0-9]+)?\\s?billion'
            THEN TRUE ELSE FALSE
        END AS VALUE_ABOVE_200M

    FROM BASE B
    LEFT JOIN COMPANY_MATCHES CM
        ON B.MESSAGE_ID = CM.MESSAGE_ID
       AND B.ARTICLE_TITLE_CLEAN = CM.ARTICLE_TITLE_CLEAN
),

FLAGS AS (
    SELECT
        *,

        CASE
            WHEN ARTICLE_TITLE_LC LIKE 'a message from %' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE 'brought to you by %' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE 'sponsored by %' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%sponsored by%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%podcast%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%webinar%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%conference%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%whitepaper%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%register now%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%register today%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%unsubscribe%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%click here%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%read in browser%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%de-risk your program%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%real-world evidence%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%action gap%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%deliver confidence%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%move to market with confidence%' THEN TRUE
            WHEN ARTICLE_TITLE_LC LIKE '%explore our services%' THEN TRUE
            WHEN ARTICLE_TITLE_LC RLIKE
                '(editor-in-chief|senior editor|senior writer|executive editor|associate editor|deputy editor|staff writer|staff writers|publisher|sales director|fact sheet|meet .* at asco)'
            THEN TRUE
            ELSE FALSE
        END AS IS_PROMOTIONAL_SIGNAL,

        CASE
            WHEN TITLE_CONTEXT_LC RLIKE
                '(acquisition|buyout|merger|deal|licensing|collaboration|partnership|supply deal|supply agreement)'
            THEN TRUE ELSE FALSE
        END AS IS_DEAL_SIGNAL,

        CASE
            WHEN TITLE_CONTEXT_LC RLIKE
                '(expansion|construction|facility|site|manufacturing|capacity|capex|investment|new plant|new site|supply)'
            THEN TRUE ELSE FALSE
        END AS IS_SIZE_OR_CAPACITY_SIGNAL,

        CASE
            WHEN TITLE_CONTEXT_LC RLIKE
                '(new capability|new platform|new modality|fill-finish|fill finish|microbial|mammalian|cell therapy|gene therapy|adc|biosimilar|biosimilars|crispr)'
            THEN TRUE ELSE FALSE
        END AS IS_NEW_CAPABILITY_SIGNAL,

        CASE
            WHEN TITLE_CONTEXT_LC RLIKE
                '(closing|shuttering|divestment|divestiture|site closure|plant closure|business unit|layoffs|job cuts|restructuring)'
            THEN TRUE ELSE FALSE
        END AS IS_NEGATIVE_BUSINESS_SIGNAL,

        CASE
            WHEN TITLE_CONTEXT_LC RLIKE
                '(fda|approval|regulatory|regulation|tariff|tariffs|supreme court|drug shortages|shortages|review)'
            THEN TRUE ELSE FALSE
        END AS IS_POLICY_OR_REGULATORY_SIGNAL,

        CASE
            WHEN CATEGORY_TEXT_LC ILIKE '%competitor_investment_capacity%'
              OR CATEGORY_TEXT_LC ILIKE '%partnership_ma%'
              OR CATEGORY_TEXT_LC ILIKE '%capability_modality%'
              OR CATEGORY_TEXT_LC ILIKE '%clinical_regulatory_signal%'
              OR CATEGORY_TEXT_LC ILIKE '%policy_market_signal%'
              OR CATEGORY_TEXT_LC ILIKE '%commercial_customer_signal%'
              OR CATEGORY_TEXT_LC ILIKE '%financing_market_signal%'
            THEN TRUE ELSE FALSE
        END AS HAS_RELEVANT_AI_CATEGORY

    FROM ENRICHED
),

SCORED AS (
    SELECT
        *,

        IFF(VALUE_ABOVE_1B, 5, 0)
        + IFF(VALUE_ABOVE_500M, 3, 0)
        + IFF(MATCHED_COMPANY_CATEGORIES ILIKE '%CDMO%', 3, 0)
        + IFF(MATCHED_COMPANY_CATEGORIES ILIKE '%Top 25%', 2, 0)
        + IFF(IS_DEAL_SIGNAL, 2, 0)
        + IFF(IS_SIZE_OR_CAPACITY_SIGNAL, 2, 0)
        + IFF(IS_NEW_CAPABILITY_SIGNAL, 2, 0)
        + IFF(IS_NEGATIVE_BUSINESS_SIGNAL, 2, 0)
        + IFF(IS_POLICY_OR_REGULATORY_SIGNAL, 2, 0)
        + IFF(
            HAS_RELEVANT_AI_CATEGORY
            AND (
                MATCHED_COMPANIES <> ''
                OR IS_DEAL_SIGNAL
                OR IS_SIZE_OR_CAPACITY_SIGNAL
                OR IS_NEW_CAPABILITY_SIGNAL
                OR IS_NEGATIVE_BUSINESS_SIGNAL
                OR IS_POLICY_OR_REGULATORY_SIGNAL
            ),
            1,
            0
        )
        + IFF(SUBJECT_GATE_FINAL = 'PASS', 1, 0)
        AS SIGNAL_SCORE,

        TRIM(
            IFF(VALUE_ABOVE_1B, 'Value above 1B; ', '') ||
            IFF(VALUE_ABOVE_500M, 'Value above 500M; ', '') ||
            IFF(MATCHED_COMPANY_CATEGORIES ILIKE '%CDMO%', 'Tracked CDMO mentioned in title/subject; ', '') ||
            IFF(MATCHED_COMPANY_CATEGORIES ILIKE '%Top 25%', 'Top 25 pharma company mentioned in title/subject; ', '') ||
            IFF(IS_DEAL_SIGNAL, 'Deal/partnership/M&A signal; ', '') ||
            IFF(IS_SIZE_OR_CAPACITY_SIGNAL, 'Manufacturing/capacity/investment signal; ', '') ||
            IFF(IS_NEW_CAPABILITY_SIGNAL, 'Capability/modality signal; ', '') ||
            IFF(IS_NEGATIVE_BUSINESS_SIGNAL, 'Layoff/closure/divestment signal; ', '') ||
            IFF(IS_POLICY_OR_REGULATORY_SIGNAL, 'Policy/regulatory signal; ', '') ||
            IFF(HAS_RELEVANT_AI_CATEGORY, 'Relevant AI category; ', '') ||
            IFF(SUBJECT_GATE_FINAL = 'PASS', 'Strong keyword gate pass; ', '')
        ) AS SIGNAL_REASONS

    FROM FLAGS
)

SELECT
    *,
    CASE
        WHEN IS_PROMOTIONAL_SIGNAL THEN 'DROP'

        -- Tune these two thresholds if needed:
        -- Higher numbers = fewer important items
        -- Lower numbers = more important items
        WHEN SIGNAL_SCORE >= 6 THEN 'VERY_IMPORTANT'
        WHEN SIGNAL_SCORE >= 3 THEN 'IMPORTANT'

        ELSE 'MONITOR'
    END AS PRIORITY_TIER

FROM SCORED;


-- ============================================================
-- 06 - Final digest queue
-- Deduplicated final list for review / later Alteryx email
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2 AS
WITH CANDIDATES AS (
    SELECT
        RECEIVED_TS_PARSED,
        EMAIL_SOURCE_TYPE,
        ARTICLE_TITLE_CLEAN AS ARTICLE_TITLE,
        ARTICLE_URL,
        PRIORITY_TIER,
        SIGNAL_SCORE,
        SIGNAL_REASONS,
        MATCHED_COMPANIES,
        MATCHED_COMPANY_CATEGORIES,
        CATEGORY_RESULT,
        DEAL_VALUE_KEY,

        LOWER(
            TRIM(
                REGEXP_REPLACE(
                    REGEXP_REPLACE(
                        REGEXP_REPLACE(
                            ARTICLE_TITLE_CLEAN,
                            '\\b(report|asset|assets|drug|in us|with|to bag|taps|tap|capping|tenure|phase|trial|the|a|an|and|of|for|to|in|on|by|from)\\b',
                            ''
                        ),
                        '[^a-zA-Z0-9 ]',
                        ''
                    ),
                    '\\s+',
                    ' '
                )
            )
        ) AS NORMALIZED_TITLE

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2

    WHERE PRIORITY_TIER IN ('VERY_IMPORTANT', 'IMPORTANT')

      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE 'a message from %'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE 'brought to you by %'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE 'sponsored by %'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%sponsored by%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%podcast%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%webinar%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%conference%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%whitepaper%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%register now%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%register today%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%unsubscribe%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%click here%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%read in browser%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%de-risk your program%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%real-world evidence%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%action gap%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%deliver confidence%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%move to market with confidence%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%explore our services%'

      AND NOT LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) RLIKE
          '(editor-in-chief|senior editor|senior writer|executive editor|associate editor|deputy editor|staff writer|staff writers|publisher|sales director|fact sheet|meet .* at asco)'
),

STORY_KEYS AS (
    SELECT
        *,
        COALESCE(NULLIF(MATCHED_COMPANIES, ''), 'NO_COMPANY')
        || '|' ||
        COALESCE(NULLIF(DEAL_VALUE_KEY, ''), 'NO_VALUE')
        || '|' ||
        LEFT(NORMALIZED_TITLE, 80) AS STORY_KEY
    FROM CANDIDATES
),

DEDUPED AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY STORY_KEY
            ORDER BY
                CASE
                    WHEN PRIORITY_TIER = 'VERY_IMPORTANT' THEN 1
                    WHEN PRIORITY_TIER = 'IMPORTANT' THEN 2
                    ELSE 3
                END,
                SIGNAL_SCORE DESC,
                RECEIVED_TS_PARSED DESC NULLS LAST,
                LENGTH(ARTICLE_TITLE) DESC
        ) AS RN
    FROM STORY_KEYS
)

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
    CATEGORY_RESULT,
    DEAL_VALUE_KEY,
    STORY_KEY

FROM DEDUPED
WHERE RN = 1;


-- ============================================================
-- 07 - Alteryx email output view
-- Creates one email-ready row for Alteryx.
-- Snowflake does NOT send the email.
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

    -- For testing: last 30 days.
    -- Later for daily operation, change -30 to -1.
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
    LIMIT 20
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
    's235701@dtu.dk' AS TO_EMAIL,

    'Daily Pharma News Digest - ' || TO_VARCHAR(CURRENT_DATE()) AS EMAIL_SUBJECT,

    'Daily Pharma News Digest' || CHR(10) ||
    'Generated from Snowflake and sent by Alteryx' || CHR(10) ||
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


-- ============================================================
-- 08 - Checks
-- Run these after the script
-- ============================================================

SELECT COUNT(*) AS STAGING_ROWS
FROM PHARMA_NEWS_SANDBOX.NEWS.STG_PHARMA_NEWS_ARTICLES_V1;

SELECT
    SUBJECT_GATE_FINAL,
    COUNT(*) AS CNT
FROM PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_SUBJECT_GATE_FINAL
GROUP BY SUBJECT_GATE_FINAL
ORDER BY CNT DESC;

SELECT
    PRIORITY_TIER,
    COUNT(*) AS CNT
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2
GROUP BY PRIORITY_TIER
ORDER BY CNT DESC;

SELECT
    PRIORITY_TIER,
    COUNT(*) AS CNT
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
GROUP BY PRIORITY_TIER
ORDER BY CNT DESC;

SELECT
    RECEIVED_TS_PARSED,
    EMAIL_SOURCE_TYPE,
    ARTICLE_TITLE,
    PRIORITY_TIER,
    SIGNAL_SCORE,
    SIGNAL_REASONS,
    MATCHED_COMPANIES,
    MATCHED_COMPANY_CATEGORIES,
    ARTICLE_URL
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
ORDER BY
    CASE
        WHEN PRIORITY_TIER = 'VERY_IMPORTANT' THEN 1
        WHEN PRIORITY_TIER = 'IMPORTANT' THEN 2
        ELSE 3
    END,
    SIGNAL_SCORE DESC,
    RECEIVED_TS_PARSED DESC NULLS LAST
LIMIT 100;

SELECT
    TO_EMAIL,
    EMAIL_SUBJECT,
    EMAIL_BODY
FROM PHARMA_NEWS_SANDBOX.NEWS.V_ALTERYX_DAILY_PHARMA_NEWS_EMAIL;

-- Test that obvious promo rows are not in the final queue.
SELECT
    ARTICLE_TITLE,
    PRIORITY_TIER,
    SIGNAL_SCORE,
    SIGNAL_REASONS,
    MATCHED_COMPANIES
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
WHERE LOWER(ARTICLE_TITLE) LIKE 'a message from %'
   OR LOWER(ARTICLE_TITLE) LIKE 'brought to you by %'
   OR LOWER(ARTICLE_TITLE) LIKE 'sponsored by %'
   OR LOWER(ARTICLE_TITLE) LIKE '%podcast%'
   OR LOWER(ARTICLE_TITLE) LIKE '%webinar%';

   SELECT
    PRIORITY_TIER,
    COUNT(*) AS CNT
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
GROUP BY PRIORITY_TIER
ORDER BY CNT DESC;

SELECT
    RECEIVED_TS_PARSED,
    EMAIL_SOURCE_TYPE,
    ARTICLE_TITLE,
    PRIORITY_TIER,
    SIGNAL_SCORE,
    SIGNAL_REASONS,
    MATCHED_COMPANIES,
    MATCHED_COMPANY_CATEGORIES,
    ARTICLE_URL
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
ORDER BY
    CASE
        WHEN PRIORITY_TIER = 'VERY_IMPORTANT' THEN 1
        WHEN PRIORITY_TIER = 'IMPORTANT' THEN 2
        ELSE 3
    END,
    SIGNAL_SCORE DESC,
    RECEIVED_TS_PARSED DESC NULLS LAST
LIMIT 50;

SELECT
    PRIORITY_TIER,
    COUNT(*) AS CNT
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
GROUP BY PRIORITY_TIER
ORDER BY CNT DESC;

SELECT
    RECEIVED_TS_PARSED,
    EMAIL_SOURCE_TYPE,
    ARTICLE_TITLE,
    PRIORITY_TIER,
    SIGNAL_SCORE,
    SIGNAL_REASONS,
    MATCHED_COMPANIES,
    MATCHED_COMPANY_CATEGORIES,
    ARTICLE_URL
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
ORDER BY
    CASE
        WHEN PRIORITY_TIER = 'VERY_IMPORTANT' THEN 1
        WHEN PRIORITY_TIER = 'IMPORTANT' THEN 2
        ELSE 3
    END,
    SIGNAL_SCORE DESC,
    RECEIVED_TS_PARSED DESC NULLS LAST
LIMIT 50;
