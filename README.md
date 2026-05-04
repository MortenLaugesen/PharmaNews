-- 01 - Base View
CREATE OR REPLACE VIEW BI.NEWS.V_PHARMA_NEWS_ARTICLES_BASE AS
SELECT
    "message_id" AS MESSAGE_ID,
    "sender_name" AS SENDER_NAME,
    "sender_email" AS SENDER_EMAIL,
    "subject_raw" AS SUBJECT_RAW,
    "received_ts" AS RECEIVED_TS,
    TRY_TO_TIMESTAMP_TZ("received_ts") AS RECEIVED_TS_PARSED,
    "email_source_type" AS EMAIL_SOURCE_TYPE,
    "article_rank" AS ARTICLE_RANK,
    "article_title" AS ARTICLE_TITLE,
    "article_url" AS ARTICLE_URL,
    "article_url_extraction_method" AS ARTICLE_URL_EXTRACTION_METHOD,
    "body_best" AS BODY_BEST,
    "article_llm_input" AS ARTICLE_LLM_INPUT,
    "parser_version" AS PARSER_VERSION
FROM BI.NEWS.STG_PHARMA_NEWS_ARTICLES_V1;

-- 01A - Check Base View
SELECT
    MESSAGE_ID,
    EMAIL_SOURCE_TYPE,
    ARTICLE_RANK,
    ARTICLE_TITLE,
    ARTICLE_URL,
    ARTICLE_URL_EXTRACTION_METHOD,
    RECEIVED_TS_PARSED
FROM BI.NEWS.V_PHARMA_NEWS_ARTICLES_BASE
LIMIT 50;
-- 02 - Subject and Title Gate
CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_SUBJECT_GATE AS
SELECT
    *,
    CASE
        WHEN LOWER(ARTICLE_TITLE) LIKE 'a message from %' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '[%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%unsubscribe%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%privacy policy%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%contact support%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%linkedin logo%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%facebook logo%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%twitter logo%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%youtube logo%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%brand logo%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%questex signature%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%webinar%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%podcast%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%event%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%whitepaper%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%download the white paper%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%register now%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%register today%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%save your spot%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE) LIKE 'http%' THEN 'DROP'

        WHEN LOWER(ARTICLE_TITLE) LIKE '%acquisition%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%buyout%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%deal%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%fda%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%approval%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%manufacturing%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%facility%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%expansion%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%phase 3%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%partnership%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%ipo%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%fundraising%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE) LIKE '%investment%' THEN 'PASS'

        ELSE 'REVIEW'
    END AS SUBJECT_GATE
FROM BI.NEWS.V_PHARMA_NEWS_ARTICLES_BASE;

-- 02A - Gate Distribution
SELECT
    SUBJECT_GATE,
    COUNT(*) AS CNT
FROM BI.NEWS.PHARMA_NEWS_SUBJECT_GATE
GROUP BY SUBJECT_GATE
ORDER BY CNT DESC;

-- 02B - Preview Non-Drop Rows
SELECT
    SUBJECT_GATE,
    EMAIL_SOURCE_TYPE,
    ARTICLE_RANK,
    ARTICLE_TITLE,
    ARTICLE_URL
FROM BI.NEWS.PHARMA_NEWS_SUBJECT_GATE
WHERE SUBJECT_GATE IN ('PASS', 'REVIEW')
ORDER BY SUBJECT_GATE, ARTICLE_TITLE
LIMIT 100;

-- 03 - Relevance
CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_RELEVANCE AS
SELECT
    *,
    CASE
        WHEN SUBJECT_GATE = 'DROP' THEN FALSE
        ELSE AI_FILTER(
            PROMPT(
                'Return TRUE if this pharma news item is relevant for a Business Intelligence & Insights team at a biologics/CDMO company. Relevant examples include competitor investments, manufacturing, site expansions, partnerships, acquisitions, customer/commercial signals, financing, IPOs, regulatory milestones, clinical milestones with business impact, supply chain changes, platform/capability updates, and strategic market signals. Not relevant examples include webinars, podcasts, events, whitepapers, sponsor messages, logos, footer links, unsubscribe links, admin content, and generic promotions. Subject: {0}. Title: {1}. URL: {2}. Body: {3}',
                SUBJECT_RAW,
                ARTICLE_TITLE,
                ARTICLE_URL,
                BODY_BEST
            )
        )
    END AS IS_RELEVANT
