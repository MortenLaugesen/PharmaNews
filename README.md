-- ============================================================
-- PHARMA NEWS MONITORING MVP - SIMPLIFIED PRODUCTION SCRIPT
-- ============================================================

-- Recommended daily architecture:
-- Alteryx -> Snowflake staging -> triage/classification -> action queue -> email digest


-- ============================================================
-- 01 - Clean Article View
-- ============================================================

CREATE OR REPLACE VIEW BI.NEWS.V_PHARMA_NEWS_ARTICLES_CLEAN AS
SELECT
    "message_id" AS MESSAGE_ID,
    "sender_name" AS SENDER_NAME,
    "sender_email" AS SENDER_EMAIL,
    "subject_raw" AS SUBJECT_RAW,
    "received_ts" AS RECEIVED_TS,
    TRY_TO_TIMESTAMP_TZ("received_ts") AS RECEIVED_TS_PARSED,
    "email_source_type" AS EMAIL_SOURCE_TYPE,
    "article_rank" AS ARTICLE_RANK,

    TRIM(
        REGEXP_REPLACE(
            REGEXP_REPLACE(
                REGEXP_REPLACE("article_title", '^[0-9]+\\.?\\s*', ''),
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
                    REGEXP_REPLACE("article_title", '^[0-9]+\\.?\\s*', ''),
                    '^,\\s*',
                    ''
                ),
                '\\s+',
                ' '
            )
        )
    ) AS ARTICLE_TITLE_LC,

    "article_url" AS ARTICLE_URL,
    "article_url_extraction_method" AS ARTICLE_URL_EXTRACTION_METHOD,
    "body_best" AS BODY_BEST,
    "article_llm_input" AS ARTICLE_LLM_INPUT,
    "parser_version" AS PARSER_VERSION
FROM BI.NEWS.STG_PHARMA_NEWS_ARTICLES_V1;


-- ============================================================
-- 02 - Final Gate
-- Removes obvious noise and separates PASS vs REVIEW
-- ============================================================

CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_SUBJECT_GATE_FINAL AS
SELECT
    *,
    CASE
        WHEN ARTICLE_TITLE_LC RLIKE
            '^(a message from|brought to you by|by |staff writer|staff writers|senior editor|senior editors|associate editor|executive editor|deputy editor|editor-in-chief|publisher|contributing writer)'
            THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '(unsubscribe|privacy policy|contact support|linkedin logo|facebook logo|twitter logo|youtube logo|brand logo|questex signature)'
            THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '(webinar|podcast|event|whitepaper|download the white paper|register now|register today|save your spot|conference|exhibition|innovation week|pharma ci|fierce biotech week|outsourcing awards)'
            THEN 'DROP'

        WHEN ARTICLE_TITLE_LC LIKE '[%' THEN 'DROP'
        WHEN ARTICLE_TITLE_LC LIKE 'http%' THEN 'DROP'
        WHEN ARTICLE_TITLE_LC LIKE 'click here%' THEN 'DROP'
        WHEN LENGTH(ARTICLE_TITLE_CLEAN) < 20 THEN 'DROP'

        WHEN ARTICLE_TITLE_LC RLIKE
            '(acquisition|buyout|deal|fda|approval|manufacturing|facility|expansion|phase 3|partnership|ipo|fundraising|investment)'
            THEN 'PASS'

        ELSE 'REVIEW'
    END AS SUBJECT_GATE_FINAL
FROM BI.NEWS.V_PHARMA_NEWS_ARTICLES_CLEAN;


-- ============================================================
-- 03 - Relevance
-- AI evaluates whether each non-drop article is relevant
-- ============================================================

CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_RELEVANCE AS
SELECT
    *,
    AI_FILTER(
        PROMPT(
            'Return TRUE if this pharma news item is relevant for a Business Intelligence & Insights team at a biologics/CDMO company. Relevant examples include competitor investments, manufacturing, site expansions, partnerships, acquisitions, customer/commercial signals, financing, IPOs, regulatory milestones, clinical milestones with business impact, supply chain changes, platform/capability updates, and strategic market signals. Not relevant examples include webinars, podcasts, events, whitepapers, sponsor messages, logos, footer links, unsubscribe links, admin content, editorial staff listings, and generic promotions. Subject: {0}. Title: {1}. URL: {2}. Body: {3}',
            SUBJECT_RAW,
            ARTICLE_TITLE_CLEAN,
            ARTICLE_URL,
            BODY_BEST
        )
    ) AS IS_RELEVANT
