<img width="418" height="245" alt="Skærmbillede 2026-06-09 kl  08 54 40" src="https://github.com/user-attachments/assets/4c5ee243-d1bb-49af-873d-01be971eb1f8" />
add publish date

The one sentence summary is basically just restating the headline, so we should either delete it entirely or expand it to be longer. Right now it's a great list of key news articles, but ideally it'll be able to summarize the articles for us and all we'll have to do is a quick fact check. When I have Genki-Bot summarize the articles I copy paste into it, I usually tell it to summarize the article in 1-3 paragraphs

I've just started using the wording "Summarize this article in 1-3 paragraphs. The summary should focus on specifics related to monetary value, capability expansion or reduction, and/or how the change is expected to shape the industry." I'm not sure if that's the right wording yet, but it's a good starting spot. I'm also not sure if that's helpful to you, but I thought I would share


-- ============================================================
-- PHARMA NEWS MONITORING - CLEAN V1.3
-- Database: PHARMA_NEWS_SANDBOX
-- Schema: NEWS
-- Warehouse: SANDBOX_WH
--
-- WHAT THIS VERSION INCLUDES
-- 1. Alteryx staging table
-- 2. Clean article view
-- 3. Subject gate
-- 4. AI relevance
-- 5. AI classification
-- 6. Priority scoring
-- 7. Final digest queue
-- 8. AI summaries
-- 9. AI semantic deduplication / story keys
-- 10. Professional HTML email view
-- 11. Send email procedure
--
-- IMPORTANT DAILY FLOW
-- After Alteryx has loaded new data:
-- 1. Run Step 02
-- 2. Run Step 03
-- 3. Run Step 04
-- 4. Run CALL SP_REFRESH_AI_SUMMARIES();
-- 5. Run CALL SP_REFRESH_AI_STORY_KEYS();
-- 6. Preview email
-- 7. Send email manually
--
-- Slow parts:
-- Step 03 and Step 04 use Cortex AI.
-- AI summaries and AI story keys also use AI, but only for rows not already stored.
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

ALTER SESSION SET ROWS_PER_RESULTSET = 0;


-- ============================================================
-- 01 - One-time setup / reference objects
-- Run once, or when reference lists change
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
-- 02 - Clean article view and subject gate
-- Run after Alteryx has loaded new data
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
-- 03 - AI relevance
-- Slow step
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
-- 04 - AI classification
-- Slow step
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
-- 05 - Priority scoring view
-- Fast view
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2 AS
WITH BASE AS (
    SELECT
        C.*,

        LOWER(
            TRIM(
                REGEXP_REPLACE(
                    REPLACE(COALESCE(C.ARTICLE_TITLE_CLEAN, ''), CHR(173), ''),
                    '\\s+',
                    ' '
                )
            )
        ) AS TITLE_CONTEXT_LC,

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
            '((\\$|€|£)\\s?[0-9]+(\\.[0-9]+)?\\s?(b|bn|m)|\\b[0-9]+(\\.[0-9]+)?(b|bn|m)\\b|\\b[0-9]+(\\.[0-9]+)?\\s?(billion|million)\\b)'
        ) AS DEAL_VALUE_KEY,

        CASE
            WHEN REGEXP_LIKE(
                B.TITLE_CONTEXT_LC,
                '.*((\\$|€|£)\\s?[0-9]+(\\.[0-9]+)?\\s?(b|bn)|\\b[0-9]+(\\.[0-9]+)?(b|bn)\\b|\\b[0-9]+(\\.[0-9]+)?\\s?billion\\b).*'
            )
            THEN TRUE ELSE FALSE
        END AS VALUE_ABOVE_1B,

        CASE
            WHEN REGEXP_LIKE(
                B.TITLE_CONTEXT_LC,
                '.*((\\$|€|£)\\s?[0-9]+(\\.[0-9]+)?\\s?(b|bn)|\\b[0-9]+(\\.[0-9]+)?(b|bn)\\b|\\b[0-9]+(\\.[0-9]+)?\\s?billion\\b).*'
            )
            OR REGEXP_LIKE(
                B.TITLE_CONTEXT_LC,
                '.*((\\$|€|£)\\s?[5-9][0-9]{2,}\\s?m|\\b[5-9][0-9]{2,}m\\b|\\b[5-9][0-9]{2,}\\s?million\\b).*'
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
            THEN TRUE ELSE FALSE
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


-- ============================================================
-- 06 - Final digest queue
-- Fast view
-- SQL-level cleanup and dedupe before AI summaries/story-key dedupe
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2 AS
WITH SOURCE_ROWS AS (
    SELECT
        RECEIVED_TS_PARSED,
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
        CATEGORY_RESULT,
        DEAL_VALUE_KEY

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_PRIORITY_TIER_V2
    WHERE PRIORITY_TIER IN ('VERY_IMPORTANT', 'IMPORTANT')
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
        ) AS TITLE_ROOT_NO_VALUE

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

      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE 'this is %'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE 'henderson now has %'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE 'the autoimmune disease drug developer%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE 'in february, it announced%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE 'the fda has extended%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE 'abbvie said the fda has approved%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%obtained as a part of its $10.1 billion%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%shareholders:%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%has an answer for%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%spotlighted this year%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%the second contract%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%read more%'
      AND LOWER(COALESCE(ARTICLE_TITLE, '')) NOT LIKE '%continue reading%'

      AND NOT REGEXP_LIKE(ARTICLE_TITLE, '^[^A-Za-z0-9].*')
      AND LENGTH(ARTICLE_TITLE) BETWEEN 20 AND 170
),

