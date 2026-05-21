-- ============================================================
-- 01 - Clean Article View
-- Fixed for Snowflake table with normal uppercase column names
-- ============================================================

CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_ARTICLES_CLEAN AS
SELECT
    message_id AS MESSAGE_ID,
    sender_name AS SENDER_NAME,
    sender_email AS SENDER_EMAIL,
    subject_raw AS SUBJECT_RAW,
    received_ts AS RECEIVED_TS,
    TRY_TO_TIMESTAMP_TZ(received_ts) AS RECEIVED_TS_PARSED,
    email_source_type AS EMAIL_SOURCE_TYPE,
    article_rank AS ARTICLE_RANK,

    TRIM(
        REGEXP_REPLACE(
            REGEXP_REPLACE(
                REGEXP_REPLACE(article_title, '^[0-9]+\\.?\\s*', ''),
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
                    REGEXP_REPLACE(article_title, '^[0-9]+\\.?\\s*', ''),
                    '^,\\s*',
                    ''
                ),
                '\\s+',
                ' '
            )
        )
    ) AS ARTICLE_TITLE_LC,

    article_url AS ARTICLE_URL,
    article_url_extraction_method AS ARTICLE_URL_EXTRACTION_METHOD,
    body_best AS BODY_BEST,
    article_llm_input AS ARTICLE_LLM_INPUT,
    parser_version AS PARSER_VERSION
FROM PHARMA_NEWS_SANDBOX.NEWS.STG_PHARMA_NEWS_ARTICLES_V1;
