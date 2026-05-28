-- ============================================================
-- PHARMA NEWS MONITORING MVP - SAFE TEAM SANDBOX VERSION
-- Database: PHARMA_NEWS_SANDBOX
-- Schema: NEWS
-- Warehouse: SANDBOX_WH
-- ============================================================


-- ============================================================
-- 00 - Set context
-- ============================================================

USE ROLE SANDBOX_DEVELOPER;
USE WAREHOUSE SANDBOX_WH;

CREATE DATABASE IF NOT EXISTS PHARMA_NEWS_SANDBOX;
CREATE SCHEMA IF NOT EXISTS PHARMA_NEWS_SANDBOX.NEWS;

USE DATABASE PHARMA_NEWS_SANDBOX;
USE SCHEMA NEWS;


-- ============================================================
-- 00A - Staging table for Alteryx output
-- IMPORTANT: IF NOT EXISTS prevents deleting Alteryx-loaded data
-- ============================================================

CREATE TABLE IF NOT EXISTS PHARMA_NEWS_SANDBOX.NEWS.STG_PHARMA_NEWS_ARTICLES_V1 (
    MESSAGE_ID STRING,
    SENDER_NAME STRING,
    SENDER_EMAIL STRING,
    SUBJECT_RAW STRING,
    RECEIVED_TS STRING,
    EMAIL_SOURCE_TYPE STRING,
    ARTICLE_RANK NUMBER,
    ARTICLE_TITLE STRING,
    ARTICLE_URL STRING,
    ARTICLE_URL_EXTRACTION_METHOD STRING,
    BODY_BEST STRING,
    ARTICLE_LLM_INPUT STRING,
    PARSER_VERSION STRING
);


-- ============================================================
-- 00B - Tracked Keyword Logic Reference
-- ============================================================

CREATE OR REPLACE TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_PRIORITY_KEYWORDS AS
SELECT
    CATEGORY::STRING AS CATEGORY,
    KEYWORD::STRING AS KEYWORD,
    CRITERIA::STRING AS CRITERIA
FROM VALUES
    ('Value', 'value >$1B', 'always'),
    ('Size', 'expansion/acquisition/construction', 'includes Top 25 or CDMO, or >$500M value'),
    ('Size', 'new capability', 'for a known CDMO'),
    ('Size', 'closing/shuttering/divestment', 'entire site or business unit of Top 25 or CDMO'),
    ('Size', 'layoffs', 'includes Top 25 or CDMO, or >200 employees'),
    ('Regulatory', 'most favored nation', 'always'),
    ('Regulatory', 'tariffs', 'includes pharma'),
    ('Regulatory', 'FDA', 'leadership changes, or includes biopharma or biologics regulations; NOT updates on every clinical trial'),
    ('Regulatory', 'regulation', 'from FDA or EU and for biopharma/biotech/biologics/gene therapy'),
    ('Regulatory', 'Supreme Court', 'includes biopharma or biologics keywords'),
    ('Modality', 'ADC', 'regulations, >$200M value, trends'),
    ('Modality', 'microbial', 'includes Top 25 or CDMO, or >$200M value'),
    ('Modality', 'biosimilars', 'regulations, >$200M value, trends'),
    ('Location', 'Denmark', 'includes Top 25 or CDMO, or >$200M value'),
    ('Location', 'UK or Britain', 'includes Top 25 or CDMO, or >$200M value'),
    ('Location', 'US, NC / RTP / Triangle / Raleigh / Durham / Cary / Apex', 'includes Top 25 or CDMO, or >$500M value'),
    ('Location', 'US, TX / College Station', 'includes Top 25 or CDMO, or >$200M value'),
    ('Companies', 'CDMO keyword', 'always'),
    ('Companies', 'Top 25 companies', 'as listed or if investment >$1B'),
    ('Companies', 'CDMO companies', 'always')
AS V(CATEGORY, KEYWORD, CRITERIA);


-- ============================================================
-- 00C - Tracked Companies
-- ============================================================

CREATE OR REPLACE TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_TRACKED_COMPANIES AS
SELECT
    COMPANY_NAME::STRING AS COMPANY_NAME,
    COMPANY_CATEGORY::STRING AS COMPANY_CATEGORY
