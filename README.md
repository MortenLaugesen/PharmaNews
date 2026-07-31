-- ============================================================
-- 08 - US Investment Tracker
-- Precision-first, but with review candidates
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_US_INVESTMENT_TRACKER AS
WITH BASE AS (
    SELECT
        D.RECEIVED_TS_PARSED,
        D.PUBLISH_DATE,
        D.PUBLISH_DATE_SOURCE,
        D.EMAIL_SOURCE_TYPE,
        D.ARTICLE_TITLE_CLEAN,
        D.ARTICLE_URL,
        D.PRIORITY_TIER,
        D.SIGNAL_SCORE,
        D.SIGNAL_REASONS,
        D.MATCHED_COMPANIES,
        D.MATCHED_COMPANY_CATEGORIES,
        D.DEAL_VALUE_KEY,

        LOWER(
            TRIM(
                REGEXP_REPLACE(
                    COALESCE(D.ARTICLE_TITLE_CLEAN, ''),
                    '\\s+',
                    ' '
                )
            )
        ) AS TITLE_LC,

        LOWER(
            TRIM(
                REGEXP_REPLACE(
                    COALESCE(D.ARTICLE_TITLE_CLEAN, '') || ' ' ||
                    COALESCE(LEFT(D.ARTICLE_LLM_INPUT, 1500), '') || ' ' ||
                    COALESCE(LEFT(D.BODY_BEST, 1500), ''),
                    '\\s+',
                    ' '
                )
            )
        ) AS LIMITED_CONTEXT_LC

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2 D
    WHERE D.ARTICLE_TITLE_CLEAN IS NOT NULL
),

FLAGS AS (
    SELECT
        B.RECEIVED_TS_PARSED,
        B.PUBLISH_DATE,
        B.PUBLISH_DATE_SOURCE,
        B.EMAIL_SOURCE_TYPE,
        B.ARTICLE_TITLE_CLEAN,
        B.ARTICLE_URL,
        B.PRIORITY_TIER,
        B.SIGNAL_SCORE,
        B.SIGNAL_REASONS,
        B.MATCHED_COMPANIES,
        B.MATCHED_COMPANY_CATEGORIES,
        B.DEAL_VALUE_KEY,
        B.TITLE_LC,
        B.LIMITED_CONTEXT_LC,

        REGEXP_LIKE(
            B.TITLE_LC,
            '.*\\b(united states|u\\.s\\.|u\\.s|usa|america|american|north carolina|massachusetts|california|texas|new jersey|pennsylvania|maryland|indiana|washington|ohio|georgia|boston|cambridge|rtp|research triangle park|durham|raleigh|new york|san diego|philadelphia)\\b.*'
        ) AS HAS_US_LOCATION_IN_TITLE,

        REGEXP_LIKE(
            B.LIMITED_CONTEXT_LC,
            '.*\\b(united states|u\\.s\\.|u\\.s|usa|america|american|north carolina|massachusetts|california|texas|new jersey|pennsylvania|maryland|indiana|washington|ohio|georgia|boston|cambridge|rtp|research triangle park|durham|raleigh|new york|san diego|philadelphia)\\b.*'
        ) AS HAS_US_LOCATION_IN_CONTEXT,

        REGEXP_LIKE(
            B.TITLE_LC,
            '.*\\b(manufacturing|facility|plant|site|campus|factory|construction|build|building|new site|new plant|biomanufacturing|production|fill-finish|fill finish)\\b.*'
        ) AS HAS_MANUFACTURING_SIGNAL,

        REGEXP_LIKE(
            B.TITLE_LC,
            '.*\\b(capacity|expansion|expand|expanded|expands|adding capacity|adds capacity|scale up|scale-up)\\b.*'
        ) AS HAS_CAPACITY_SIGNAL,

        REGEXP_LIKE(
            B.TITLE_LC,
            '.*\\b(investment|invests|invested|invest|capex|capital expenditure|funding|grant|grants|tax credit|tax credits|incentive|incentives|spending|spend|committed|commitment)\\b.*'
        ) AS HAS_INVESTMENT_SIGNAL,

        REGEXP_LIKE(
            B.TITLE_LC,
            '.*\\b(jobs|job creation|hiring|hire|headcount|layoffs|layoff|workforce|employees|staff|personnel)\\b.*'
        ) AS HAS_WORKFORCE_SIGNAL,

        REGEXP_LIKE(
            B.TITLE_LC,
            '.*\\b(tariff|tariffs|reshoring|onshoring|nearshoring|domestic supply|supply chain|localization|localisation|u\\.s\\. supply|us supply)\\b.*'
        ) AS HAS_SUPPLY_CHAIN_SIGNAL,

        REGEXP_LIKE(
            B.TITLE_LC,
            '.*\\b(state incentive|state incentives|federal funding|tax credit|tax credits|grant|grants|subsidy|subsidies|government funding|public funding)\\b.*'
        ) AS HAS_POLICY_SIGNAL,

        REGEXP_LIKE(
            B.TITLE_LC,
            '.*\\b(acquisition|buyout|merger|licensing|collaboration|partnership|pact|clinical|phase 1|phase 2|phase 3|phase i|phase ii|phase iii|r&d|research and development|antitrust|lawsuit|litigation|verdict)\\b.*'
        ) AS HAS_GENERAL_DEAL_OR_CLINICAL_SIGNAL,

        CASE
            WHEN B.MATCHED_COMPANY_CATEGORIES LIKE '%CDMO%' OR B.TITLE_LC LIKE '%cdmo%' THEN TRUE
            ELSE FALSE
        END AS HAS_CDMO_SIGNAL,

        CASE
            WHEN B.MATCHED_COMPANY_CATEGORIES LIKE '%Top 25%' THEN TRUE
            ELSE FALSE
        END AS HAS_TOP25_SIGNAL

    FROM BASE B
),

