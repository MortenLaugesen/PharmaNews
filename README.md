-- ============================================================
-- 04 - Final digest queue
-- Deterministic dedupe + ranking
--
-- V1.6 changes:
-- - Backfill from MONITOR tier (score >= 2) when fewer than 10
--   IMPORTANT/VERY_IMPORTANT stories survive dedup
-- - Cap output at 10 stories
--
-- V1.5.1 changes:
-- - Better duplicate handling
-- - Fixes duplicate deal stories such as GSK/Nuvalent
-- - Keeps cost-effective deterministic dedupe
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2 AS
WITH SOURCE_ROWS AS (
    SELECT
        RECEIVED_TS_PARSED,
        PUBLISH_DATE,
        PUBLISH_DATE_SOURCE,
        EMAIL_SOURCE_TYPE,

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
        DEAL_VALUE_KEY,

        LEFT(
            REGEXP_REPLACE(
                COALESCE(ARTICLE_LLM_INPUT, ARTICLE_TITLE_CLEAN, ''),
                '\\s+',
                ' '
            ),
            9000
        ) AS ARTICLE_SUMMARY_INPUT

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2
    WHERE (
        PRIORITY_TIER IN ('VERY_IMPORTANT', 'IMPORTANT')
        OR (PRIORITY_TIER = 'MONITOR' AND SIGNAL_SCORE >= 2)
    )
),

CANDIDATES AS (
    SELECT
        *,

        LOWER(
            TRIM(
                REGEXP_REPLACE(
                    REGEXP_REPLACE(
                        REGEXP_REPLACE(
                            ARTICLE_TITLE,
                            '((\\$|€|£)?\\s?[0-9]+(\\.[0-9]+)?\\s?(b|bn|m|billion|million))',
                            ' '
                        ),
                        '[^a-zA-Z0-9 ]',
                        ' '
                    ),
                    '\\s+',
                    ' '
                )
            )
        ) AS TITLE_ROOT_NO_VALUE,

        LOWER(TRIM(SPLIT_PART(COALESCE(MATCHED_COMPANIES, ''), ', ', 1))) AS PRIMARY_MATCHED_COMPANY

    FROM SOURCE_ROWS
    WHERE 1=1
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

      AND NOT LOWER(COALESCE(ARTICLE_TITLE, '')) RLIKE
          '(editor-in-chief|senior editor|senior writer|executive editor|associate editor|deputy editor|staff writer|staff writers|publisher|sales director|fact sheet|meet .* at asco)'

      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%read more%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%continue reading%'

      AND NOT REGEXP_LIKE(ARTICLE_TITLE, '^[^A-Za-z0-9].*')
      AND LENGTH(ARTICLE_TITLE) BETWEEN 20 AND 190
),

STORY_KEYS AS (
    SELECT
        *,

        -- Normalize deal value for dedup: remove spaces so "$10.6 b" and "$10.6b" match
        LOWER(REPLACE(COALESCE(DEAL_VALUE_KEY, ''), ' ', '')) AS DEAL_VALUE_NORMALIZED,

        CASE
            WHEN DEAL_VALUE_KEY IS NOT NULL
             AND COALESCE(NULLIF(PRIMARY_MATCHED_COMPANY, ''), '') <> ''
            THEN PRIMARY_MATCHED_COMPANY || '|' || LOWER(REPLACE(DEAL_VALUE_KEY, ' ', ''))

            WHEN DEAL_VALUE_KEY IS NOT NULL
            THEN LOWER(REPLACE(DEAL_VALUE_KEY, ' ', '')) || '|' || LEFT(TITLE_ROOT_NO_VALUE, 45)

            WHEN COALESCE(NULLIF(PRIMARY_MATCHED_COMPANY, ''), '') <> ''
            THEN PRIMARY_MATCHED_COMPANY || '|' || LEFT(TITLE_ROOT_NO_VALUE, 30)

            ELSE 'no_company|' || LEFT(TITLE_ROOT_NO_VALUE, 55)
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
                CASE
                    WHEN LENGTH(ARTICLE_TITLE) BETWEEN 55 AND 160 THEN 1
                    WHEN LENGTH(ARTICLE_TITLE) BETWEEN 35 AND 180 THEN 2
                    ELSE 3
                END,
                LENGTH(ARTICLE_TITLE) DESC
        ) AS STORY_RN
    FROM STORY_KEYS
),

RANKED AS (
    SELECT
        *,
        CASE
            WHEN PRIORITY_TIER = 'VERY_IMPORTANT' THEN 1
            WHEN PRIORITY_TIER = 'IMPORTANT' THEN 2
            ELSE 3
        END AS PRIORITY_SORT,

        ROW_NUMBER() OVER (
            ORDER BY
                CASE
                    WHEN PRIORITY_TIER = 'VERY_IMPORTANT' THEN 1
                    WHEN PRIORITY_TIER = 'IMPORTANT' THEN 2
                    ELSE 3
                END,
                SIGNAL_SCORE DESC,
                RECEIVED_TS_PARSED DESC NULLS LAST,
                LENGTH(ARTICLE_TITLE) DESC
        ) AS DIGEST_RANK

    FROM DEDUPED
    WHERE STORY_RN = 1
)

SELECT
    RECEIVED_TS_PARSED,
    PUBLISH_DATE,
    PUBLISH_DATE_SOURCE,
    EMAIL_SOURCE_TYPE,
    ARTICLE_TITLE,
    ARTICLE_URL,
    PRIORITY_TIER,
    PRIORITY_SORT,
    SIGNAL_SCORE,
    SIGNAL_REASONS,
    MATCHED_COMPANIES,
    MATCHED_COMPANY_CATEGORIES,
    DEAL_VALUE_KEY,
    STORY_KEY,
    DIGEST_RANK,
    ARTICLE_SUMMARY_INPUT,

    SHA2(
        LOWER(
            COALESCE(ARTICLE_URL, '') || '|' ||
            COALESCE(ARTICLE_TITLE, '')
        ),
        256
    ) AS ITEM_KEY,

    CASE
        WHEN SIGNAL_SCORE >= 7 THEN TRUE
        WHEN PRIORITY_TIER = 'VERY_IMPORTANT' THEN TRUE
        WHEN DEAL_VALUE_KEY IS NOT NULL AND REGEXP_LIKE(
            LOWER(REPLACE(COALESCE(DEAL_VALUE_KEY, ''), ' ', '')),
            '.*(\\$|€|£)?[0-9]+(\\.[0-9]+)?(b|bn|billion).*'
        ) THEN TRUE
        WHEN DIGEST_RANK <= 3 THEN TRUE
        ELSE FALSE
    END AS DEEP_DIVE_FLAG

FROM RANKED
WHERE DIGEST_RANK <= 10;
