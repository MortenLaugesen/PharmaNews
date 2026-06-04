-- ============================================================
-- 06 - Final digest queue
-- Cleaner final list for review / later Alteryx email
--
-- Purpose:
-- 1. Keeps only IMPORTANT and VERY_IMPORTANT
-- 2. Removes promo/noise rows
-- 3. Removes article fragments and very long newsletter snippets
-- 4. Deduplicates similar stories better
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
                            '\\b(report|reports|asset|assets|drug|drugs|in|us|with|to|bag|bags|taps|tap|capping|tenure|phase|trial|the|a|an|and|of|for|on|by|from|worth|up|new|data|program|programs)\\b',
                            ' '
                        ),
                        '[^a-zA-Z0-9 ]',
                        ' '
                    ),
                    '\\s+',
                    ' '
                )
            )
        ) AS NORMALIZED_TITLE

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2

    WHERE PRIORITY_TIER IN ('VERY_IMPORTANT', 'IMPORTANT')

      -- Remove obvious promo/noise
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
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%fierce ai innovation award%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%biopharma sentiment index%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%partnerships with sites%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE 'fierce pharma asia%'

      -- Remove editorial/admin rows
      AND NOT LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) RLIKE
          '(editor-in-chief|senior editor|senior writer|executive editor|associate editor|deputy editor|staff writer|staff writers|publisher|sales director|fact sheet|meet .* at asco)'

      -- Remove text fragments that start with lower-case
      AND NOT REGEXP_LIKE(ARTICLE_TITLE_CLEAN, '^[a-z]')

      -- Remove newsletter paragraph snippets masquerading as article titles
      AND LENGTH(ARTICLE_TITLE_CLEAN) BETWEEN 20 AND 180
),

STORY_KEYS AS (
    SELECT
        *,

        CASE
            -- If there is a company + value, this is usually the strongest dedupe key
            WHEN DEAL_VALUE_KEY IS NOT NULL
             AND COALESCE(NULLIF(MATCHED_COMPANIES, ''), '') <> ''
            THEN
                COALESCE(NULLIF(MATCHED_COMPANIES, ''), 'NO_COMPANY')
                || '|'
                || DEAL_VALUE_KEY

            -- If only value exists, combine value with shortened title
            WHEN DEAL_VALUE_KEY IS NOT NULL
            THEN
                DEAL_VALUE_KEY
                || '|'
                || LEFT(NORMALIZED_TITLE, 40)

            -- Otherwise use company + shortened normalized title
            ELSE
                COALESCE(NULLIF(MATCHED_COMPANIES, ''), 'NO_COMPANY')
                || '|'
                || LEFT(NORMALIZED_TITLE, 45)
        END AS STORY_KEY

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