CANDIDATES AS (
    SELECT
        F.RECEIVED_TS_PARSED,
        F.PUBLISH_DATE,
        F.PUBLISH_DATE_SOURCE,
        F.EMAIL_SOURCE_TYPE,
        F.ARTICLE_TITLE_CLEAN,
        F.ARTICLE_URL,
        F.PRIORITY_TIER,
        F.SIGNAL_SCORE,
        F.SIGNAL_REASONS,
        F.MATCHED_COMPANIES,
        F.MATCHED_COMPANY_CATEGORIES,
        F.DEAL_VALUE_KEY,
        F.TITLE_LC,
        F.LIMITED_CONTEXT_LC,

        F.HAS_US_LOCATION_IN_TITLE,
        F.HAS_US_LOCATION_IN_CONTEXT,
        F.HAS_MANUFACTURING_SIGNAL,
        F.HAS_CAPACITY_SIGNAL,
        F.HAS_INVESTMENT_SIGNAL,
        F.HAS_WORKFORCE_SIGNAL,
        F.HAS_SUPPLY_CHAIN_SIGNAL,
        F.HAS_POLICY_SIGNAL,
        F.HAS_GENERAL_DEAL_OR_CLINICAL_SIGNAL,
        F.HAS_CDMO_SIGNAL,
        F.HAS_TOP25_SIGNAL,

        CASE
            WHEN F.HAS_US_LOCATION_IN_TITLE = TRUE
             AND (
                    F.HAS_MANUFACTURING_SIGNAL = TRUE
                 OR F.HAS_CAPACITY_SIGNAL = TRUE
                 OR F.HAS_INVESTMENT_SIGNAL = TRUE
                 OR F.HAS_WORKFORCE_SIGNAL = TRUE
                 OR F.HAS_SUPPLY_CHAIN_SIGNAL = TRUE
                 OR F.HAS_POLICY_SIGNAL = TRUE
             )
            THEN 'STRICT_TITLE_MATCH'

            WHEN (F.HAS_POLICY_SIGNAL = TRUE OR F.HAS_SUPPLY_CHAIN_SIGNAL = TRUE)
            THEN 'STRICT_POLICY_OR_SUPPLY_CHAIN_MATCH'

            WHEN F.HAS_US_LOCATION_IN_CONTEXT = TRUE
             AND (
                    F.HAS_MANUFACTURING_SIGNAL = TRUE
                 OR F.HAS_CAPACITY_SIGNAL = TRUE
                 OR F.HAS_WORKFORCE_SIGNAL = TRUE
             )
            THEN 'REVIEW_CONTEXT_LOCATION_MATCH'

            ELSE 'NO_MATCH'
        END AS US_INVESTMENT_MATCH_LEVEL

    FROM FLAGS F
),

