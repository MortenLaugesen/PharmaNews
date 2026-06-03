-- ============================================================
-- 05 - Priority tier
-- Clean scoring logic
--
-- Fix:
-- Uses LIKE ANY instead of loose RLIKE patterns.
-- This makes normal article titles score correctly.
-- A row cannot become IMPORTANT only because a company is mentioned.
-- It must also have a real story signal.
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2 AS
WITH BASE AS (
    SELECT
        C.*,

        LOWER(COALESCE(C.ARTICLE_TITLE_CLEAN, '')) AS TITLE_CONTEXT_LC,

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
        ON B.TITLE_CONTEXT_LC LIKE '%' || LOWER(TC.COMPANY_NAME) || '%'

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
            '(\\$|€|£)?[0-9]+(\\.[0-9]+)?\\s?(b|bn|billion|m|million)'
        ) AS DEAL_VALUE_KEY,

        CASE
            WHEN REGEXP_LIKE(
                B.TITLE_CONTEXT_LC,
                '.*(\\$|€|£)?[0-9]+(\\.[0-9]+)?\\s?(b|bn|billion).*'
            )
            THEN TRUE ELSE FALSE
        END AS VALUE_ABOVE_1B,

        CASE
            WHEN REGEXP_LIKE(
                B.TITLE_CONTEXT_LC,
                '.*(\\$|€|£)?[0-9]+(\\.[0-9]+)?\\s?(b|bn|billion).*'
            )
            OR REGEXP_LIKE(
                B.TITLE_CONTEXT_LC,
                '.*(\\$|€|£)?[5-9][0-9]{2,}\\s?(m|million).*'
            )
            THEN TRUE ELSE FALSE
        END AS VALUE_ABOVE_500M

    FROM BASE B
    LEFT JOIN COMPANY_MATCHES CM
        ON B.MESSAGE_ID = CM.MESSAGE_ID
       AND B.ARTICLE_TITLE_CLEAN = CM.ARTICLE_TITLE_CLEAN
),

FLAGS AS (
    SELECT
        *,

        CASE
            WHEN TITLE_CONTEXT_LC LIKE ANY (
                'a message from %',
                'brought to you by %',
                'sponsored by %',
                '%sponsored by%',
                '%podcast%',
                '%webinar%',
                '%conference%',
                '%whitepaper%',
                '%register now%',
                '%register today%',
                '%unsubscribe%',
                '%click here%',
                '%read in browser%',
                '%de-risk your program%',
                '%real-world evidence%',
                '%action gap%',
                '%deliver confidence%',
                '%move to market with confidence%',
                '%explore our services%',
                '%fierce ai innovation award%',
                '%partnerships with sites%',
                '%biopharma sentiment index%',
                'the company announced%'
            )
            OR REGEXP_LIKE(
                TITLE_CONTEXT_LC,
                '.*(editor-in-chief|senior editor|senior writer|executive editor|associate editor|deputy editor|staff writer|staff writers|publisher|sales director|fact sheet|meet .* at asco).*'
            )
            THEN TRUE
            ELSE FALSE
        END AS IS_PROMOTIONAL_SIGNAL,

        CASE
            WHEN TITLE_CONTEXT_LC LIKE ANY (
                '%acquisition%',
                '%buyout%',
                '%merger%',
                '%deal%',
                '%licensing%',
                '%collaboration%',
                '%partnership%',
                '%supply deal%',
                '%supply agreement%',
                '%pact%'
            )
            THEN TRUE ELSE FALSE
        END AS IS_DEAL_SIGNAL,

        CASE
            WHEN TITLE_CONTEXT_LC LIKE ANY (
                '%expansion%',
                '%construction%',
                '%facility%',
                '%site%',
                '%manufacturing%',
                '%capacity%',
                '%capex%',
                '%investment%',
                '%invests%',
                '%new plant%',
                '%new site%'
            )
            THEN TRUE ELSE FALSE
        END AS IS_SIZE_OR_CAPACITY_SIGNAL,

        CASE
            WHEN TITLE_CONTEXT_LC LIKE ANY (
                '%new capability%',
                '%new platform%',
                '%new modality%',
                '%fill-finish%',
                '%fill finish%',
                '%microbial%',
                '%mammalian%',
                '%cell therapy%',
                '%gene therapy%',
                '%adc%',
                '%biosimilar%',
                '%biosimilars%',
                '%crispr%',
                '%car-t%'
            )
            THEN TRUE ELSE FALSE
        END AS IS_NEW_CAPABILITY_SIGNAL,

        CASE
            WHEN TITLE_CONTEXT_LC LIKE ANY (
                '%closing%',
                '%shuttering%',
                '%divestment%',
                '%divestiture%',
                '%site closure%',
                '%plant closure%',
                '%business unit%',
                '%layoffs%',
                '%job cuts%',
                '%restructuring%'
            )
            THEN TRUE ELSE FALSE
        END AS IS_NEGATIVE_BUSINESS_SIGNAL,

        CASE
            WHEN TITLE_CONTEXT_LC LIKE ANY (
                '%fda%',
                '%approval%',
                '%approved%',
                '%regulatory%',
                '%regulation%',
                '%tariff%',
                '%tariffs%',
                '%supreme court%',
                '%drug shortages%',
                '%shortages%',
                '%review%',
                '%phase 3%',
                '%phase iii%',
                '%clinical hold%'
            )
            THEN TRUE ELSE FALSE
        END AS IS_POLICY_OR_REGULATORY_SIGNAL,

        CASE
            WHEN TITLE_CONTEXT_LC LIKE ANY (
                '%coverage%',
                '%launch%',
                '%sales outlook%',
                '%commercial%',
                '%market access%',
                '%reimbursement%',
                '%customer%',
                '%contract%',
                '%award%',
                '%supply%'
            )
            THEN TRUE ELSE FALSE
        END AS IS_COMMERCIAL_SIGNAL,

        CASE
            WHEN CATEGORY_TEXT_LC LIKE '%competitor_investment_capacity%'
              OR CATEGORY_TEXT_LC LIKE '%partnership_ma%'
              OR CATEGORY_TEXT_LC LIKE '%capability_modality%'
              OR CATEGORY_TEXT_LC LIKE '%clinical_regulatory_signal%'
              OR CATEGORY_TEXT_LC LIKE '%policy_market_signal%'
              OR CATEGORY_TEXT_LC LIKE '%commercial_customer_signal%'
              OR CATEGORY_TEXT_LC LIKE '%financing_market_signal%'
            THEN TRUE ELSE FALSE
        END AS HAS_RELEVANT_AI_CATEGORY

    FROM ENRICHED
),