FROM VALUES
    ('Eli Lilly', 'Top 25'),
    ('Lilly', 'Top 25'),
    ('Novo Nordisk', 'Top 25'),
    ('Johnson & Johnson', 'Top 25'),
    ('J&J', 'Top 25'),
    ('Merck & Co', 'Top 25'),
    ('Merck', 'Top 25'),
    ('AbbVie', 'Hybrid, Top 25'),
    ('AstraZeneca', 'Top 25'),
    ('Roche', 'Top 25'),
    ('Novartis', 'Top 25'),
    ('Amgen', 'Top 25'),
    ('Pfizer', 'Hybrid, Top 25'),
    ('Sanofi', 'Top 25'),
    ('Vertex Pharma', 'Top 25'),
    ('Vertex', 'Top 25'),
    ('Regeneron', 'Top 25'),
    ('CSL', 'Top 25'),
    ('Gilead', 'Top 25'),
    ('Bristol Myers Squibb', 'Top 25'),
    ('BMS', 'Top 25'),
    ('GSK', 'Top 25'),
    ('Merck KGaA', 'Hybrid, Top 25'),
    ('MilliporeSigma', 'Hybrid, Top 25'),
    ('Daiichi Sankyo', 'Top 25'),
    ('Chugai', 'Top 25'),
    ('Sun Pharma', 'Top 25'),
    ('Takeda', 'Top 25'),
    ('Moderna', 'Top 25'),
    ('Biogen', 'Top 25'),
    ('Alnylam Pharma', 'Top 25'),
    ('Alnylam', 'Top 25'),
    ('UCB', 'Top 25'),

    ('Lonza', 'CDMO'),
    ('WuXi Biologics', 'CDMO'),
    ('Wuxi Biologics', 'CDMO'),
    ('Samsung Biologics', 'CDMO'),
    ('Catalent', 'CDMO'),
    ('Patheon', 'CDMO'),
    ('Thermo Fisher', 'CDMO'),
    ('ThermoFisher', 'CDMO'),
    ('Recipharm', 'CDMO'),
    ('Lotte', 'CDMO'),
    ('Lotte Biologics', 'CDMO'),
    ('Simtra BioPharma', 'CDMO'),
    ('Wacker', 'CDMO'),
    ('Vetter', 'CDMO'),
    ('Eurofins', 'CDMO'),
    ('KBI Biopharma', 'CDMO'),
    ('Aldevron', 'CDMO'),
    ('Ajinomoto BioPharma Services', 'CDMO'),
    ('AGC Biologics', 'CDMO'),
    ('Syngene', 'CDMO'),
    ('Abzena', 'CDMO'),
    ('PCI Pharma', 'CDMO'),
    ('Rentschler Biopharma', 'CDMO')
AS V(COMPANY_NAME, COMPANY_CATEGORY);


-- ============================================================
-- 01 - Clean Article View
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_ARTICLES_CLEAN AS
SELECT
    MESSAGE_ID,
    SENDER_NAME,
    SENDER_EMAIL,
    SUBJECT_RAW,
    RECEIVED_TS,
    TRY_TO_TIMESTAMP_TZ(RECEIVED_TS) AS RECEIVED_TS_PARSED,
    EMAIL_SOURCE_TYPE,
    ARTICLE_RANK,

    TRIM(
        REGEXP_REPLACE(
            REGEXP_REPLACE(
                REGEXP_REPLACE(ARTICLE_TITLE, '^[0-9]+\\.?\\s*', ''),
                '^,\\s*',
                ''
            ),
            '\\s+',
            ' '
        )
    ) AS ARTICLE_TITLE_CLEAN,

    LOWER(
        TRIM(
            REGEXP_REPLACE(
                REGEXP_REPLACE(
                    REGEXP_REPLACE(ARTICLE_TITLE, '^[0-9]+\\.?\\s*', ''),
                    '^,\\s*',
                    ''
                ),
                '\\s+',
                ' '
            )
        )
    ) AS ARTICLE_TITLE_LC,

    ARTICLE_URL,
    ARTICLE_URL_EXTRACTION_METHOD,
    BODY_BEST,
    ARTICLE_LLM_INPUT,
    PARSER_VERSION
FROM PHARMA_NEWS_SANDBOX.NEWS.STG_PHARMA_NEWS_ARTICLES_V1;


-- ============================================================
-- 02 - Final Gate
-- Removes obvious newsletter noise before AI
-- ============================================================