FILTERED AS (
    SELECT
        C.RECEIVED_TS_PARSED,
        C.PUBLISH_DATE,
        C.PUBLISH_DATE_SOURCE,
        C.EMAIL_SOURCE_TYPE,
        C.ARTICLE_TITLE_CLEAN,
        C.ARTICLE_URL,
        C.PRIORITY_TIER,
        C.SIGNAL_SCORE,
        C.SIGNAL_REASONS,
        C.MATCHED_COMPANIES,
        C.MATCHED_COMPANY_CATEGORIES,
        C.DEAL_VALUE_KEY,
        C.TITLE_LC,
        C.LIMITED_CONTEXT_LC,

        C.HAS_US_LOCATION_IN_TITLE,
        C.HAS_US_LOCATION_IN_CONTEXT,
        C.HAS_MANUFACTURING_SIGNAL,
        C.HAS_CAPACITY_SIGNAL,
        C.HAS_INVESTMENT_SIGNAL,
        C.HAS_WORKFORCE_SIGNAL,
        C.HAS_SUPPLY_CHAIN_SIGNAL,
        C.HAS_POLICY_SIGNAL,
        C.HAS_GENERAL_DEAL_OR_CLINICAL_SIGNAL,
        C.HAS_CDMO_SIGNAL,
        C.HAS_TOP25_SIGNAL,
        C.US_INVESTMENT_MATCH_LEVEL

    FROM CANDIDATES C
    WHERE C.US_INVESTMENT_MATCH_LEVEL <> 'NO_MATCH'

      -- Exclude broad deal/clinical stories unless they have a concrete operational, workforce,
      -- policy, supply-chain or capacity signal in the title.
      AND NOT (
            C.HAS_GENERAL_DEAL_OR_CLINICAL_SIGNAL = TRUE
        AND C.HAS_MANUFACTURING_SIGNAL = FALSE
        AND C.HAS_CAPACITY_SIGNAL = FALSE
        AND C.HAS_WORKFORCE_SIGNAL = FALSE
        AND C.HAS_SUPPLY_CHAIN_SIGNAL = FALSE
        AND C.HAS_POLICY_SIGNAL = FALSE
      )
),