STORY AS (
    SELECT
        *,

        CASE
            WHEN VALUE_ABOVE_1B THEN TRUE
            WHEN VALUE_ABOVE_500M THEN TRUE
            WHEN IS_DEAL_SIGNAL THEN TRUE
            WHEN IS_SIZE_OR_CAPACITY_SIGNAL THEN TRUE
            WHEN IS_NEW_CAPABILITY_SIGNAL THEN TRUE
            WHEN IS_NEGATIVE_BUSINESS_SIGNAL THEN TRUE
            WHEN IS_POLICY_OR_REGULATORY_SIGNAL THEN TRUE
            WHEN IS_COMMERCIAL_SIGNAL THEN TRUE
            ELSE FALSE
        END AS HAS_STORY_SIGNAL

    FROM FLAGS
),

SCORED AS (
    SELECT
        *,

        IFF(VALUE_ABOVE_1B, 5, 0)
        + IFF(VALUE_ABOVE_500M, 3, 0)
        + IFF(IS_DEAL_SIGNAL, 2, 0)
        + IFF(IS_SIZE_OR_CAPACITY_SIGNAL, 2, 0)
        + IFF(IS_NEW_CAPABILITY_SIGNAL, 2, 0)
        + IFF(IS_NEGATIVE_BUSINESS_SIGNAL, 2, 0)
        + IFF(IS_POLICY_OR_REGULATORY_SIGNAL, 2, 0)
        + IFF(IS_COMMERCIAL_SIGNAL, 2, 0)

        + IFF(MATCHED_COMPANY_CATEGORIES LIKE '%CDMO%' AND HAS_STORY_SIGNAL, 3, 0)
        + IFF(MATCHED_COMPANY_CATEGORIES LIKE '%Top 25%' AND HAS_STORY_SIGNAL, 1, 0)

        + IFF(HAS_RELEVANT_AI_CATEGORY AND HAS_STORY_SIGNAL, 1, 0)

        + IFF(SUBJECT_GATE_FINAL = 'PASS' AND HAS_STORY_SIGNAL, 1, 0)

        AS SIGNAL_SCORE,

        TRIM(
            IFF(VALUE_ABOVE_1B, 'Value above 1B; ', '') ||
            IFF(VALUE_ABOVE_500M, 'Value above 500M; ', '') ||
            IFF(IS_DEAL_SIGNAL, 'Deal/partnership/M&A signal; ', '') ||
            IFF(IS_SIZE_OR_CAPACITY_SIGNAL, 'Manufacturing/capacity/investment signal; ', '') ||
            IFF(IS_NEW_CAPABILITY_SIGNAL, 'Capability/modality signal; ', '') ||
            IFF(IS_NEGATIVE_BUSINESS_SIGNAL, 'Layoff/closure/divestment signal; ', '') ||
            IFF(IS_POLICY_OR_REGULATORY_SIGNAL, 'Policy/regulatory signal; ', '') ||
            IFF(IS_COMMERCIAL_SIGNAL, 'Commercial/customer signal; ', '') ||
            IFF(MATCHED_COMPANY_CATEGORIES LIKE '%CDMO%' AND HAS_STORY_SIGNAL, 'Tracked CDMO mentioned in title; ', '') ||
            IFF(MATCHED_COMPANY_CATEGORIES LIKE '%Top 25%' AND HAS_STORY_SIGNAL, 'Top 25 pharma company mentioned in title; ', '') ||
            IFF(HAS_RELEVANT_AI_CATEGORY AND HAS_STORY_SIGNAL, 'Relevant AI category; ', '') ||
            IFF(SUBJECT_GATE_FINAL = 'PASS' AND HAS_STORY_SIGNAL, 'Strong keyword gate pass; ', '')
        ) AS SIGNAL_REASONS

    FROM STORY
)

SELECT
    *,
    CASE
        WHEN IS_PROMOTIONAL_SIGNAL THEN 'DROP'
        WHEN HAS_STORY_SIGNAL = FALSE THEN 'MONITOR'

        WHEN SIGNAL_SCORE >= 7 THEN 'VERY_IMPORTANT'
        WHEN SIGNAL_SCORE >= 4 THEN 'IMPORTANT'

        ELSE 'MONITOR'
    END AS PRIORITY_TIER

FROM SCORED;


SELECT
    PRIORITY_TIER,
    COUNT(*) AS CNT
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2
GROUP BY PRIORITY_TIER
ORDER BY CNT DESC;