CREATE OR REPLACE TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_SUBJECT_GATE_FINAL AS
SELECT
    *,
    CASE
        WHEN ARTICLE_TITLE_LC IS NULL THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '^(a message from|brought to you by|by |staff writer|staff writers|senior editor|senior editors|associate editor|associate editors|executive editor|deputy editor|editor-in-chief|publisher|contributing writer|sales director)'
            THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '(unsubscribe|privacy policy|contact support|linkedin logo|facebook logo|twitter logo|youtube logo|brand logo|questex signature)'
            THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '(webinar|podcast|whitepaper|download the white paper|register now|register today|save your spot|conference|exhibition|innovation week|pharma ci|fierce biotech week|outsourcing awards)'
            THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '(sponsored by|explore our services|move to market with confidence|reliable delivery|global regulatory expertise|scalable gmp manufacturing)'
            THEN 'DROP'

        WHEN ARTICLE_TITLE_LC LIKE '[%' THEN 'DROP'
        WHEN ARTICLE_TITLE_LC LIKE 'http%' THEN 'DROP'
        WHEN ARTICLE_TITLE_LC LIKE 'click here%' THEN 'DROP'
        WHEN LENGTH(ARTICLE_TITLE_CLEAN) < 20 THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '(acquisition|buyout|deal|fda|approval|manufacturing|facility|expansion|phase 3|partnership|ipo|fundraising|investment|capacity|tariffs|regulation|supreme court|most favored nation|shortage|shortages|layoffs|divestment|closing|shuttering)'
            THEN 'PASS'

        ELSE 'REVIEW'
    END AS SUBJECT_GATE_FINAL
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_ARTICLES_CLEAN;


-- ============================================================
-- 03 - Relevance
-- AI evaluates only non-drop rows
-- ============================================================

CREATE OR REPLACE TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_RELEVANCE AS
SELECT
    *,
    AI_FILTER(
        PROMPT(
            'Return TRUE if this pharma news item is relevant for a Business Intelligence & Insights team at a biologics/CDMO company. Relevant examples include competitor investments, manufacturing, site expansions, partnerships, acquisitions, customer/commercial signals, financing, IPOs, regulatory milestones, clinical milestones with business impact, supply chain changes, platform/capability updates, and strategic market signals. Not relevant examples include webinars, podcasts, events, whitepapers, sponsor messages, logos, footer links, unsubscribe links, admin content, editorial staff listings, and generic promotions. Subject: {0}. Title: {1}. URL: {2}. Body: {3}',
            COALESCE(SUBJECT_RAW, ''),
            COALESCE(ARTICLE_TITLE_CLEAN, ''),
            COALESCE(ARTICLE_URL, ''),
            COALESCE(LEFT(BODY_BEST, 3000), '')
        )
    ) AS IS_RELEVANT
FROM PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_SUBJECT_GATE_FINAL
WHERE SUBJECT_GATE_FINAL IN ('PASS', 'REVIEW');


-- ============================================================
-- 04 - Classification
-- AI classifies relevant rows into business categories
-- ============================================================

CREATE OR REPLACE TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_CLASSIFIED AS
SELECT
    *,
    AI_CLASSIFY(
        CONCAT(
            'Subject: ', COALESCE(SUBJECT_RAW, ''), '. ',
            'Title: ', COALESCE(ARTICLE_TITLE_CLEAN, ''), '. ',
            'URL: ', COALESCE(ARTICLE_URL, ''), '. ',
            'Body: ', COALESCE(LEFT(BODY_BEST, 3000), '')
        ),
        [
            {'label': 'competitor_investment_capacity', 'description': 'facility build, site expansion, capex, added manufacturing capacity, major investment'},
            {'label': 'partnership_ma', 'description': 'acquisition, merger, strategic partnership, licensing, collaboration, supply agreement'},
            {'label': 'capability_modality', 'description': 'new manufacturing capability, modality, platform, fill-finish, analytical, microbial, mammalian, cell therapy, gene therapy'},
            {'label': 'clinical_regulatory_signal', 'description': 'phase advancement, approval, filing, warning letter, inspection, regulatory or clinical milestone with strategic impact'},
            {'label': 'policy_market_signal', 'description': 'trade, tariffs, policy, FDA framework, reshoring, Supreme Court, manufacturing autonomy, drug shortages, macro signal affecting pharma manufacturing or the market'},
            {'label': 'commercial_customer_signal', 'description': 'customer win, launch, commercial supply, demand signal, backlog, manufacturing award'},
            {'label': 'financing_market_signal', 'description': 'IPO, fundraising, financing, public offering, major market signal'}
        ],
        {
            'task_description': 'Classify relevant pharma-news items for a business intelligence team',
            'output_mode': 'multi'
        }
    ) AS CATEGORY_RESULT
FROM PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_RELEVANCE
WHERE IS_RELEVANT = TRUE;


-- ============================================================
-- 05 - Priority Tier V2
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2 AS
WITH BASE AS (
    SELECT
        C.*,
        LOWER(
            COALESCE(C.ARTICLE_TITLE_CLEAN, '') || ' ' ||
            COALESCE(C.SUBJECT_RAW, '') || ' ' ||
            COALESCE(C.BODY_BEST, '')
        ) AS FULL_TEXT_LC
    FROM PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_CLASSIFIED C
    WHERE C.IS_RELEVANT = TRUE
),