STORY_KEYS AS (
    SELECT
        *,
        CASE
            WHEN DEAL_VALUE_KEY IS NOT NULL
             AND COALESCE(NULLIF(MATCHED_COMPANIES, ''), '') <> ''
            THEN COALESCE(NULLIF(MATCHED_COMPANIES, ''), 'NO_COMPANY') || '|' || LOWER(DEAL_VALUE_KEY)

            WHEN DEAL_VALUE_KEY IS NOT NULL
            THEN LOWER(DEAL_VALUE_KEY) || '|' || LEFT(TITLE_ROOT_NO_VALUE, 35)

            WHEN COALESCE(NULLIF(MATCHED_COMPANIES, ''), '') <> ''
            THEN COALESCE(NULLIF(MATCHED_COMPANIES, ''), 'NO_COMPANY') || '|' || LEFT(TITLE_ROOT_NO_VALUE, 38)

            ELSE 'NO_COMPANY|' || LEFT(TITLE_ROOT_NO_VALUE, 42)
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
                    WHEN LENGTH(ARTICLE_TITLE) BETWEEN 45 AND 140 THEN 1
                    WHEN LENGTH(ARTICLE_TITLE) BETWEEN 30 AND 160 THEN 2
                    ELSE 3
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


-- ============================================================
-- 06A - AI summaries
-- Adds AI-generated summaries for VERY_IMPORTANT items
-- Run after Step 06 exists.
-- ============================================================

CREATE TABLE IF NOT EXISTS PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES (
    SUMMARY_KEY STRING,
    ARTICLE_TITLE STRING,
    ARTICLE_URL STRING,
    EMAIL_SOURCE_TYPE STRING,
    AI_SUMMARY STRING,
    CREATED_AT TIMESTAMP_TZ DEFAULT CURRENT_TIMESTAMP()
);


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

        D.ARTICLE_TITLE,
        D.ARTICLE_URL,
        D.EMAIL_SOURCE_TYPE,
        D.PRIORITY_TIER,
        D.SIGNAL_SCORE,
        D.SIGNAL_REASONS,
        D.MATCHED_COMPANIES,
        D.MATCHED_COMPANY_CATEGORIES

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2 D

    LEFT JOIN PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES S
        ON SHA2(
            LOWER(
                COALESCE(D.ARTICLE_URL, '') || '|' ||
                COALESCE(D.ARTICLE_TITLE, '')
            ),
            256
        ) = S.SUMMARY_KEY

    WHERE D.PRIORITY_TIER = 'VERY_IMPORTANT'
      AND D.RECEIVED_TS_PARSED >= DATEADD('day', -7, CURRENT_TIMESTAMP())
      AND S.SUMMARY_KEY IS NULL
      AND D.ARTICLE_TITLE IS NOT NULL;


    SELECT COUNT(*)
    INTO :V_NEW_SUMMARIES
    FROM TMP_AI_SUMMARY_CANDIDATES;


    INSERT INTO PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES (
        SUMMARY_KEY,
        ARTICLE_TITLE,
        ARTICLE_URL,
        EMAIL_SOURCE_TYPE,
        AI_SUMMARY,
        CREATED_AT
    )
    SELECT
        SUMMARY_KEY,
        ARTICLE_TITLE,
        ARTICLE_URL,
        EMAIL_SOURCE_TYPE,

        AI_COMPLETE(
            'llama3.3-70b',
            CONCAT(
                'Write one concise executive summary for this pharma news item. ',
                'Audience: Business Intelligence & Insights team at a biologics/CDMO company. ',
                'Explain why the story matters strategically in max 28 words. ',
                'Focus on pipeline, competitor movement, market positioning, regulatory impact, financing, investment, or portfolio strategy. ',
                'Only mention manufacturing, capacity, CDMO, GMP, supply, or facilities if those topics are explicitly present in the headline or signal reasons. ',
                'Do not invent manufacturing or CDMO relevance. ',
                'Do not say "manufacturing capabilities" unless manufacturing/capacity/facility/site/plant/CDMO/GMP/supply is explicitly mentioned. ',
                'Do not repeat the full headline. ',
                'No hype. No bullet points. No quotation marks. ',
                'Headline: ', COALESCE(ARTICLE_TITLE, ''), '. ',
                'Source: ', COALESCE(EMAIL_SOURCE_TYPE, ''), '. ',
                'Matched companies: ', COALESCE(MATCHED_COMPANIES, ''), '. ',
                'Company categories: ', COALESCE(MATCHED_COMPANY_CATEGORIES, ''), '. ',
                'Signal reasons: ', COALESCE(SIGNAL_REASONS, '')
            )
        ) AS AI_SUMMARY,

        CURRENT_TIMESTAMP() AS CREATED_AT

    FROM TMP_AI_SUMMARY_CANDIDATES;


    RETURN 'AI summaries refreshed. New summaries created: ' || V_NEW_SUMMARIES;

END;
$$;


-- ============================================================
-- 06B - AI semantic deduplication / story keys
-- Creates semantic story keys so similar news stories are grouped together.
-- Run after Step 06 exists.
-- ============================================================

CREATE TABLE IF NOT EXISTS PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_STORY_KEYS (
    STORY_ITEM_KEY STRING,
    ARTICLE_TITLE STRING,
    ARTICLE_URL STRING,
    EMAIL_SOURCE_TYPE STRING,
    PRIORITY_TIER STRING,
    SIGNAL_SCORE NUMBER,
    MATCHED_COMPANIES STRING,
    MATCHED_COMPANY_CATEGORIES STRING,
    AI_STORY_KEY STRING,
    CREATED_AT TIMESTAMP_TZ DEFAULT CURRENT_TIMESTAMP()
);


CREATE OR REPLACE PROCEDURE PHARMA_NEWS_SANDBOX.NEWS.SP_REFRESH_AI_STORY_KEYS()
RETURNS STRING
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
    V_NEW_KEYS NUMBER DEFAULT 0;
BEGIN

    CREATE OR REPLACE TEMP TABLE TMP_AI_STORY_KEY_CANDIDATES AS
    SELECT
        SHA2(
            LOWER(
                COALESCE(D.ARTICLE_URL, '') || '|' ||
                COALESCE(D.ARTICLE_TITLE, '')
            ),
            256
        ) AS STORY_ITEM_KEY,

        D.ARTICLE_TITLE,
        D.ARTICLE_URL,
        D.EMAIL_SOURCE_TYPE,
        D.PRIORITY_TIER,
        D.SIGNAL_SCORE,
        D.SIGNAL_REASONS,
        D.MATCHED_COMPANIES,
        D.MATCHED_COMPANY_CATEGORIES

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2 D

    LEFT JOIN PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_STORY_KEYS K
        ON SHA2(
            LOWER(
                COALESCE(D.ARTICLE_URL, '') || '|' ||
                COALESCE(D.ARTICLE_TITLE, '')
            ),
            256
        ) = K.STORY_ITEM_KEY

    WHERE D.RECEIVED_TS_PARSED >= DATEADD('day', -7, CURRENT_TIMESTAMP())
      AND D.PRIORITY_TIER IN ('VERY_IMPORTANT', 'IMPORTANT')
      AND K.STORY_ITEM_KEY IS NULL
      AND D.ARTICLE_TITLE IS NOT NULL;


    SELECT COUNT(*)
    INTO :V_NEW_KEYS
    FROM TMP_AI_STORY_KEY_CANDIDATES;


    INSERT INTO PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_STORY_KEYS (
        STORY_ITEM_KEY,
        ARTICLE_TITLE,
        ARTICLE_URL,
        EMAIL_SOURCE_TYPE,
        PRIORITY_TIER,
        SIGNAL_SCORE,
        MATCHED_COMPANIES,
        MATCHED_COMPANY_CATEGORIES,
        AI_STORY_KEY,
        CREATED_AT
    )
    SELECT
        STORY_ITEM_KEY,
        ARTICLE_TITLE,
        ARTICLE_URL,
        EMAIL_SOURCE_TYPE,
        PRIORITY_TIER,
        SIGNAL_SCORE,
        MATCHED_COMPANIES,
        MATCHED_COMPANY_CATEGORIES,

        REGEXP_REPLACE(
            REGEXP_REPLACE(
                LOWER(
                    AI_COMPLETE(
                        'llama3.3-70b',
                        CONCAT(
                            'Create one stable semantic deduplication key for this pharma news item. ',
                            'Two headlines should get the same key if they describe the same underlying story, even if wording, source, or deal value differs. ',
                            'Use lowercase snake_case only. ',
                            'Return only the key. No explanation. No quotes. ',
                            'Ignore minor differences in deal value if the company, partner/target, asset/disease area and event are the same. ',
                            'Examples: ',
                            'Servier signs $1.55B upfront deal for Edgewise muscular dystrophy assets -> servier_edgewise_muscular_dystrophy_buyout. ',
                            'Servier inks $2.6B buyout of Edgewise muscular dystrophy unit -> servier_edgewise_muscular_dystrophy_buyout. ',
                            'Pfizer strikes Innovent deal worth up to $10.5B for cancer programs -> pfizer_innovent_oncology_program_deal. ',
                            'Pfizer pens $10B 12-drug deal with Innovent -> pfizer_innovent_oncology_program_deal. ',
                            'ASCO Celcuity dares to dream with breast cancer data -> celcuity_breast_cancer_phase3_data. ',
                            'Headline: ', COALESCE(ARTICLE_TITLE, ''), '. ',
                            'Source: ', COALESCE(EMAIL_SOURCE_TYPE, ''), '. ',
                            'Matched companies: ', COALESCE(MATCHED_COMPANIES, ''), '. ',
                            'Company categories: ', COALESCE(MATCHED_COMPANY_CATEGORIES, ''), '. ',
                            'Signal reasons: ', COALESCE(SIGNAL_REASONS, '')
                        )
                    )
                ),
                '[^a-z0-9]+',
                '_'
            ),
            '^_+|_+$',
            ''
        ) AS AI_STORY_KEY,

        CURRENT_TIMESTAMP() AS CREATED_AT

    FROM TMP_AI_STORY_KEY_CANDIDATES;


    RETURN 'AI story keys refreshed. New story keys created: ' || V_NEW_KEYS;

END;
$$;


-- ============================================================
-- 07 - Professional HTML email output view
-- Uses AI summaries + AI story keys
-- Result: Top 10 unique stories
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_ALTERYX_DAILY_PHARMA_NEWS_EMAIL AS
WITH ITEMS_RAW AS (
    SELECT
        D.RECEIVED_TS_PARSED,
        D.EMAIL_SOURCE_TYPE,
        D.ARTICLE_TITLE,
        D.ARTICLE_URL,
        D.PRIORITY_TIER,
        D.SIGNAL_SCORE,
        D.SIGNAL_REASONS,
        D.MATCHED_COMPANIES,
        D.MATCHED_COMPANY_CATEGORIES,

        S.AI_SUMMARY,
        K.AI_STORY_KEY,

        SHA2(
            LOWER(
                COALESCE(D.ARTICLE_URL, '') || '|' ||
                COALESCE(D.ARTICLE_TITLE, '')
            ),
            256
        ) AS ITEM_KEY,

        CASE
            WHEN D.PRIORITY_TIER = 'VERY_IMPORTANT' THEN 1
            WHEN D.PRIORITY_TIER = 'IMPORTANT' THEN 2
            ELSE 3
        END AS PRIORITY_SORT

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2 D

    LEFT JOIN PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES S
        ON SHA2(
            LOWER(
                COALESCE(D.ARTICLE_URL, '') || '|' ||
                COALESCE(D.ARTICLE_TITLE, '')
            ),
            256
        ) = S.SUMMARY_KEY

    LEFT JOIN PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_STORY_KEYS K
        ON SHA2(
            LOWER(
                COALESCE(D.ARTICLE_URL, '') || '|' ||
                COALESCE(D.ARTICLE_TITLE, '')
            ),
            256
        ) = K.STORY_ITEM_KEY

    WHERE D.RECEIVED_TS_PARSED >= DATEADD('day', -7, CURRENT_TIMESTAMP())
      AND D.PRIORITY_TIER IN ('VERY_IMPORTANT', 'IMPORTANT')
),

ITEMS_WITH_DEDUPE_KEY AS (
    SELECT
        *,
        COALESCE(NULLIF(AI_STORY_KEY, ''), ITEM_KEY) AS FINAL_STORY_KEY
    FROM ITEMS_RAW
),

ITEMS_DEDUPED AS (
    SELECT *
    FROM ITEMS_WITH_DEDUPE_KEY
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY FINAL_STORY_KEY
        ORDER BY
            PRIORITY_SORT,
            SIGNAL_SCORE DESC,
            RECEIVED_TS_PARSED DESC NULLS LAST,

            CASE
                WHEN AI_SUMMARY IS NOT NULL THEN 1
                ELSE 2
            END,

            CASE
                WHEN LENGTH(ARTICLE_TITLE) BETWEEN 45 AND 150 THEN 1
                WHEN LENGTH(ARTICLE_TITLE) BETWEEN 30 AND 170 THEN 2
                ELSE 3
            END,

            LENGTH(ARTICLE_TITLE) DESC
    ) = 1
),

