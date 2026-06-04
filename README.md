-- ============================================================
-- 06 - Final digest queue
-- Cleaner final list for review / later Alteryx email
--
-- Purpose:
-- 1. Keeps only IMPORTANT and VERY_IMPORTANT
-- 2. Removes promo/noise rows
-- 3. Removes article fragments and very long newsletter snippets
-- 4. Cleans soft-hyphens so duplicates like "Ab­b­Vie" and "AbbVie" match
-- 5. Deduplicates similar stories better
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2 AS
WITH SOURCE_ROWS AS (
    SELECT
        RECEIVED_TS_PARSED,
        EMAIL_SOURCE_TYPE,

        -- Clean title for display and deduplication
        TRIM(
            REGEXP_REPLACE(
                REPLACE(COALESCE(ARTICLE_TITLE_CLEAN, ''), CHR(173), ''),
                '\\s+',
                ' '
            )
        ) AS ARTICLE_TITLE,

        ARTICLE_URL,
        PRIORITY_TIER,
        SIGNAL_SCORE,
        SIGNAL_REASONS,
        MATCHED_COMPANIES,
        MATCHED_COMPANY_CATEGORIES,
        CATEGORY_RESULT,
        DEAL_VALUE_KEY

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2
    WHERE PRIORITY_TIER IN ('VERY_IMPORTANT', 'IMPORTANT')
),

CANDIDATES AS (
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

        LOWER(
            TRIM(
                REGEXP_REPLACE(
                    REGEXP_REPLACE(
                        REGEXP_REPLACE(
                            ARTICLE_TITLE,
                            '\\b(report|reports|asset|assets|drug|drugs|in|us|with|to|bag|bags|taps|tap|capping|tenure|phase|trial|the|a|an|and|of|for|on|by|from|worth|up|new|data|program|programs|being|aimed|at|after|as|into|via)\\b',
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

    FROM SOURCE_ROWS

    WHERE 1=1

      -- Remove obvious promo/noise
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE 'a message from %'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE 'brought to you by %'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE 'sponsored by %'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%sponsored by%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%podcast%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%webinar%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%conference%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%whitepaper%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%register now%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%register today%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%unsubscribe%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%click here%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%read in browser%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%de-risk your program%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%real-world evidence%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%action gap%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%deliver confidence%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%move to market with confidence%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%explore our services%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%fierce ai innovation award%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%biopharma sentiment index%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%partnerships with sites%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE 'fierce pharma asia%'

      -- Remove editorial/admin rows
      AND NOT LOWER(COALESCE(ARTICLE_TITLE, '')) RLIKE
          '(editor-in-chief|senior editor|senior writer|executive editor|associate editor|deputy editor|staff writer|staff writers|publisher|sales director|fact sheet|meet .* at asco)'

      -- Remove fragments that start with lower-case
      AND NOT REGEXP_LIKE(ARTICLE_TITLE, '^[a-z]')

      -- Remove very short titles and long newsletter snippets
      AND LENGTH(ARTICLE_TITLE) BETWEEN 20 AND 180
),

STORY_KEYS AS (
    SELECT
        *,

        CASE
            -- Strongest dedupe: same company + same deal value
            WHEN DEAL_VALUE_KEY IS NOT NULL
             AND COALESCE(NULLIF(MATCHED_COMPANIES, ''), '') <> ''
            THEN
                COALESCE(NULLIF(MATCHED_COMPANIES, ''), 'NO_COMPANY')
                || '|'
                || LOWER(DEAL_VALUE_KEY)

            -- If no company but value exists, use value + shortened normalized title
            WHEN DEAL_VALUE_KEY IS NOT NULL
            THEN
                LOWER(DEAL_VALUE_KEY)
                || '|'
                || LEFT(NORMALIZED_TITLE, 45)

            -- If company exists but no value, use company + title
            WHEN COALESCE(NULLIF(MATCHED_COMPANIES, ''), '') <> ''
            THEN
                COALESCE(NULLIF(MATCHED_COMPANIES, ''), 'NO_COMPANY')
                || '|'
                || LEFT(NORMALIZED_TITLE, 50)

            -- Fallback: title only
            ELSE
                'NO_COMPANY|'
                || LEFT(NORMALIZED_TITLE, 60)
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

                -- Prefer cleaner/more complete titles, but avoid very long fragments
                CASE
                    WHEN LENGTH(ARTICLE_TITLE) BETWEEN 40 AND 140 THEN 1
                    ELSE 2
                END,

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