COMPANY_MATCHES AS (
    SELECT
        B.MESSAGE_ID,
        B.ARTICLE_TITLE_CLEAN,
        LISTAGG(DISTINCT TC.COMPANY_NAME, ', ') WITHIN GROUP (ORDER BY TC.COMPANY_NAME) AS MATCHED_COMPANIES,
        LISTAGG(DISTINCT TC.COMPANY_CATEGORY, ', ') WITHIN GROUP (ORDER BY TC.COMPANY_CATEGORY) AS MATCHED_COMPANY_CATEGORIES
    FROM BASE B
    LEFT JOIN PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_TRACKED_COMPANIES TC
        ON B.FULL_TEXT_LC ILIKE '%' || LOWER(TC.COMPANY_NAME) || '%'
    GROUP BY
        B.MESSAGE_ID,
        B.ARTICLE_TITLE_CLEAN
),

ENRICHED_1 AS (
    SELECT
        B.*,
        COALESCE(CM.MATCHED_COMPANIES, '') AS MATCHED_COMPANIES,
        COALESCE(CM.MATCHED_COMPANY_CATEGORIES, '') AS MATCHED_COMPANY_CATEGORIES,

        REGEXP_SUBSTR(
            B.FULL_TEXT_LC,
            '(\\$|€|£)[0-9]+(\\.[0-9]+)?\\s?(b|m)|[0-9]+(\\.[0-9]+)?\\s?(billion|million)'
        ) AS DEAL_VALUE_KEY,

        CASE
            WHEN B.FULL_TEXT_LC RLIKE '(\\$|€|£)[0-9]+(\\.[0-9]+)?\\s?b'
              OR B.FULL_TEXT_LC RLIKE '[0-9]+(\\.[0-9]+)?\\s?billion'
            THEN TRUE ELSE FALSE
        END AS VALUE_ABOVE_1B,

        CASE
            WHEN B.FULL_TEXT_LC RLIKE '(\\$|€|£)[5-9][0-9]{2}\\s?m'
              OR B.FULL_TEXT_LC RLIKE '(\\$|€|£)[0-9]+(\\.[0-9]+)?\\s?b'
              OR B.FULL_TEXT_LC RLIKE '[5-9][0-9]{2}\\s?million'
              OR B.FULL_TEXT_LC RLIKE '[0-9]+(\\.[0-9]+)?\\s?billion'
            THEN TRUE ELSE FALSE
        END AS VALUE_ABOVE_500M,

        CASE
            WHEN B.FULL_TEXT_LC RLIKE '(\\$|€|£)[2-9][0-9]{2}\\s?m'
              OR B.FULL_TEXT_LC RLIKE '(\\$|€|£)[0-9]+(\\.[0-9]+)?\\s?b'
              OR B.FULL_TEXT_LC RLIKE '[2-9][0-9]{2}\\s?million'
              OR B.FULL_TEXT_LC RLIKE '[0-9]+(\\.[0-9]+)?\\s?billion'
            THEN TRUE ELSE FALSE
        END AS VALUE_ABOVE_200M

    FROM BASE B
    LEFT JOIN COMPANY_MATCHES CM
        ON B.MESSAGE_ID = CM.MESSAGE_ID
       AND B.ARTICLE_TITLE_CLEAN = CM.ARTICLE_TITLE_CLEAN
),