FROM BI.NEWS.PHARMA_NEWS_SUBJECT_GATE_FINAL
WHERE SUBJECT_GATE_FINAL IN ('PASS', 'REVIEW');


-- ============================================================
-- 04 - Classification
-- AI classifies relevant articles into business categories
-- ============================================================

CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_CLASSIFIED AS
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
            {'label': 'policy_market_signal', 'description': 'trade, tariffs, policy, FDA framework, reshoring, macro signal affecting pharma manufacturing or the market'},
            {'label': 'commercial_customer_signal', 'description': 'customer win, launch, commercial supply, demand signal, backlog, manufacturing award'},
            {'label': 'financing_market_signal', 'description': 'IPO, fundraising, financing, public offering, major market signal'}
        ],
        {
            'task_description': 'Classify relevant pharma-news items for a business intelligence team',
            'output_mode': 'multi'
        }
    ) AS CATEGORY_RESULT
FROM BI.NEWS.PHARMA_NEWS_RELEVANCE
WHERE IS_RELEVANT = TRUE;


-- ============================================================
-- 05 - Priority Bucket
-- Deterministic priority is more stable than AI scoring
-- ============================================================

CREATE OR REPLACE VIEW BI.NEWS.V_PHARMA_NEWS_PRIORITY_BUCKET AS
SELECT
    MESSAGE_ID,
    SENDER_NAME,
    SENDER_EMAIL,
    SUBJECT_RAW,
    RECEIVED_TS,
    RECEIVED_TS_PARSED,
    EMAIL_SOURCE_TYPE,
    ARTICLE_RANK,
    ARTICLE_TITLE_CLEAN AS ARTICLE_TITLE,
    ARTICLE_URL,
    ARTICLE_URL_EXTRACTION_METHOD,
    CATEGORY_RESULT,
    BODY_BEST,
    PARSER_VERSION,

    CASE
        WHEN SUBJECT_GATE_FINAL = 'PASS'
         AND (
              ARTICLE_TITLE_LC RLIKE
              '(acquisition|buyout|approval|phase 3|fda|investment|facility|manufacturing|expansion|ipo|fundraising|partnership|deal)'
              OR TO_JSON(CATEGORY_RESULT) ILIKE '%partnership_ma%'
              OR TO_JSON(CATEGORY_RESULT) ILIKE '%competitor_investment_capacity%'
              OR TO_JSON(CATEGORY_RESULT) ILIKE '%clinical_regulatory_signal%'
              OR TO_JSON(CATEGORY_RESULT) ILIKE '%financing_market_signal%'
         )
        THEN 'HIGH'

        WHEN SUBJECT_GATE_FINAL = 'PASS'
        THEN 'MEDIUM'

        WHEN SUBJECT_GATE_FINAL = 'REVIEW'
        THEN 'MONITOR'

        ELSE 'DROP'
    END AS PRIORITY_BUCKET
FROM BI.NEWS.PHARMA_NEWS_CLASSIFIED;


-- ============================================================
-- 06 - AI Explanation For HIGH Priority Only
-- Adds why_it_matters and recommended_action
-- ============================================================

CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_HIGH_PRIORITY_EXPLAINED AS
SELECT
    *,
    AI_COMPLETE(
        model => 'llama3.3-70b',
        prompt => CONCAT(
            'You are helping a Business Intelligence & Insights team at a biologics/CDMO company. ',
            'Explain briefly why this news matters and suggest a recommended action. ',
            'Return structured output only. ',
            'Title: ', COALESCE(ARTICLE_TITLE, ''), '. ',
            'URL: ', COALESCE(ARTICLE_URL, ''), '. ',
            'Categories: ', COALESCE(TO_JSON(CATEGORY_RESULT), ''), '. ',
            'Body: ', COALESCE(LEFT(BODY_BEST, 1500), '')
        ),
        response_format => TYPE OBJECT(
            why_it_matters STRING,
            recommended_action STRING
        )
    ) AS AI_EXPLANATION
FROM BI.NEWS.V_PHARMA_NEWS_PRIORITY_BUCKET
WHERE PRIORITY_BUCKET = 'HIGH';


-- ============================================================
-- 07 - Final Clean Action Queue
-- This is the operational output view
-- ============================================================