FROM BI.NEWS.PHARMA_NEWS_SUBJECT_GATE
WHERE SUBJECT_GATE IN ('PASS', 'REVIEW');

-- 03A - Relevance Distribution
SELECT
    IS_RELEVANT,
    COUNT(*) AS CNT
FROM BI.NEWS.PHARMA_NEWS_RELEVANCE
GROUP BY IS_RELEVANT
ORDER BY CNT DESC;

-- 03B - Relevant Preview
SELECT
    IS_RELEVANT,
    SUBJECT_GATE,
    EMAIL_SOURCE_TYPE,
    ARTICLE_RANK,
    ARTICLE_TITLE,
    ARTICLE_URL
FROM BI.NEWS.PHARMA_NEWS_RELEVANCE
ORDER BY IS_RELEVANT DESC, SUBJECT_GATE, ARTICLE_TITLE
LIMIT 100;

-- 03.5 - Clean Article Titles Before AI
CREATE OR REPLACE VIEW BI.NEWS.V_PHARMA_NEWS_ARTICLES_CLEAN AS
SELECT
    MESSAGE_ID,
    SENDER_NAME,
    SENDER_EMAIL,
    SUBJECT_RAW,
    RECEIVED_TS,
    RECEIVED_TS_PARSED,
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

    ARTICLE_URL,
    ARTICLE_URL_EXTRACTION_METHOD,
    BODY_BEST,
    ARTICLE_LLM_INPUT,
    PARSER_VERSION
FROM BI.NEWS.V_PHARMA_NEWS_ARTICLES_BASE;

-- 03.6 - Clean Subject and Title Gate
CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_SUBJECT_GATE_CLEAN AS
SELECT
    *,
    CASE
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'a message from %' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'brought to you by %' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'by %' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '[%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%unsubscribe%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%privacy policy%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%contact support%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%linkedin logo%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%facebook logo%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%twitter logo%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%youtube logo%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%brand logo%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%questex signature%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%webinar%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%podcast%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%event%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%whitepaper%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%download the white paper%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%register now%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%register today%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%save your spot%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'http%' THEN 'DROP'
        WHEN LENGTH(ARTICLE_TITLE_CLEAN) < 20 THEN 'DROP'

        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%acquisition%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%buyout%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%deal%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%fda%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%approval%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%manufacturing%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%facility%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%expansion%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%phase 3%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%partnership%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%ipo%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%fundraising%' THEN 'PASS'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%investment%' THEN 'PASS'

        ELSE 'REVIEW'
    END AS SUBJECT_GATE
FROM BI.NEWS.V_PHARMA_NEWS_ARTICLES_CLEAN;

-- 03.7 - Preview Clean Non-Drop Rows
SELECT
    SUBJECT_GATE,
    EMAIL_SOURCE_TYPE,
    ARTICLE_RANK,
    ARTICLE_TITLE_CLEAN,
    ARTICLE_URL
FROM BI.NEWS.PHARMA_NEWS_SUBJECT_GATE_CLEAN
WHERE SUBJECT_GATE IN ('PASS', 'REVIEW')
ORDER BY SUBJECT_GATE, ARTICLE_TITLE_CLEAN
LIMIT 100;

-- 04 - Relevance From Clean Gate
CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_RELEVANCE AS
SELECT
    *,
    CASE
        WHEN SUBJECT_GATE = 'DROP' THEN FALSE
        ELSE AI_FILTER(
            PROMPT(
                'Return TRUE if this pharma news item is relevant for a Business Intelligence & Insights team at a biologics/CDMO company. Relevant examples include competitor investments, manufacturing, site expansions, partnerships, acquisitions, customer/commercial signals, financing, IPOs, regulatory milestones, clinical milestones with business impact, supply chain changes, platform/capability updates, and strategic market signals. Not relevant examples include webinars, podcasts, events, whitepapers, sponsor messages, logos, footer links, unsubscribe links, admin content, and generic promotions. Subject: {0}. Title: {1}. URL: {2}. Body: {3}',
                SUBJECT_RAW,
                ARTICLE_TITLE_CLEAN,
                ARTICLE_URL,
                BODY_BEST
            )
        )
    END AS IS_RELEVANT