ENRICHED_2 AS (
    SELECT
        *,

        CASE WHEN FULL_TEXT_LC RLIKE '(acquisition|buyout|merger|deal|licensing|collaboration|partnership)'
            THEN TRUE ELSE FALSE END AS IS_DEAL_SIGNAL,

        CASE WHEN FULL_TEXT_LC RLIKE '(expansion|construction|facility|site|manufacturing|capacity|capex|investment)'
            THEN TRUE ELSE FALSE END AS IS_SIZE_OR_CAPACITY_SIGNAL,

        CASE WHEN FULL_TEXT_LC RLIKE '(new capability|new platform|new modality|fill-finish|fill finish|microbial|mammalian|cell therapy|gene therapy|adc|biosimilar|biosimilars)'
            THEN TRUE ELSE FALSE END AS IS_NEW_CAPABILITY_SIGNAL,

        CASE WHEN FULL_TEXT_LC RLIKE '(closing|shuttering|divestment|divestiture|site closure|plant closure|business unit)'
            THEN TRUE ELSE FALSE END AS IS_SITE_OR_BUSINESS_UNIT_NEGATIVE_SIGNAL,

        CASE
            WHEN FULL_TEXT_LC RLIKE '(layoffs|job cuts|cut [0-9]+ jobs|workers|employees)'
             AND FULL_TEXT_LC RLIKE '([2-9][0-9]{2,}\\s?(employees|jobs|workers|staff|positions))'
            THEN TRUE ELSE FALSE
        END AS IS_LAYOFF_OVER_200_SIGNAL,

        CASE WHEN FULL_TEXT_LC RLIKE '(most favored nation|most-favored nation)'
            THEN TRUE ELSE FALSE END AS IS_MFN_SIGNAL,

        CASE
            WHEN FULL_TEXT_LC RLIKE '(tariff|tariffs)'
             AND FULL_TEXT_LC RLIKE '(pharma|biopharma|biotech|biologics|drug|medicine)'
            THEN TRUE ELSE FALSE
        END AS IS_TARIFF_SIGNAL,

        CASE WHEN FULL_TEXT_LC RLIKE '(fda commissioner|fda leadership|fda framework|fda regulation|fda regulations|biopharma regulation|biologics regulation|gene therapy regulation)'
            THEN TRUE ELSE FALSE END AS IS_FDA_LEADERSHIP_OR_REGULATION_SIGNAL,

        CASE
            WHEN FULL_TEXT_LC RLIKE '(regulation|regulatory|eu scheme|eu advances|manufacturing autonomy|drug shortages|shortages)'
             AND FULL_TEXT_LC RLIKE '(fda|eu|biopharma|biotech|biologics|gene therapy|manufacturing)'
            THEN TRUE ELSE FALSE
        END AS IS_REGULATION_SIGNAL,

        CASE
            WHEN FULL_TEXT_LC RLIKE '(supreme court)'
             AND FULL_TEXT_LC RLIKE '(biopharma|biologics|biotech|pharma|drug|medicine)'
            THEN TRUE ELSE FALSE
        END AS IS_SUPREME_COURT_SIGNAL,

        CASE
            WHEN FULL_TEXT_LC RLIKE '(adc|antibody-drug conjugate|antibody drug conjugate)'
             AND (VALUE_ABOVE_200M OR FULL_TEXT_LC RLIKE '(regulation|regulatory|trend|trends|market|capacity|manufacturing)')
            THEN TRUE ELSE FALSE
        END AS IS_ADC_SIGNAL,

        CASE
            WHEN FULL_TEXT_LC RLIKE '(microbial)'
             AND (VALUE_ABOVE_200M OR MATCHED_COMPANY_CATEGORIES ILIKE '%Top 25%' OR MATCHED_COMPANY_CATEGORIES ILIKE '%CDMO%')
            THEN TRUE ELSE FALSE
        END AS IS_MICROBIAL_SIGNAL,

        CASE
            WHEN FULL_TEXT_LC RLIKE '(biosimilar|biosimilars)'
             AND (VALUE_ABOVE_200M OR FULL_TEXT_LC RLIKE '(regulation|regulatory|trend|trends|market|capacity|manufacturing)')
            THEN TRUE ELSE FALSE
        END AS IS_BIOSIMILAR_SIGNAL,

        CASE
            WHEN FULL_TEXT_LC RLIKE '(denmark|danish)'
             AND (VALUE_ABOVE_200M OR MATCHED_COMPANY_CATEGORIES ILIKE '%Top 25%' OR MATCHED_COMPANY_CATEGORIES ILIKE '%CDMO%')
            THEN TRUE ELSE FALSE
        END AS IS_DENMARK_SIGNAL,

        CASE
            WHEN FULL_TEXT_LC RLIKE '(uk|britain|british|united kingdom)'
             AND (VALUE_ABOVE_200M OR MATCHED_COMPANY_CATEGORIES ILIKE '%Top 25%' OR MATCHED_COMPANY_CATEGORIES ILIKE '%CDMO%')
            THEN TRUE ELSE FALSE
        END AS IS_UK_SIGNAL,

        CASE
            WHEN FULL_TEXT_LC RLIKE '(north carolina|rtp|triangle|raleigh|durham|cary|apex)'
             AND (VALUE_ABOVE_500M OR MATCHED_COMPANY_CATEGORIES ILIKE '%Top 25%' OR MATCHED_COMPANY_CATEGORIES ILIKE '%CDMO%')
            THEN TRUE ELSE FALSE
        END AS IS_NC_SIGNAL,

        CASE
            WHEN FULL_TEXT_LC RLIKE '(college station|texas|tx)'
             AND (VALUE_ABOVE_200M OR MATCHED_COMPANY_CATEGORIES ILIKE '%Top 25%' OR MATCHED_COMPANY_CATEGORIES ILIKE '%CDMO%')
            THEN TRUE ELSE FALSE
        END AS IS_TX_SIGNAL,

        CASE
            WHEN FULL_TEXT_LC RLIKE '(webinar|register now|register today|whitepaper|sponsored by|explore our services|move to market with confidence|podcast|conference|event|logo|unsubscribe)'
            THEN TRUE ELSE FALSE
        END AS IS_PROMOTIONAL_SIGNAL

    FROM ENRICHED_1
)

