LEFT JOIN PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES S
    ON SHA2(
        LOWER(
            COALESCE(D.ARTICLE_URL, '') || '|' ||
            COALESCE(D.ARTICLE_TITLE, '')
        ),
        256
    ) = S.SUMMARY_KEY
   AND S.SUMMARY_VERSION = 'v2_1_3_paragraphs'