CREATE OR REPLACE VIEW BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN AS
SELECT
    RECEIVED_TS_PARSED,
    EMAIL_SOURCE_TYPE,
    ARTICLE_TITLE,
    ARTICLE_URL,
    PRIORITY_BUCKET,
    WHY_IT_MATTERS,
    RECOMMENDED_ACTION
FROM (
    SELECT
        RECEIVED_TS_PARSED,
        EMAIL_SOURCE_TYPE,
        ARTICLE_TITLE,
        ARTICLE_URL,
        PRIORITY_BUCKET,
        AI_EXPLANATION:why_it_matters::STRING AS WHY_IT_MATTERS,
        AI_EXPLANATION:recommended_action::STRING AS RECOMMENDED_ACTION,

        ROW_NUMBER() OVER (
            PARTITION BY
                LOWER(
                    REGEXP_REPLACE(
                        REGEXP_REPLACE(ARTICLE_TITLE, '[^a-zA-Z0-9 ]', ''),
                        '\\s+',
                        ' '
                    )
                )
            ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
        ) AS RN
    FROM BI.NEWS.PHARMA_NEWS_HIGH_PRIORITY_EXPLAINED
    WHERE LENGTH(ARTICLE_TITLE) >= 20
      AND LOWER(ARTICLE_TITLE) NOT LIKE '%read in browser%'
      AND LOWER(ARTICLE_TITLE) NOT LIKE 'click here%'
      AND LOWER(ARTICLE_TITLE) NOT LIKE 'brought to you by %'
      AND LOWER(ARTICLE_TITLE) NOT LIKE 'staff writer:%'
      AND LOWER(ARTICLE_TITLE) NOT LIKE 'senior editor:%'
      AND LOWER(ARTICLE_TITLE) NOT LIKE 'executive editor:%'
      AND LOWER(ARTICLE_TITLE) NOT LIKE 'publisher:%'
      AND LOWER(ARTICLE_TITLE) NOT LIKE 'the unit for %'
      AND LOWER(ARTICLE_TITLE) NOT LIKE 'plus %'
      AND NOT REGEXP_LIKE(ARTICLE_TITLE, '^[a-z]')
)
WHERE RN = 1;


-- ============================================================
-- 08 - Email Notification Integration
-- Run once. Requires ACCOUNTADMIN or sufficient privileges.
-- Currently sends to DTU email because this recipient is verified.
-- ============================================================

CREATE OR REPLACE NOTIFICATION INTEGRATION PHARMA_NEWS_EMAIL_INT
    TYPE = EMAIL
    ENABLED = TRUE
    ALLOWED_RECIPIENTS = ('s235701@dtu.dk');


-- ============================================================
-- 09 - Stored Procedure: Daily Pharma News Digest
-- Sends top 10 high-priority items from the last 24 hours
-- For demo/testing, change -1 to -7
-- ============================================================

CREATE OR REPLACE PROCEDURE BI.NEWS.SP_SEND_DAILY_PHARMA_NEWS_DIGEST()
RETURNS STRING
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
    V_TOTAL_COUNT NUMBER;
    V_SENT_COUNT NUMBER;
    V_BODY STRING;