FROM BI.NEWS.PHARMA_NEWS_SUBJECT_GATE_CLEAN
WHERE SUBJECT_GATE IN ('PASS', 'REVIEW');

-- 05 - Classification From Clean Relevance
CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_CLASSIFIED AS
SELECT
    *,
    CASE
        WHEN IS_RELEVANT THEN
            AI_CLASSIFY(
                CONCAT(
                    'Subject: ', COALESCE(SUBJECT_RAW, ''), '. ',
                    'Title: ', COALESCE(ARTICLE_TITLE_CLEAN, ''), '. ',
                    'URL: ', COALESCE(ARTICLE_URL, ''), '. ',
                    'Body: ', COALESCE(BODY_BEST, '')
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
            )
        ELSE NULL
    END AS CATEGORY_RESULT
FROM BI.NEWS.PHARMA_NEWS_RELEVANCE;

-- 05.5 - Clean Title Filter Before Final AI
CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_SUBJECT_GATE_FINAL AS
SELECT
    *,
    CASE
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'a message from %' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'brought to you by %' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'by %' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'staff writers:%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'staff writer:%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'senior editor:%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'senior editors:%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'associate editor:%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'associate editors:%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'executive editor:%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'deputy editor:%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'editor-in-chief:%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'publisher:%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'contributing writer:%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'click here to continue reading%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%conference%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%exhibition%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%innovation week%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%pharma ci%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%fierce biotech week%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%fierce pharma marketing innovation week%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%outsourcing awards%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%podcast%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%webinar%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%event%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%whitepaper%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%register now%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%register today%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '%save your spot%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE '[%' THEN 'DROP'
        WHEN LOWER(ARTICLE_TITLE_CLEAN) LIKE 'http%' THEN 'DROP'
        WHEN LENGTH(ARTICLE_TITLE_CLEAN) < 20 THEN 'DROP'
        ELSE SUBJECT_GATE
    END AS SUBJECT_GATE_FINAL
FROM BI.NEWS.PHARMA_NEWS_SUBJECT_GATE_CLEAN;

-- 05.6 - Preview Final Non-Drop Rows
SELECT
    SUBJECT_GATE_FINAL,
    EMAIL_SOURCE_TYPE,
    ARTICLE_RANK,
    ARTICLE_TITLE_CLEAN,
    ARTICLE_URL
FROM BI.NEWS.PHARMA_NEWS_SUBJECT_GATE_FINAL
WHERE SUBJECT_GATE_FINAL IN ('PASS', 'REVIEW')
ORDER BY SUBJECT_GATE_FINAL, ARTICLE_TITLE_CLEAN
LIMIT 100;

-- 05.7 - Relevance From Final Gate
CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_RELEVANCE AS
SELECT
    *,
    CASE
        WHEN SUBJECT_GATE_FINAL = 'DROP' THEN FALSE
        ELSE AI_FILTER(
            PROMPT(
                'Return TRUE if this pharma news item is relevant for a Business Intelligence & Insights team at a biologics/CDMO company. Relevant examples include competitor investments, manufacturing, site expansions, partnerships, acquisitions, customer/commercial signals, financing, IPOs, regulatory milestones, clinical milestones with business impact, supply chain changes, platform/capability updates, and strategic market signals. Not relevant examples include webinars, podcasts, events, whitepapers, sponsor messages, logos, footer links, unsubscribe links, admin content, editorial staff listings, and generic promotions. Subject: {0}. Title: {1}. URL: {2}. Body: {3}',
                SUBJECT_RAW,
                ARTICLE_TITLE_CLEAN,
                ARTICLE_URL,
                BODY_BEST
            )
        )
    END AS IS_RELEVANT
FROM BI.NEWS.PHARMA_NEWS_SUBJECT_GATE_FINAL
WHERE SUBJECT_GATE_FINAL IN ('PASS', 'REVIEW');

-- 05.8 - Classification From Final Relevance
CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_CLASSIFIED AS
SELECT
    *,
    CASE
        WHEN IS_RELEVANT THEN
            AI_CLASSIFY(
                CONCAT(
                    'Subject: ', COALESCE(SUBJECT_RAW, ''), '. ',
                    'Title: ', COALESCE(ARTICLE_TITLE_CLEAN, ''), '. ',
                    'URL: ', COALESCE(ARTICLE_URL, ''), '. ',
                    'Body: ', COALESCE(BODY_BEST, '')
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
            )
        ELSE NULL
    END AS CATEGORY_RESULT
FROM BI.NEWS.PHARMA_NEWS_RELEVANCE;

-- 05.9 - Final Classification Check
SELECT
    ARTICLE_TITLE_CLEAN,
    CATEGORY_RESULT
FROM BI.NEWS.PHARMA_NEWS_CLASSIFIED
WHERE IS_RELEVANT = TRUE
ORDER BY ARTICLE_TITLE_CLEAN
LIMIT 100;

-- 06_TINY_V2 - Priority Test on 5 Rows With Stronger Scoring Rules
CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_PRIORITY_TEST AS
WITH CANDIDATES AS (
    SELECT *
    FROM BI.NEWS.PHARMA_NEWS_CLASSIFIED
    WHERE IS_RELEVANT = TRUE
      AND SUBJECT_GATE_FINAL = 'PASS'
    ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
    LIMIT 5
)
SELECT
    *,
    AI_COMPLETE(
        model => 'llama3.3-70b',
        prompt => CONCAT(
            'You are prioritizing pharma news for a Business Intelligence & Insights team at a biologics/CDMO company. ',
            'Return structured output only. ',
            'Use the full importance scale from 1 to 5. ',
            'Scoring rules: ',
            '1 = low-value or minor update with limited strategic relevance. ',
            '2 = somewhat relevant but not important right now. ',
            '3 = useful to monitor but not urgent. ',
            '4 = important strategic update that should likely be reviewed by the team. ',
            '5 = highly important or urgent item with major competitive, manufacturing, customer, regulatory, or market impact. ',
            'Do not default to 4. Use 4 or 5 only when clearly justified. ',
            'Subject: ', COALESCE(SUBJECT_RAW, ''), '. ',
            'Title: ', COALESCE(ARTICLE_TITLE_CLEAN, ''), '. ',
            'URL: ', COALESCE(ARTICLE_URL, ''), '. ',
            'Categories: ', COALESCE(TO_JSON(CATEGORY_RESULT), ''), '. ',
            'Body: ', COALESCE(LEFT(BODY_BEST, 1500), '')
        ),
        response_format => TYPE OBJECT(
            importance_score NUMBER,
            why_it_matters STRING,
            recommended_action STRING
        )
    ) AS PRIORITY_RESULT
FROM CANDIDATES;

-- 06A_TINY_V2 - Priority Test Preview
SELECT
    ARTICLE_TITLE_CLEAN,
    PRIORITY_RESULT:importance_score::INT AS IMPORTANCE_SCORE,
    PRIORITY_RESULT:why_it_matters::STRING AS WHY_IT_MATTERS,
    PRIORITY_RESULT:recommended_action::STRING AS RECOMMENDED_ACTION
FROM BI.NEWS.PHARMA_NEWS_PRIORITY_TEST
ORDER BY IMPORTANCE_SCORE DESC, ARTICLE_TITLE_CLEAN;

-- 06_TEN_MIXED - Priority Test on 10 Mixed Rows
CREATE OR REPLACE TABLE BI.NEWS.PHARMA_NEWS_PRIORITY_TEST AS
WITH PASS_ROWS AS (
    SELECT *
    FROM BI.NEWS.PHARMA_NEWS_CLASSIFIED
    WHERE IS_RELEVANT = TRUE
      AND SUBJECT_GATE_FINAL = 'PASS'
    ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
    LIMIT 5
),
REVIEW_ROWS AS (
    SELECT *
    FROM BI.NEWS.PHARMA_NEWS_CLASSIFIED
    WHERE IS_RELEVANT = TRUE
      AND SUBJECT_GATE_FINAL = 'REVIEW'
    ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
    LIMIT 5
),
CANDIDATES AS (
    SELECT * FROM PASS_ROWS
    UNION ALL
    SELECT * FROM REVIEW_ROWS
)
SELECT
    *,
    AI_COMPLETE(
        model => 'llama3.3-70b',
        prompt => CONCAT(
            'You are prioritizing pharma news for a Business Intelligence & Insights team at a biologics/CDMO company. ',
            'Return structured output only. ',
            'Use the full importance scale from 1 to 5. ',
            '1 = low-value or minor update with limited strategic relevance. ',
            '2 = somewhat relevant but not important right now. ',
            '3 = useful to monitor but not urgent. ',
            '4 = important strategic update that should likely be reviewed by the team. ',
            '5 = highly important or urgent item with major competitive, manufacturing, customer, regulatory, or market impact. ',
            'Do not default to 3 or 4. Use the full scale when justified. ',
            'Subject: ', COALESCE(SUBJECT_RAW, ''), '. ',
            'Title: ', COALESCE(ARTICLE_TITLE_CLEAN, ''), '. ',
            'URL: ', COALESCE(ARTICLE_URL, ''), '. ',
            'Categories: ', COALESCE(TO_JSON(CATEGORY_RESULT), ''), '. ',
            'Body: ', COALESCE(LEFT(BODY_BEST, 1500), '')
        ),
        response_format => TYPE OBJECT(
            importance_score NUMBER,
            why_it_matters STRING,
            recommended_action STRING
        )
    ) AS PRIORITY_RESULT
FROM CANDIDATES;

-- 06A_TEN_MIXED - Priority Test Preview
SELECT
    SUBJECT_GATE_FINAL,
    ARTICLE_TITLE_CLEAN,
    PRIORITY_RESULT:importance_score::INT AS IMPORTANCE_SCORE,
    PRIORITY_RESULT:why_it_matters::STRING AS WHY_IT_MATTERS,
    PRIORITY_RESULT:recommended_action::STRING AS RECOMMENDED_ACTION
FROM BI.NEWS.PHARMA_NEWS_PRIORITY_TEST
ORDER BY IMPORTANCE_SCORE DESC, SUBJECT_GATE_FINAL, ARTICLE_TITLE_CLEAN;

-- 07 - Deterministic Priority Bucket
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
              LOWER(ARTICLE_TITLE_CLEAN) RLIKE 'acquisition|buyout|approval|phase 3|fda|investment|facility|manufacturing|expansion|ipo|fundraising|partnership|deal'
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
FROM BI.NEWS.PHARMA_NEWS_CLASSIFIED
WHERE IS_RELEVANT = TRUE;

-- 07A - Priority Bucket Distribution
SELECT
    PRIORITY_BUCKET,
    COUNT(*) AS CNT
FROM BI.NEWS.V_PHARMA_NEWS_PRIORITY_BUCKET
GROUP BY PRIORITY_BUCKET
ORDER BY CNT DESC;

-- 07B - Priority Bucket Preview
SELECT
    PRIORITY_BUCKET,
    EMAIL_SOURCE_TYPE,
    ARTICLE_TITLE,
    ARTICLE_URL,
    CATEGORY_RESULT
FROM BI.NEWS.V_PHARMA_NEWS_PRIORITY_BUCKET
ORDER BY PRIORITY_BUCKET, ARTICLE_TITLE
LIMIT 100;

-- 08 - AI Explanation For High Priority Only
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

-- 08A - High Priority Explanation Preview
SELECT
    ARTICLE_TITLE,
    PRIORITY_BUCKET,
    AI_EXPLANATION:why_it_matters::STRING AS WHY_IT_MATTERS,
    AI_EXPLANATION:recommended_action::STRING AS RECOMMENDED_ACTION
FROM BI.NEWS.PHARMA_NEWS_HIGH_PRIORITY_EXPLAINED
LIMIT 100;

-- 09 - Final Action Queue
CREATE OR REPLACE VIEW BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE AS
SELECT
    RECEIVED_TS_PARSED,
    EMAIL_SOURCE_TYPE,
    ARTICLE_TITLE,
    ARTICLE_URL,
    PRIORITY_BUCKET,
    AI_EXPLANATION:why_it_matters::STRING AS WHY_IT_MATTERS,
    AI_EXPLANATION:recommended_action::STRING AS RECOMMENDED_ACTION
FROM BI.NEWS.PHARMA_NEWS_HIGH_PRIORITY_EXPLAINED
ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST, ARTICLE_TITLE;

-- 09A - Final Action Queue Preview
SELECT
    RECEIVED_TS_PARSED,
    EMAIL_SOURCE_TYPE,
    ARTICLE_TITLE,
    ARTICLE_URL,
    PRIORITY_BUCKET,
    WHY_IT_MATTERS,
    RECOMMENDED_ACTION
FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE
LIMIT 100;

-- 09B - Final Action Queue Clean
CREATE OR REPLACE VIEW BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN AS
SELECT *
FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE
WHERE 1=1
  AND LENGTH(ARTICLE_TITLE) >= 20
  AND LOWER(ARTICLE_TITLE) NOT LIKE '%read in browser%'
  AND LOWER(ARTICLE_TITLE) NOT LIKE 'click here%'
  AND LOWER(ARTICLE_TITLE) NOT LIKE 'brought to you by %'
  AND LOWER(ARTICLE_TITLE) NOT LIKE 'staff writer:%'
  AND LOWER(ARTICLE_TITLE) NOT LIKE 'staff writers:%'
  AND LOWER(ARTICLE_TITLE) NOT LIKE 'senior editor:%'
  AND LOWER(ARTICLE_TITLE) NOT LIKE 'executive editor:%'
  AND LOWER(ARTICLE_TITLE) NOT LIKE 'publisher:%'
  AND LOWER(ARTICLE_TITLE) NOT LIKE 'the unit for %'
  AND LOWER(ARTICLE_TITLE) NOT LIKE 'plus %'
  AND NOT REGEXP_LIKE(ARTICLE_TITLE, '^[a-z]')
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY ARTICLE_TITLE, ARTICLE_URL
    ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
) = 1;

-- 09C - Final Action Queue Clean Preview
SELECT
    RECEIVED_TS_PARSED,
    EMAIL_SOURCE_TYPE,
    ARTICLE_TITLE,
    ARTICLE_URL,
    PRIORITY_BUCKET,
    WHY_IT_MATTERS,
    RECOMMENDED_ACTION
FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN
LIMIT 100;

-- 10 - Create Email Notification Integration
CREATE OR REPLACE NOTIFICATION INTEGRATION PHARMA_NEWS_EMAIL_INT
    TYPE = EMAIL
    ENABLED = TRUE
    ALLOWED_RECIPIENTS = ('s235701@dtu.dk');

    -- 11 - Test Email Send
CALL SYSTEM$SEND_EMAIL(
    'PHARMA_NEWS_EMAIL_INT',
    's235701@dtu.dk',
    'Test: Pharma News Snowflake Email',
    'This is a test email from Snowflake for the Pharma News Monitoring MVP.'
);

-- 12 - Stored Procedure: Send Daily Pharma News Digest
CREATE OR REPLACE PROCEDURE BI.NEWS.SP_SEND_DAILY_PHARMA_NEWS_DIGEST()
RETURNS STRING
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
    V_BODY STRING;
    V_COUNT NUMBER;
BEGIN
    SELECT COUNT(*)
    INTO :V_COUNT
    FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN
    WHERE RECEIVED_TS_PARSED >= DATEADD('day', -7, CURRENT_TIMESTAMP());

    IF (V_COUNT = 0) THEN
        RETURN 'No pharma news items to send.';
    END IF;

    SELECT
        'Daily Pharma News Digest' || CHR(10) ||
        'Generated from Snowflake action queue' || CHR(10) ||
        'Number of high-priority items: ' || :V_COUNT || CHR(10) || CHR(10) ||
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
    FROM (
        SELECT
            RECEIVED_TS_PARSED,
            EMAIL_SOURCE_TYPE,
            ARTICLE_TITLE,
            ARTICLE_URL,
            PRIORITY_BUCKET,
            WHY_IT_MATTERS,
            RECOMMENDED_ACTION
        FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN
        WHERE RECEIVED_TS_PARSED >= DATEADD('day', -7, CURRENT_TIMESTAMP())
        ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
        LIMIT 10
    );

    CALL SYSTEM$SEND_EMAIL(
        'PHARMA_NEWS_EMAIL_INT',
        's235701@dtu.dk',
        'Daily Pharma News Digest',
        :V_BODY
    );

    RETURN 'Daily pharma news digest sent. Items found: ' || V_COUNT;
END;
$$;

-- 12A - Test Daily Digest Procedure
CALL BI.NEWS.SP_SEND_DAILY_PHARMA_NEWS_DIGEST();

-- 12B - Stored Procedure: Send Cleaner Daily Pharma News Digest
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
    WHERE RECEIVED_TS_PARSED >= DATEADD('day', -7, CURRENT_TIMESTAMP());

    IF (V_TOTAL_COUNT = 0) THEN
        RETURN 'No pharma news items to send.';
    END IF;

    WITH DEDUPED AS (
        SELECT
            RECEIVED_TS_PARSED,
            EMAIL_SOURCE_TYPE,
            ARTICLE_TITLE,
            ARTICLE_URL,
            PRIORITY_BUCKET,
            WHY_IT_MATTERS,
            RECOMMENDED_ACTION,
            ROW_NUMBER() OVER (
                PARTITION BY LOWER(
                    REGEXP_REPLACE(
                        REGEXP_REPLACE(ARTICLE_TITLE, '[^a-zA-Z0-9 ]', ''),
                        '\\s+',
                        ' '
                    )
                )
                ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
            ) AS RN
        FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN
        WHERE RECEIVED_TS_PARSED >= DATEADD('day', -7, CURRENT_TIMESTAMP())
    ),
    TOP_ITEMS AS (
        SELECT *
        FROM DEDUPED
        WHERE RN = 1
        ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
        LIMIT 10
    )
    SELECT COUNT(*)
    INTO :V_SENT_COUNT
    FROM TOP_ITEMS;

    WITH DEDUPED AS (
        SELECT
            RECEIVED_TS_PARSED,
            EMAIL_SOURCE_TYPE,
            ARTICLE_TITLE,
            ARTICLE_URL,
            PRIORITY_BUCKET,
            WHY_IT_MATTERS,
            RECOMMENDED_ACTION,
            ROW_NUMBER() OVER (
                PARTITION BY LOWER(
                    REGEXP_REPLACE(
                        REGEXP_REPLACE(ARTICLE_TITLE, '[^a-zA-Z0-9 ]', ''),
                        '\\s+',
                        ' '
                    )
                )
                ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
            ) AS RN
        FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN
        WHERE RECEIVED_TS_PARSED >= DATEADD('day', -7, CURRENT_TIMESTAMP())
    ),
    TOP_ITEMS AS (
        SELECT *
        FROM DEDUPED
        WHERE RN = 1
        ORDER BY RECEIVED_TS_PARSED DESC NULLS LAST
        LIMIT 10
    )
    SELECT
        'Daily Pharma News Digest' || CHR(10) ||
        'Generated from Snowflake action queue' || CHR(10) ||
        'High-priority items found in last 7 days: ' || :V_TOTAL_COUNT || CHR(10) ||
        'Showing top items after deduplication: ' || :V_SENT_COUNT || CHR(10) ||
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

    RETURN 'Daily pharma news digest sent. Total found: ' || V_TOTAL_COUNT || ', sent after dedupe: ' || V_SENT_COUNT;
END;
$$;

-- 12C - Test Cleaner Daily Digest Procedure
CALL BI.NEWS.SP_SEND_DAILY_PHARMA_NEWS_DIGEST();

-- 12E - Count High-Priority Items From Last 24 Hours
SELECT COUNT(*) AS CNT
FROM BI.NEWS.V_PHARMA_NEWS_ACTION_QUEUE_CLEAN
WHERE RECEIVED_TS_PARSED >= DATEADD('day', -1, CURRENT_TIMESTAMP());