SELECT
    *,
    CASE
        WHEN IS_PROMOTIONAL_SIGNAL THEN 'DROP'

        WHEN VALUE_ABOVE_1B THEN 'VERY_IMPORTANT'

        WHEN FULL_TEXT_LC RLIKE '\\bcdmo\\b'
             AND NOT IS_PROMOTIONAL_SIGNAL
        THEN 'VERY_IMPORTANT'

        WHEN MATCHED_COMPANY_CATEGORIES ILIKE '%CDMO%'
             AND (
                    IS_SIZE_OR_CAPACITY_SIGNAL
                 OR IS_DEAL_SIGNAL
                 OR IS_NEW_CAPABILITY_SIGNAL
                 OR IS_SITE_OR_BUSINESS_UNIT_NEGATIVE_SIGNAL
                 OR IS_LAYOFF_OVER_200_SIGNAL
             )
        THEN 'VERY_IMPORTANT'

        WHEN MATCHED_COMPANY_CATEGORIES ILIKE '%Top 25%'
             AND (
                    VALUE_ABOVE_500M
                 OR IS_DEAL_SIGNAL
                 OR IS_SIZE_OR_CAPACITY_SIGNAL
                 OR IS_SITE_OR_BUSINESS_UNIT_NEGATIVE_SIGNAL
                 OR IS_LAYOFF_OVER_200_SIGNAL
                 OR IS_MFN_SIGNAL
                 OR IS_TARIFF_SIGNAL
                 OR IS_FDA_LEADERSHIP_OR_REGULATION_SIGNAL
                 OR IS_REGULATION_SIGNAL
                 OR IS_SUPREME_COURT_SIGNAL
             )
        THEN 'VERY_IMPORTANT'

        WHEN IS_MFN_SIGNAL
          OR IS_TARIFF_SIGNAL
          OR IS_FDA_LEADERSHIP_OR_REGULATION_SIGNAL
          OR IS_REGULATION_SIGNAL
          OR IS_SUPREME_COURT_SIGNAL
        THEN 'VERY_IMPORTANT'

        WHEN IS_ADC_SIGNAL
          OR IS_MICROBIAL_SIGNAL
          OR IS_BIOSIMILAR_SIGNAL
          OR IS_DENMARK_SIGNAL
          OR IS_UK_SIGNAL
          OR IS_NC_SIGNAL
          OR IS_TX_SIGNAL
        THEN 'IMPORTANT'

        WHEN SUBJECT_GATE_FINAL = 'PASS' THEN 'IMPORTANT'

        ELSE 'MONITOR'
    END AS PRIORITY_TIER
FROM ENRICHED_2;