SCORING AS (
    SELECT
        F.RECEIVED_TS_PARSED,
        F.PUBLISH_DATE,
        F.PUBLISH_DATE_SOURCE,
        F.EMAIL_SOURCE_TYPE,
        F.ARTICLE_TITLE_CLEAN,
        F.ARTICLE_URL,
        F.PRIORITY_TIER,
        F.SIGNAL_SCORE,
        F.SIGNAL_REASONS,
        F.MATCHED_COMPANIES,
        F.MATCHED_COMPANY_CATEGORIES,
        F.DEAL_VALUE_KEY,
        F.TITLE_LC,
        F.LIMITED_CONTEXT_LC,
        F.US_INVESTMENT_MATCH_LEVEL,

        F.HAS_US_LOCATION_IN_TITLE,
        F.HAS_US_LOCATION_IN_CONTEXT,
        F.HAS_MANUFACTURING_SIGNAL,
        F.HAS_CAPACITY_SIGNAL,
        F.HAS_INVESTMENT_SIGNAL,
        F.HAS_WORKFORCE_SIGNAL,
        F.HAS_SUPPLY_CHAIN_SIGNAL,
        F.HAS_POLICY_SIGNAL,
        F.HAS_CDMO_SIGNAL,
        F.HAS_TOP25_SIGNAL,

        IFF(F.US_INVESTMENT_MATCH_LEVEL = 'STRICT_TITLE_MATCH', 5, 0)
        + IFF(F.US_INVESTMENT_MATCH_LEVEL = 'STRICT_POLICY_OR_SUPPLY_CHAIN_MATCH', 5, 0)
        + IFF(F.US_INVESTMENT_MATCH_LEVEL = 'REVIEW_CONTEXT_LOCATION_MATCH', 2, 0)
        + IFF(F.HAS_US_LOCATION_IN_TITLE, 4, 0)
        + IFF(F.HAS_US_LOCATION_IN_CONTEXT, 1, 0)
        + IFF(F.HAS_MANUFACTURING_SIGNAL, 5, 0)
        + IFF(F.HAS_CAPACITY_SIGNAL, 5, 0)
        + IFF(F.HAS_INVESTMENT_SIGNAL, 3, 0)
        + IFF(F.HAS_WORKFORCE_SIGNAL, 3, 0)
        + IFF(F.HAS_SUPPLY_CHAIN_SIGNAL, 3, 0)
        + IFF(F.HAS_POLICY_SIGNAL, 3, 0)
        + IFF(F.DEAL_VALUE_KEY IS NOT NULL, 2, 0)
        + IFF(F.HAS_CDMO_SIGNAL, 2, 0)
        + IFF(F.HAS_TOP25_SIGNAL, 1, 0)
        + IFF(F.PRIORITY_TIER = 'VERY_IMPORTANT', 1, 0)
        AS US_INVESTMENT_SCORE,

        CASE
            WHEN F.HAS_POLICY_SIGNAL THEN 'Policy / incentive'
            WHEN F.HAS_SUPPLY_CHAIN_SIGNAL THEN 'Supply chain localization'
            WHEN F.HAS_WORKFORCE_SIGNAL THEN 'Job creation / workforce'
            WHEN F.HAS_MANUFACTURING_SIGNAL
             AND REGEXP_LIKE(F.TITLE_LC, '.*\\b(new site|new plant|construction|build|building|campus)\\b.*')
                THEN 'New facility'
            WHEN F.HAS_MANUFACTURING_SIGNAL THEN 'Manufacturing expansion'
            WHEN F.HAS_CAPACITY_SIGNAL THEN 'Capacity expansion'
            WHEN F.HAS_CDMO_SIGNAL
             AND (F.HAS_MANUFACTURING_SIGNAL OR F.HAS_CAPACITY_SIGNAL OR F.HAS_INVESTMENT_SIGNAL)
                THEN 'CDMO investment'
            ELSE 'Other US investment signal'
        END AS US_INVESTMENT_TYPE,

        CASE
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bnorth carolina\\b.*') THEN 'North Carolina'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bmassachusetts\\b.*') THEN 'Massachusetts'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bcalifornia\\b.*') THEN 'California'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\btexas\\b.*') THEN 'Texas'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bnew jersey\\b.*') THEN 'New Jersey'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bpennsylvania\\b.*') THEN 'Pennsylvania'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bmaryland\\b.*') THEN 'Maryland'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bindiana\\b.*') THEN 'Indiana'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bwashington\\b.*') THEN 'Washington'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bohio\\b.*') THEN 'Ohio'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bgeorgia\\b.*') THEN 'Georgia'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\b(boston|cambridge)\\b.*') THEN 'Massachusetts'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\b(rtp|research triangle park|durham|raleigh)\\b.*') THEN 'North Carolina'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bnew york\\b.*') THEN 'New York'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bsan diego\\b.*') THEN 'California'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\bphiladelphia\\b.*') THEN 'Pennsylvania'
            WHEN REGEXP_LIKE(F.TITLE_LC, '.*\\b(united states|u\\.s\\.|u\\.s|usa|america|american)\\b.*') THEN 'United States'

            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bnorth carolina\\b.*') THEN 'North Carolina (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bmassachusetts\\b.*') THEN 'Massachusetts (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bcalifornia\\b.*') THEN 'California (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\btexas\\b.*') THEN 'Texas (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bnew jersey\\b.*') THEN 'New Jersey (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bpennsylvania\\b.*') THEN 'Pennsylvania (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bmaryland\\b.*') THEN 'Maryland (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bindiana\\b.*') THEN 'Indiana (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bwashington\\b.*') THEN 'Washington (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bohio\\b.*') THEN 'Ohio (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bgeorgia\\b.*') THEN 'Georgia (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\b(boston|cambridge)\\b.*') THEN 'Massachusetts (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\b(rtp|research triangle park|durham|raleigh)\\b.*') THEN 'North Carolina (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bnew york\\b.*') THEN 'New York (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bsan diego\\b.*') THEN 'California (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\bphiladelphia\\b.*') THEN 'Pennsylvania (context)'
            WHEN REGEXP_LIKE(F.LIMITED_CONTEXT_LC, '.*\\b(united states|u\\.s\\.|u\\.s|usa|america|american)\\b.*') THEN 'United States (context)'

            ELSE 'No explicit US location'
        END AS US_LOCATION_SIGNAL,

        TRIM(
            IFF(F.US_INVESTMENT_MATCH_LEVEL = 'STRICT_TITLE_MATCH', 'Strict title match; ', '') ||
            IFF(F.US_INVESTMENT_MATCH_LEVEL = 'STRICT_POLICY_OR_SUPPLY_CHAIN_MATCH', 'Strict policy/supply-chain match; ', '') ||
            IFF(F.US_INVESTMENT_MATCH_LEVEL = 'REVIEW_CONTEXT_LOCATION_MATCH', 'Review match: US location found in limited context; ', '') ||
            IFF(F.HAS_US_LOCATION_IN_TITLE, 'US location in title; ', '') ||
            IFF(F.HAS_US_LOCATION_IN_CONTEXT AND NOT F.HAS_US_LOCATION_IN_TITLE, 'US location in limited context; ', '') ||
            IFF(F.HAS_MANUFACTURING_SIGNAL, 'Manufacturing/facility signal in title; ', '') ||
            IFF(F.HAS_CAPACITY_SIGNAL, 'Capacity/expansion signal in title; ', '') ||
            IFF(F.HAS_INVESTMENT_SIGNAL, 'Investment/capex/funding signal in title; ', '') ||
            IFF(F.HAS_WORKFORCE_SIGNAL, 'Workforce/headcount signal in title; ', '') ||
            IFF(F.HAS_SUPPLY_CHAIN_SIGNAL, 'Supply-chain/localization/tariff signal in title; ', '') ||
            IFF(F.HAS_POLICY_SIGNAL, 'Policy/incentive/funding signal in title; ', '') ||
            IFF(F.HAS_CDMO_SIGNAL, 'CDMO signal; ', '') ||
            IFF(F.DEAL_VALUE_KEY IS NOT NULL, 'Monetary value detected; ', '')
        ) AS INVESTMENT_RELEVANCE_REASON

    FROM FILTERED F
),

DEDUPED AS (
    SELECT
        S.RECEIVED_TS_PARSED,
        S.PUBLISH_DATE,
        S.PUBLISH_DATE_SOURCE,
        S.EMAIL_SOURCE_TYPE,
        S.ARTICLE_TITLE_CLEAN,
        S.ARTICLE_URL,
        S.PRIORITY_TIER,
        S.SIGNAL_SCORE,
        S.SIGNAL_REASONS,
        S.MATCHED_COMPANIES,
        S.MATCHED_COMPANY_CATEGORIES,
        S.DEAL_VALUE_KEY,
        S.US_INVESTMENT_SCORE,
        S.US_INVESTMENT_TYPE,
        S.US_LOCATION_SIGNAL,
        S.INVESTMENT_RELEVANCE_REASON,
        S.US_INVESTMENT_MATCH_LEVEL,

        LOWER(
            TRIM(
                REGEXP_REPLACE(
                    REGEXP_REPLACE(S.ARTICLE_TITLE_CLEAN, '[^a-zA-Z0-9 ]', ' '),
                    '\\s+',
                    ' '
                )
            )
        ) AS STORY_TITLE_KEY,

        ROW_NUMBER() OVER (
            PARTITION BY
                LOWER(
                    TRIM(
                        REGEXP_REPLACE(
                            REGEXP_REPLACE(S.ARTICLE_TITLE_CLEAN, '[^a-zA-Z0-9 ]', ' '),
                            '\\s+',
                            ' '
                        )
                    )
                )
            ORDER BY
                S.US_INVESTMENT_SCORE DESC,
                CASE
                    WHEN S.PRIORITY_TIER = 'VERY_IMPORTANT' THEN 1
                    WHEN S.PRIORITY_TIER = 'IMPORTANT' THEN 2
                    ELSE 3
                END,
                S.SIGNAL_SCORE DESC,
                S.RECEIVED_TS_PARSED DESC NULLS LAST
        ) AS STORY_RN

    FROM SCORING S
),

FINAL_RANKED AS (
    SELECT
        D.RECEIVED_TS_PARSED,
        D.PUBLISH_DATE,
        D.PUBLISH_DATE_SOURCE,
        D.EMAIL_SOURCE_TYPE,
        D.ARTICLE_TITLE_CLEAN AS ARTICLE_TITLE,
        D.ARTICLE_URL,
        D.PRIORITY_TIER,
        D.SIGNAL_SCORE,
        D.SIGNAL_REASONS,
        D.MATCHED_COMPANIES,
        D.MATCHED_COMPANY_CATEGORIES,
        D.DEAL_VALUE_KEY,
        D.US_INVESTMENT_SCORE,
        D.US_INVESTMENT_TYPE,
        D.US_LOCATION_SIGNAL,
        D.INVESTMENT_RELEVANCE_REASON,
        D.US_INVESTMENT_MATCH_LEVEL,

        ROW_NUMBER() OVER (
            ORDER BY
                D.US_INVESTMENT_SCORE DESC,
                CASE
                    WHEN D.US_INVESTMENT_MATCH_LEVEL = 'STRICT_TITLE_MATCH' THEN 1
                    WHEN D.US_INVESTMENT_MATCH_LEVEL = 'STRICT_POLICY_OR_SUPPLY_CHAIN_MATCH' THEN 2
                    WHEN D.US_INVESTMENT_MATCH_LEVEL = 'REVIEW_CONTEXT_LOCATION_MATCH' THEN 3
                    ELSE 4
                END,
                CASE
                    WHEN D.PRIORITY_TIER = 'VERY_IMPORTANT' THEN 1
                    WHEN D.PRIORITY_TIER = 'IMPORTANT' THEN 2
                    ELSE 3
                END,
                IFF(D.DEAL_VALUE_KEY IS NOT NULL, 1, 0) DESC,
                D.SIGNAL_SCORE DESC,
                D.RECEIVED_TS_PARSED DESC NULLS LAST
        ) AS US_INVESTMENT_RANK

    FROM DEDUPED D
    WHERE D.STORY_RN = 1
)

SELECT
    RECEIVED_TS_PARSED,
    PUBLISH_DATE,
    PUBLISH_DATE_SOURCE,
    EMAIL_SOURCE_TYPE,
    ARTICLE_TITLE,
    ARTICLE_URL,
    PRIORITY_TIER,
    SIGNAL_SCORE,
    SIGNAL_REASONS,
    MATCHED_COMPANIES,
    MATCHED_COMPANY_CATEGORIES,
    DEAL_VALUE_KEY,
    US_INVESTMENT_SCORE,
    US_INVESTMENT_TYPE,
    US_LOCATION_SIGNAL,
    INVESTMENT_RELEVANCE_REASON,
    US_INVESTMENT_MATCH_LEVEL,
    US_INVESTMENT_RANK

FROM FINAL_RANKED;

SELECT
    US_INVESTMENT_RANK,
    ARTICLE_TITLE,
    EMAIL_SOURCE_TYPE,
    PUBLISH_DATE,
    MATCHED_COMPANIES,
    DEAL_VALUE_KEY,
    US_INVESTMENT_TYPE,
    US_INVESTMENT_SCORE,
    US_LOCATION_SIGNAL,
    US_INVESTMENT_MATCH_LEVEL,
    INVESTMENT_RELEVANCE_REASON
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_US_INVESTMENT_TRACKER
WHERE US_INVESTMENT_RANK <= 30
ORDER BY US_INVESTMENT_RANK;
