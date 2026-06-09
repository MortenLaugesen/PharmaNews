CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_ARTICLES_CLEAN AS
SELECT
    MESSAGE_ID,
    SENDER_NAME,
    SENDER_EMAIL,
    SUBJECT_RAW,
    RECEIVED_TS,
    TRY_TO_TIMESTAMP_TZ(RECEIVED_TS) AS RECEIVED_TS_PARSED,

    COALESCE(
        TRY_TO_DATE(ARTICLE_PUBLISH_DATE_RAW),
        TO_DATE(TRY_TO_TIMESTAMP_TZ(RECEIVED_TS))
    ) AS PUBLISH_DATE,

    CASE
        WHEN TRY_TO_DATE(ARTICLE_PUBLISH_DATE_RAW) IS NOT NULL
            THEN 'Article publish date'
        WHEN TRY_TO_TIMESTAMP_TZ(RECEIVED_TS) IS NOT NULL
            THEN 'Email received date fallback'
        ELSE 'Unknown'
    END AS PUBLISH_DATE_SOURCE,

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