BEGIN
    SELECT COUNT(*)
    INTO :V_TOTAL_COUNT
    FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN
    WHERE RECEIVED_TS_PARSED >= DATEADD('day', -1, CURRENT_TIMESTAMP());

    IF (V_TOTAL_COUNT = 0) THEN
        RETURN 'No pharma news items to send.';
    END IF;

    WITH TOP_ITEMS AS (
        SELECT
            RECEIVED_TS_PARSED,
            EMAIL_SOURCE_TYPE,
            ARTICLE_TITLE,
            ARTICLE_URL,
            PRIORITY_BUCKET,
            WHY_IT_MATTERS,
            RECOMMENDED_ACTION
        FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN
        WHERE RECEIVED_TS_PARSED >= DATEADD('day', -1, CURRENT_TIMESTAMP())
        ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
        LIMIT 10
    )
    SELECT COUNT(*)
    INTO :V_SENT_COUNT
    FROM TOP_ITEMS;

    WITH TOP_ITEMS AS (
        SELECT
            RECEIVED_TS_PARSED,
            EMAIL_SOURCE_TYPE,
            ARTICLE_TITLE,
            ARTICLE_URL,
            PRIORITY_BUCKET,
            WHY_IT_MATTERS,
            RECOMMENDED_ACTION
        FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN
        WHERE RECEIVED_TS_PARSED >= DATEADD('day', -1, CURRENT_TIMESTAMP())
        ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
        LIMIT 10
    )
    SELECT
        'Daily Pharma News Digest' || CHR(10) ||
        'Generated from Snowflake action queue' || CHR(10) ||
        'High-priority items found in last 24 hours: ' || :V_TOTAL_COUNT || CHR(10) ||
        'Showing top items: ' || :V_SENT_COUNT || CHR(10) ||
        CHR(10) ||
        LISTAGG(
            ARTICLE_TITLE || CHR(10) ||
            'Source: ' || COALESCE(EMAIL_SOURCE_TYPE, '') || CHR(10) ||
            'Priority: ' || COALESCE(PRIORITY_BUCKET, '') || CHR(10) ||
            'Why it matters: ' || COALESCE(WHY_IT_MATTERS, '') || CHR(10) ||
            'Recommended action: ' || COALESCE(RECOMMENDED_ACTION, '') || CHR(10) ||
            'URL: ' || COALESCE(ARTICLE_URL, '') || CHR(10) ||
            CHR(10) || '------------------------' || CHR(10),
            ''
        ) WITHIN GROUP (ORDER BY RECEIVED_TS_PARSED DESC)
    INTO :V_BODY
    FROM TOP_ITEMS;

    CALL SYSTEM$SEND_EMAIL(
        'PHARMA_NEWS_EMAIL_INT',
        's235701@dtu.dk',
        'Daily Pharma News Digest',
        :V_BODY
    );

    RETURN 'Daily pharma news digest sent. Total found: ' || V_TOTAL_COUNT || ', sent: ' || V_SENT_COUNT;
END;
$$;


-- ============================================================
-- 10 - Daily Email Task
-- Runs every weekday at 08:15 Europe/Copenhagen
-- Only activate when ready
-- ============================================================

CREATE OR REPLACE TASK BI.NEWS.TASK_SEND_DAILY_PHARMA_NEWS_DIGEST
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = 'USING CRON 15 8 * * MON-FRI Europe/Copenhagen'
AS
    CALL BI.NEWS.SP_SEND_DAILY_PHARMA_NEWS_DIGEST();


-- ============================================================
-- OPTIONAL MANUAL TESTS
-- Run these manually when needed
-- ============================================================

-- Test the email integration:
-- CALL SYSTEM$SEND_EMAIL(
--     'PHARMA_NEWS_EMAIL_INT',
--     's235701@dtu.dk',
--     'Test: Pharma News Snowflake Email',
--     'This is a test email from Snowflake for the Pharma News Monitoring MVP.'
-- );

-- Test the digest procedure manually:
-- CALL BI.NEWS.SP_SEND_DAILY_PHARMA_NEWS_DIGEST();

-- Activate the daily task:
-- ALTER TASK BI.NEWS.TASK_SEND_DAILY_PHARMA_NEWS_DIGEST RESUME;

-- Pause the daily task:
-- ALTER TASK BI.NEWS.TASK_SEND_DAILY_PHARMA_NEWS_DIGEST SUSPEND;

-- Execute the task once manually:
-- EXECUTE TASK BI.NEWS.TASK_SEND_DAILY_PHARMA_NEWS_DIGEST;


-- ============================================================
-- OPTIONAL DEBUG CHECKS
-- ============================================================

-- Check final action queue:
-- SELECT *
-- FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN
-- ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
-- LIMIT 100;

-- Check priority bucket distribution:
-- SELECT
--     PRIORITY_BUCKET,
--     COUNT(*) AS CNT
-- FROM BI.NEWS.V_PHARMA_NEWS_PRIORITY_BUCKET
-- GROUP BY PRIORITY_BUCKET
-- ORDER BY CNT DESC;

-- Check how many items would be sent in last 24 hours:
-- SELECT COUNT(*) AS CNT
-- FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN
-- WHERE RECEIVED_TS_PARSED >= DATEADD('day', -1, CURRENT_TIMESTAMP());

-- For demo/testing with last 7 days:
-- SELECT COUNT(*) AS CNT
-- FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN
-- WHERE RECEIVED_TS_PARSED >= DATEADD('day', -7, CURRENT_TIMESTAMP());