ITEMS AS (
    SELECT
        *,

        REPLACE(REPLACE(REPLACE(COALESCE(ARTICLE_TITLE, ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;') AS ARTICLE_TITLE_HTML,
        REPLACE(REPLACE(REPLACE(COALESCE(EMAIL_SOURCE_TYPE, ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;') AS EMAIL_SOURCE_TYPE_HTML,
        REPLACE(REPLACE(REPLACE(COALESCE(ARTICLE_URL, ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;') AS ARTICLE_URL_HTML,
        REPLACE(REPLACE(REPLACE(COALESCE(AI_SUMMARY, ''), '&', '&amp;'), '<', '&lt;'), '>', '&gt;') AS AI_SUMMARY_HTML,

        CASE
            WHEN COALESCE(NULLIF(MATCHED_COMPANIES, ''), '') <> ''
              OR COALESCE(NULLIF(MATCHED_COMPANY_CATEGORIES, ''), '') <> ''
            THEN
                CASE
                    WHEN COALESCE(NULLIF(MATCHED_COMPANIES, ''), '') <> ''
                    THEN REPLACE(REPLACE(REPLACE(MATCHED_COMPANIES, '&', '&amp;'), '<', '&lt;'), '>', '&gt;')
                    ELSE ''
                END ||
                CASE
                    WHEN COALESCE(NULLIF(MATCHED_COMPANIES, ''), '') <> ''
                     AND COALESCE(NULLIF(MATCHED_COMPANY_CATEGORIES, ''), '') <> ''
                    THEN ' · '
                    ELSE ''
                END ||
                CASE
                    WHEN COALESCE(NULLIF(MATCHED_COMPANY_CATEGORIES, ''), '') <> ''
                    THEN REPLACE(REPLACE(REPLACE(MATCHED_COMPANY_CATEGORIES, '&', '&amp;'), '<', '&lt;'), '>', '&gt;')
                    ELSE ''
                END
            ELSE ''
        END AS TAGS_TEXT_HTML

    FROM ITEMS_DEDUPED
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

RAW_COUNTS AS (
    SELECT
        COUNT(*) AS RAW_TOTAL_ITEMS,
        COUNT_IF(PRIORITY_TIER = 'VERY_IMPORTANT') AS RAW_VERY_IMPORTANT_ITEMS,
        COUNT_IF(PRIORITY_TIER = 'IMPORTANT') AS RAW_IMPORTANT_ITEMS
    FROM ITEMS_RAW
),

DEDUPED_COUNTS AS (
    SELECT
        COUNT(*) AS UNIQUE_STORIES,
        COUNT_IF(PRIORITY_TIER = 'VERY_IMPORTANT') AS UNIQUE_VERY_IMPORTANT_ITEMS,
        COUNT_IF(PRIORITY_TIER = 'IMPORTANT') AS UNIQUE_IMPORTANT_ITEMS
    FROM ITEMS_DEDUPED
),

EMAIL_TEXT AS (
    SELECT
        LISTAGG(
            '<tr>
                <td style="padding:16px 0; border-bottom:1px solid #e5e5e5;">

                    <div style="font-size:12px; font-weight:bold; letter-spacing:0.3px; margin-bottom:6px;">' ||
                        CASE
                            WHEN PRIORITY_TIER = 'VERY_IMPORTANT'
                            THEN '<span style="color:#b00020;">VERY IMPORTANT</span>'
                            ELSE '<span style="color:#555555;">IMPORTANT</span>'
                        END ||
                    '</div>

                    <div style="font-size:16px; font-weight:bold; line-height:1.35; margin-bottom:8px;">
                        <a href="' || ARTICLE_URL_HTML || '" style="color:#1a1a1a; text-decoration:none;">' ||
                            ARTICLE_TITLE_HTML ||
                        '</a>
                    </div>' ||

                    CASE
                        WHEN PRIORITY_TIER = 'VERY_IMPORTANT'
                         AND COALESCE(NULLIF(AI_SUMMARY_HTML, ''), '') <> ''
                        THEN
                            '<div style="font-size:14px; color:#333333; line-height:1.45; margin-bottom:8px;">
                                <b>Summary:</b> ' || AI_SUMMARY_HTML || '
                            </div>'
                        ELSE ''
                    END ||

                    CASE
                        WHEN COALESCE(NULLIF(TAGS_TEXT_HTML, ''), '') <> ''
                        THEN
                            '<div style="font-size:13px; color:#666666; margin-bottom:8px;">
                                <b>Tags:</b> ' || TAGS_TEXT_HTML || ' · ' || EMAIL_SOURCE_TYPE_HTML || '
                            </div>'
                        ELSE
                            '<div style="font-size:13px; color:#666666; margin-bottom:8px;">
                                <b>Source:</b> ' || EMAIL_SOURCE_TYPE_HTML || '
                            </div>'
                    END ||

                    '<div style="font-size:13px;">
                        <a href="' || ARTICLE_URL_HTML || '" style="color:#0b57d0; text-decoration:none;">Read article →</a>
                    </div>

                </td>
            </tr>',
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

    '<html>
        <body style="margin:0; padding:0; background-color:#f6f6f6; font-family:Arial, sans-serif;">
            <table width="100%" cellpadding="0" cellspacing="0" style="background-color:#f6f6f6; padding:24px 0;">
                <tr>
                    <td align="center">
                        <table width="720" cellpadding="0" cellspacing="0" style="background-color:#ffffff; border:1px solid #e5e5e5;">
                            <tr>
                                <td style="padding:24px 28px 14px 28px;">
                                    <div style="font-size:22px; font-weight:bold; color:#1a1a1a; margin-bottom:6px;">
                                        Daily Pharma News Digest
                                    </div>
                                    <div style="font-size:13px; color:#666666; line-height:1.5;">
                                        Generated and sent from Snowflake<br>
                                        Lookback window: last 7 days · Top 10 unique stories
                                    </div>
                                </td>
                            </tr>

                            <tr>
                                <td style="padding:0 28px 18px 28px;">
                                    <table width="100%" cellpadding="0" cellspacing="0" style="background-color:#f3f5f7; border:1px solid #e2e6ea;">
                                        <tr>
                                            <td style="padding:12px 14px; font-size:13px; color:#333333;">
                                                <b>Raw relevant items:</b> ' || COALESCE(R.RAW_TOTAL_ITEMS, 0) || ' &nbsp; | &nbsp;
                                                <b>Unique stories:</b> ' || COALESCE(C.UNIQUE_STORIES, 0) || ' &nbsp; | &nbsp;
                                                <b>Very important:</b> ' || COALESCE(C.UNIQUE_VERY_IMPORTANT_ITEMS, 0) || ' &nbsp; | &nbsp;
                                                <b>Important:</b> ' || COALESCE(C.UNIQUE_IMPORTANT_ITEMS, 0) || '
                                            </td>
                                        </tr>
                                    </table>
                                </td>
                            </tr>

                            <tr>
                                <td style="padding:0 28px 28px 28px;">
                                    <table width="100%" cellpadding="0" cellspacing="0">' ||

                                        COALESCE(E.BODY_ITEMS, '<tr><td style="padding:16px 0;">No important pharma news items found in the selected period.</td></tr>') ||

                                    '</table>
                                </td>
                            </tr>
                        </table>
                    </td>
                </tr>
            </table>
        </body>
    </html>' AS EMAIL_BODY

FROM RAW_COUNTS R
CROSS JOIN DEDUPED_COUNTS C
CROSS JOIN EMAIL_TEXT E;


-- ============================================================
-- 08 - Send email procedure
-- Run once, or after editing the procedure
-- ============================================================

CREATE OR REPLACE PROCEDURE PHARMA_NEWS_SANDBOX.NEWS.SP_SEND_PHARMA_NEWS_EMAIL_FROM_VIEW()
RETURNS STRING
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
    V_TO STRING;
    V_SUBJECT STRING;
    V_BODY STRING;
BEGIN

    SELECT
        TO_EMAIL,
        EMAIL_SUBJECT,
        EMAIL_BODY
    INTO
        :V_TO,
        :V_SUBJECT,
        :V_BODY
    FROM PHARMA_NEWS_SANDBOX.NEWS.V_ALTERYX_DAILY_PHARMA_NEWS_EMAIL
    LIMIT 1;

    CALL SYSTEM$SEND_EMAIL(
        'EMAIL_INT_MORTEN',
        :V_TO,
        :V_SUBJECT,
        :V_BODY,
        'text/html'
    );

    RETURN 'HTML email sent to: ' || V_TO;

END;
$$;


-- ============================================================
-- 09 - Manual daily run block
-- Run this after Alteryx and after Step 02-04 have been refreshed.
-- This does NOT send the email automatically.
-- ============================================================

CALL PHARMA_NEWS_SANDBOX.NEWS.SP_REFRESH_AI_SUMMARIES();

CALL PHARMA_NEWS_SANDBOX.NEWS.SP_REFRESH_AI_STORY_KEYS();


-- Preview email
SELECT
    TO_EMAIL,
    EMAIL_SUBJECT,
    LEFT(EMAIL_BODY, 5000) AS EMAIL_BODY_PREVIEW
FROM PHARMA_NEWS_SANDBOX.NEWS.V_ALTERYX_DAILY_PHARMA_NEWS_EMAIL;


-- Optional checks
SELECT
    COUNT(*) AS STAGING_ROWS
FROM PHARMA_NEWS_SANDBOX.NEWS.STG_PHARMA_NEWS_ARTICLES_V1;

SELECT
    PRIORITY_TIER,
    COUNT(*) AS CNT
FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
GROUP BY PRIORITY_TIER
ORDER BY CNT DESC;

SELECT
    K.AI_STORY_KEY,
    COUNT(*) AS CNT,
    LISTAGG(K.ARTICLE_TITLE, ' | ') WITHIN GROUP (ORDER BY K.ARTICLE_TITLE) AS TITLES
FROM PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_STORY_KEYS K
WHERE K.CREATED_AT >= DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY K.AI_STORY_KEY
ORDER BY CNT DESC, K.AI_STORY_KEY;


-- ============================================================
-- 10 - Send email
-- Run manually ONLY after preview looks good.
-- ============================================================

-- CALL PHARMA_NEWS_SANDBOX.NEWS.SP_SEND_PHARMA_NEWS_EMAIL_FROM_VIEW();

CALL PHARMA_NEWS_SANDBOX.NEWS.SP_REFRESH_AI_SUMMARIES();

CALL PHARMA_NEWS_SANDBOX.NEWS.SP_REFRESH_AI_STORY_KEYS();

SELECT
    TO_EMAIL,
    EMAIL_SUBJECT,
    LEFT(EMAIL_BODY, 5000) AS EMAIL_BODY_PREVIEW
FROM PHARMA_NEWS_SANDBOX.NEWS.V_ALTERYX_DAILY_PHARMA_NEWS_EMAIL;

CALL PHARMA_NEWS_SANDBOX.NEWS.SP_SEND_PHARMA_NEWS_EMAIL_FROM_VIEW();