-- ============================================================
-- 06 - Digest Queue V2
-- Deduplication before email output
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2 AS
WITH CANDIDATES AS (
    SELECT
        RECEIVED_TS_PARSED,
        EMAIL_SOURCE_TYPE,
        ARTICLE_TITLE_CLEAN AS ARTICLE_TITLE,
        ARTICLE_URL,
        PRIORITY_TIER,
        MATCHED_COMPANIES,
        MATCHED_COMPANY_CATEGORIES,
        CATEGORY_RESULT,
        FULL_TEXT_LC,
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
      AND PRIORITY_TIER <> 'DROP'
      AND NOT FULL_TEXT_LC RLIKE '(sponsored by|explore our services|move to market with confidence|webinar|register today|register now|whitepaper|podcast|conference)'
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
                CASE WHEN PRIORITY_TIER = 'VERY_IMPORTANT' THEN 1 ELSE 2 END,
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
    MATCHED_COMPANIES,
    MATCHED_COMPANY_CATEGORIES,
    CATEGORY_RESULT,
    FULL_TEXT_LC,
    DEAL_VALUE_KEY,
    STORY_KEY
FROM DEDUPED
WHERE RN = 1;


-- ============================================================
-- 07 - Email Notification Integration
-- Keep commented unless your role has permission
-- ============================================================

-- CREATE OR REPLACE NOTIFICATION INTEGRATION PHARMA_NEWS_EMAIL_INT
--     TYPE = EMAIL
--     ENABLED = TRUE
--     ALLOWED_RECIPIENTS = ('s235701@dtu.dk');


-- ============================================================
-- 08 - Stored Procedure: Daily Pharma News Digest V2
-- P_DAYS_BACK = 1 for daily production
-- P_DAYS_BACK = 7 or 30 for testing/demo
-- ============================================================

CREATE OR REPLACE PROCEDURE PHARMA_NEWS_SANDBOX.NEWS.SP_SEND_DAILY_PHARMA_NEWS_DIGEST_V2(P_DAYS_BACK NUMBER)
RETURNS STRING
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
    V_TOTAL_COUNT NUMBER DEFAULT 0;
    V_VERY_IMPORTANT_COUNT NUMBER DEFAULT 0;
    V_IMPORTANT_COUNT NUMBER DEFAULT 0;
    V_BODY STRING DEFAULT '';
BEGIN

    V_TOTAL_COUNT := (
        SELECT COUNT(*)
        FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
        WHERE RECEIVED_TS_PARSED >= DATEADD('day', -1 * :P_DAYS_BACK, CURRENT_TIMESTAMP())
    );

    V_VERY_IMPORTANT_COUNT := (
        SELECT COUNT(*)
        FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
        WHERE RECEIVED_TS_PARSED >= DATEADD('day', -1 * :P_DAYS_BACK, CURRENT_TIMESTAMP())
          AND PRIORITY_TIER = 'VERY_IMPORTANT'
    );

    V_IMPORTANT_COUNT := (
        SELECT COUNT(*)
        FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
        WHERE RECEIVED_TS_PARSED >= DATEADD('day', -1 * :P_DAYS_BACK, CURRENT_TIMESTAMP())
          AND PRIORITY_TIER = 'IMPORTANT'
    );

    IF (V_TOTAL_COUNT = 0) THEN
        RETURN 'No pharma news items to send.';
    END IF;

    V_BODY := (
        WITH VERY_IMPORTANT_TOP AS (
            SELECT *
            FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
            WHERE RECEIVED_TS_PARSED >= DATEADD('day', -1 * :P_DAYS_BACK, CURRENT_TIMESTAMP())
              AND PRIORITY_TIER = 'VERY_IMPORTANT'
            ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
            LIMIT 5
        ),

        VERY_IMPORTANT_ANALYZED AS (
            SELECT
                *,
                AI_COMPLETE(
                    model => 'llama3.3-70b',
                    prompt => CONCAT(
                        'You are supporting a Business Intelligence & Insights team at a biologics/CDMO company. ',
                        'Write a useful 2-3 paragraph explanation of why this news matters. ',
                        'Use simple language. Focus on business impact, competitors, manufacturing, CDMO relevance, regulatory impact, customers, or capacity. ',
                        'Only include a recommended action if there is a clear action worth taking. ',
                        'Return structured output only. ',
                        'Title: ', COALESCE(ARTICLE_TITLE, ''), '. ',
                        'Source: ', COALESCE(EMAIL_SOURCE_TYPE, ''), '. ',
                        'Matched companies: ', COALESCE(MATCHED_COMPANIES, ''), '. ',
                        'Company categories: ', COALESCE(MATCHED_COMPANY_CATEGORIES, ''), '. ',
                        'Categories: ', COALESCE(TO_JSON(CATEGORY_RESULT), ''), '. ',
                        'URL: ', COALESCE(ARTICLE_URL, ''), '. ',
                        'Body/context: ', COALESCE(LEFT(FULL_TEXT_LC, 3000), '')
                    ),
                    response_format => TYPE OBJECT(
                        analysis STRING,
                        recommended_action STRING
                    )
                ) AS DEEP_ANALYSIS_RESULT
            FROM VERY_IMPORTANT_TOP
        ),

        IMPORTANT_TOP AS (
            SELECT *
            FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
            WHERE RECEIVED_TS_PARSED >= DATEADD('day', -1 * :P_DAYS_BACK, CURRENT_TIMESTAMP())
              AND PRIORITY_TIER = 'IMPORTANT'
            ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
            LIMIT 10
        ),

        VERY_IMPORTANT_TEXT AS (
            SELECT
                LISTAGG(
                    ARTICLE_TITLE || CHR(10) ||
                    'Source: ' || COALESCE(EMAIL_SOURCE_TYPE, '') || CHR(10) ||
                    'Matched companies: ' || COALESCE(MATCHED_COMPANIES, 'None') || CHR(10) ||
                    CHR(10) ||
                    COALESCE(DEEP_ANALYSIS_RESULT:analysis::STRING, 'No analysis generated.') || CHR(10) ||
                    CASE
                        WHEN DEEP_ANALYSIS_RESULT:recommended_action::STRING IS NOT NULL
                         AND LENGTH(TRIM(DEEP_ANALYSIS_RESULT:recommended_action::STRING)) > 0
                        THEN CHR(10) || 'Recommended action: ' || DEEP_ANALYSIS_RESULT:recommended_action::STRING || CHR(10)
                        ELSE ''
                    END ||
                    'URL: ' || COALESCE(ARTICLE_URL, '') || CHR(10) ||
                    CHR(10) || '------------------------' || CHR(10),
                    ''
                ) WITHIN GROUP (ORDER BY RECEIVED_TS_PARSED DESC) AS TXT
            FROM VERY_IMPORTANT_ANALYZED
        ),

        IMPORTANT_TEXT AS (
            SELECT
                LISTAGG(
                    '- ' || ARTICLE_TITLE ||
                    ' | Source: ' || COALESCE(EMAIL_SOURCE_TYPE, '') ||
                    ' | Companies: ' || COALESCE(MATCHED_COMPANIES, 'None') ||
                    CHR(10) ||
                    '  URL: ' || COALESCE(ARTICLE_URL, '') ||
                    CHR(10),
                    ''
                ) WITHIN GROUP (ORDER BY RECEIVED_TS_PARSED DESC) AS TXT
            FROM IMPORTANT_TOP
        )

        SELECT
            'Daily Pharma News Digest' || CHR(10) ||
            'Generated from Snowflake action queue' || CHR(10) ||
            'Lookback window: last ' || :P_DAYS_BACK || ' day(s)' || CHR(10) ||
            'Very important items: ' || :V_VERY_IMPORTANT_COUNT || CHR(10) ||
            'Important items: ' || :V_IMPORTANT_COUNT || CHR(10) ||
            CHR(10) ||
            'VERY IMPORTANT NEWS' || CHR(10) ||
            '====================' || CHR(10) ||
            COALESCE((SELECT TXT FROM VERY_IMPORTANT_TEXT), 'No very important items today.') ||
            CHR(10) ||
            'OTHER RELEVANT HEADLINES' || CHR(10) ||
            '========================' || CHR(10) ||
            COALESCE((SELECT TXT FROM IMPORTANT_TEXT), 'No additional important headlines today.')
    );

    CALL SYSTEM$SEND_EMAIL(
        'PHARMA_NEWS_EMAIL_INT',
        's235701@dtu.dk',
        'Daily Pharma News Digest',
        :V_BODY
    );

    RETURN 'Daily pharma news digest V2 sent. Total: ' || V_TOTAL_COUNT ||
           ', Very important: ' || V_VERY_IMPORTANT_COUNT ||
           ', Important: ' || V_IMPORTANT_COUNT ||
           ', Lookback days: ' || P_DAYS_BACK;

EXCEPTION
    WHEN OTHER THEN
        RETURN 'Procedure failed. Most likely email integration, permissions, or Cortex AI issue. Error: ' || SQLERRM;

END;
$$;


-- ============================================================
-- 09 - Daily Email Task
-- Kept commented to avoid permission errors
-- ============================================================

-- CREATE OR REPLACE TASK PHARMA_NEWS_SANDBOX.NEWS.TASK_SEND_DAILY_PHARMA_NEWS_DIGEST_V2
--     WAREHOUSE = SANDBOX_WH
--     SCHEDULE = 'USING CRON 15 8 * * MON-FRI Europe/Copenhagen'
-- AS
--     CALL PHARMA_NEWS_SANDBOX.NEWS.SP_SEND_DAILY_PHARMA_NEWS_DIGEST_V2(1);


-- ============================================================
-- 10 - Debug checks
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

SELECT *
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
LIMIT 50;


-- ============================================================
-- 11 - Optional manual commands
-- Uncomment when needed
-- ============================================================

-- Test digest with last 7 days:
-- CALL PHARMA_NEWS_SANDBOX.NEWS.SP_SEND_DAILY_PHARMA_NEWS_DIGEST_V2(7);

-- Test digest with last 30 days:
-- CALL PHARMA_NEWS_SANDBOX.NEWS.SP_SEND_DAILY_PHARMA_NEWS_DIGEST_V2(30);

-- Test email integration only:
-- CALL SYSTEM$SEND_EMAIL(
--     'PHARMA_NEWS_EMAIL_INT',
--     's235701@dtu.dk',
--     'Test: Pharma News Snowflake Email',
--     'This is a quick test from Snowflake.'
-- );

-- Activate daily task:
-- ALTER TASK PHARMA_NEWS_SANDBOX.NEWS.TASK_SEND_DAILY_PHARMA_NEWS_DIGEST_V2 RESUME;

-- Pause daily task:
-- ALTER TASK PHARMA_NEWS_SANDBOX.NEWS.TASK_SEND_DAILY_PHARMA_NEWS_DIGEST_V2 SUSPEND;

-- Execute task manually:
-- EXECUTE TASK PHARMA_NEWS_SANDBOX.NEWS.TASK_SEND_DAILY_PHARMA_NEWS_DIGEST_V2;
