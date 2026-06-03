-- ============================================================
-- PHARMA NEWS MONITORING MVP - CLEAN FULL WORKFLOW
-- Database: PHARMA_NEWS_SANDBOX
-- Schema: NEWS
-- Warehouse: SANDBOX_WH
--
-- Purpose:
-- 1. Alteryx writes parsed pharma news articles to staging table
-- 2. Snowflake cleans article titles
-- 3. Snowflake removes obvious newsletter noise
-- 4. Snowflake uses AI to evaluate relevance
-- 5. Snowflake uses AI to classify relevant news
-- 6. Snowflake scores and prioritizes the news
-- 7. Snowflake creates a deduplicated final digest queue
-- 8. Snowflake creates one email-ready row for Alteryx
--
-- Important:
-- This script does NOT send emails from Snowflake.
-- Alteryx will send the email later.
--
-- Slow parts:
-- Step 03 and Step 04 use Cortex AI.
-- If you are only changing filters/scoring, only rerun Step 05-08.
-- ============================================================


-- ============================================================
-- 00 - Context
-- ============================================================

USE ROLE SANDBOX_DEVELOPER;
USE WAREHOUSE SANDBOX_WH;

CREATE DATABASE IF NOT EXISTS PHARMA_NEWS_SANDBOX;
CREATE SCHEMA IF NOT EXISTS PHARMA_NEWS_SANDBOX.NEWS;

USE DATABASE PHARMA_NEWS_SANDBOX;
USE SCHEMA NEWS;


-- ============================================================
-- 00A - Staging table for Alteryx output
-- Safe: IF NOT EXISTS means this does NOT delete existing data
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
-- 00B - Priority keyword reference table
-- Mainly for documentation/transparency
-- ============================================================

CREATE OR REPLACE TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_PRIORITY_KEYWORDS AS
SELECT
    COLUMN1::STRING AS CATEGORY,
    COLUMN2::STRING AS KEYWORD,
    COLUMN3::STRING AS CRITERIA
FROM VALUES
    ('Value', 'value >$1B', 'always'),
    ('Size', 'expansion/acquisition/construction', 'includes Top 25 or CDMO, or >$500M value'),
    ('Size', 'new capability', 'for a known CDMO'),
    ('Size', 'closing/shuttering/divestment', 'entire site or business unit of Top 25 or CDMO'),
    ('Size', 'layoffs', 'includes Top 25 or CDMO, or >200 employees'),
    ('Regulatory', 'most favored nation', 'always'),
    ('Regulatory', 'tariffs', 'includes pharma'),
    ('Regulatory', 'FDA', 'leadership changes, or includes biopharma or biologics regulations'),
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
    ('Companies', 'CDMO companies', 'always');


-- ============================================================
-- 00C - Tracked companies
-- ============================================================

CREATE OR REPLACE TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_TRACKED_COMPANIES AS
SELECT
    COLUMN1::STRING AS COMPANY_NAME,
    COLUMN2::STRING AS COMPANY_CATEGORY
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
    ('Thermo Fisher Scientific', 'CDMO'),
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
    ('Rentschler Biopharma', 'CDMO');


-- ============================================================
-- 01 - Clean article view
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
-- 02 - Final gate
-- Removes obvious newsletter noise before AI
-- ============================================================

CREATE OR REPLACE TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_SUBJECT_GATE_FINAL AS
SELECT
    *,
    CASE
        WHEN ARTICLE_TITLE_LC IS NULL THEN 'DROP'

        WHEN ARTICLE_TITLE_LC LIKE 'a message from %' THEN 'DROP'
        WHEN ARTICLE_TITLE_LC LIKE 'brought to you by %' THEN 'DROP'
        WHEN ARTICLE_TITLE_LC LIKE 'sponsored by %' THEN 'DROP'
        WHEN ARTICLE_TITLE_LC LIKE '%sponsored by%' THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '^(editor-in-chief|senior editor|senior writer|executive editor|associate editor|deputy editor|staff writer|staff writers|publisher|contributing writer|sales director)'
            THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '(unsubscribe|privacy policy|contact support|linkedin logo|facebook logo|twitter logo|youtube logo|brand logo|questex signature)'
            THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '(webinar|podcast|top line podcast|conference|exhibition|whitepaper|download the white paper|register now|register today|save your spot|innovation week|pharma ci|fierce biotech week|outsourcing awards)'
            THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '(de-risk your program|real-world evidence|action gap|shaping what.s next|deliver confidence|move to market with confidence|explore our services|reliable delivery|global regulatory expertise|scalable gmp manufacturing|meet .* at asco|fierce ai innovation award|biopharma sentiment index|partnerships with sites)'
            THEN 'DROP'

        WHEN ARTICLE_TITLE_LC LIKE '[%' THEN 'DROP'
        WHEN ARTICLE_TITLE_LC LIKE 'http%' THEN 'DROP'
        WHEN ARTICLE_TITLE_LC LIKE 'click here%' THEN 'DROP'
        WHEN LENGTH(ARTICLE_TITLE_CLEAN) < 20 THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '(acquisition|buyout|merger|deal|licensing|collaboration|partnership|supply deal|supply agreement|fda|approval|approved|manufacturing|facility|expansion|phase 3|ipo|fundraising|investment|capacity|tariffs|regulation|regulatory|supreme court|most favored nation|shortage|shortages|layoffs|divestment|closing|shuttering|clinical hold|commercial|contract|launch)'
            THEN 'PASS'

        ELSE 'REVIEW'
    END AS SUBJECT_GATE_FINAL

FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_ARTICLES_CLEAN;


-- ============================================================
-- 03 - Relevance
-- AI evaluates only non-drop rows
-- This is slow
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
            COALESCE(LEFT(ARTICLE_LLM_INPUT, 3000), '')
        )
    ) AS IS_RELEVANT

FROM PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_SUBJECT_GATE_FINAL
WHERE SUBJECT_GATE_FINAL IN ('PASS', 'REVIEW');


-- ============================================================
-- 04 - Classification
-- AI classifies relevant rows into business categories
-- This is slow
-- ============================================================

CREATE OR REPLACE TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_CLASSIFIED AS
SELECT
    *,
    AI_CLASSIFY(
        CONCAT(
            'Subject: ', COALESCE(SUBJECT_RAW, ''), '. ',
            'Title: ', COALESCE(ARTICLE_TITLE_CLEAN, ''), '. ',
            'URL: ', COALESCE(ARTICLE_URL, ''), '. ',
            'Body: ', COALESCE(LEFT(ARTICLE_LLM_INPUT, 3000), '')
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
-- 05 - Priority tier
-- Clean scoring logic
--
-- Important:
-- A row cannot become IMPORTANT just because a company is mentioned.
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
            WHEN TITLE_CONTEXT_LC LIKE 'a message from %' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE 'brought to you by %' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE 'sponsored by %' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%sponsored by%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%podcast%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%webinar%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%conference%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%whitepaper%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%register now%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%register today%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%unsubscribe%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%click here%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%read in browser%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%de-risk your program%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%real-world evidence%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%action gap%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%deliver confidence%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%move to market with confidence%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%explore our services%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%fierce ai innovation award%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%partnerships with sites%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE '%biopharma sentiment index%' THEN TRUE
            WHEN TITLE_CONTEXT_LC LIKE 'the company announced%' THEN TRUE
            WHEN TITLE_CONTEXT_LC RLIKE
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
                '(expansion|construction|facility|site|manufacturing|capacity|capex|investment|new plant|new site)'
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
                '(fda|approval|approved|regulatory|regulation|tariff|tariffs|supreme court|drug shortages|shortages|review|phase 3|clinical hold)'
            THEN TRUE ELSE FALSE
        END AS IS_POLICY_OR_REGULATORY_SIGNAL,

        CASE
            WHEN TITLE_CONTEXT_LC RLIKE
                '(coverage|launch|sales outlook|commercial|market access|reimbursement|customer|contract|award|supply)'
            THEN TRUE ELSE FALSE
        END AS IS_COMMERCIAL_SIGNAL,

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

        -- Company score only counts if there is a real story signal
        + IFF(MATCHED_COMPANY_CATEGORIES ILIKE '%CDMO%' AND HAS_STORY_SIGNAL, 3, 0)
        + IFF(MATCHED_COMPANY_CATEGORIES ILIKE '%Top 25%' AND HAS_STORY_SIGNAL, 1, 0)

        -- AI category only counts if there is a real story signal
        + IFF(HAS_RELEVANT_AI_CATEGORY AND HAS_STORY_SIGNAL, 1, 0)

        -- PASS only counts if there is a real story signal
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
            IFF(MATCHED_COMPANY_CATEGORIES ILIKE '%CDMO%' AND HAS_STORY_SIGNAL, 'Tracked CDMO mentioned in title; ', '') ||
            IFF(MATCHED_COMPANY_CATEGORIES ILIKE '%Top 25%' AND HAS_STORY_SIGNAL, 'Top 25 pharma company mentioned in title; ', '') ||
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
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%fierce ai innovation award%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%biopharma sentiment index%'
      AND LOWER(COALESCE(ARTICLE_TITLE_CLEAN, '')) NOT LIKE '%partnerships with sites%'

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

    -- For testing/demo: last 30 days
    -- For daily production, change -30 to -1
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

-- Promo check: this should return 0 rows
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
