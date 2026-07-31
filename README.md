-- ============================================================
-- 06 - Professional HTML Email Output View
-- V6 Executive Context Summary Email
-- ============================================================

USE ROLE SANDBOX_DEVELOPER;
USE WAREHOUSE SANDBOX_WH;
USE DATABASE PHARMA_NEWS_SANDBOX;
USE SCHEMA NEWS;


CREATE OR REPLACE VIEW PHARMA_NEWS_SANDBOX.NEWS.V_ALTERYX_DAILY_PHARMA_NEWS_EMAIL AS
WITH TOP_ITEMS_RAW AS (
    SELECT
        D.RECEIVED_TS_PARSED,
        D.PUBLISH_DATE,
        D.PUBLISH_DATE_SOURCE,
        D.EMAIL_SOURCE_TYPE,
        D.ARTICLE_TITLE,
        D.ARTICLE_URL,
        D.DIGEST_RANK,
        D.ITEM_KEY,

        S.AI_SUMMARY,
        S.NEWS_ARCHETYPE,

        COALESCE(E.SOURCE_DEPTH, 'snippet_only') AS SOURCE_DEPTH,

        MD5(
            LOWER(
                TRIM(
                    REGEXP_REPLACE(
                        REGEXP_REPLACE(COALESCE(D.ARTICLE_TITLE, ''), '[^a-zA-Z0-9 ]', ' '),
                        '\\s+',
                        ' '
                    )
                )
            )
        ) AS SUMMARY_KEY_STABLE

    FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2 D
    LEFT JOIN PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES S
        ON MD5(
               LOWER(
                   TRIM(
                       REGEXP_REPLACE(
                           REGEXP_REPLACE(COALESCE(D.ARTICLE_TITLE, ''), '[^a-zA-Z0-9 ]', ' '),
                           '\\s+',
                           ' '
                       )
                   )
               )
           ) = S.SUMMARY_KEY
       AND S.SUMMARY_VERSION = 'v6_executive_context_summary'
    LEFT JOIN PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_ENRICHED_ARTICLES E
        ON E.SUMMARY_KEY_STABLE = MD5(
               LOWER(
                   TRIM(
                       REGEXP_REPLACE(
                           REGEXP_REPLACE(COALESCE(D.ARTICLE_TITLE, ''), '[^a-zA-Z0-9 ]', ' '),
                           '\\s+',
                           ' '
                       )
                   )
               )
           )
       AND E.EXTRACTION_STATUS = 'success'
    WHERE D.DIGEST_RANK <= 10
),

TOP_ITEMS AS (
    SELECT
        T.*,

        -- HTML-safe title
        REPLACE(
            REPLACE(
                REPLACE(
                    REPLACE(
                        REPLACE(COALESCE(T.ARTICLE_TITLE, 'Untitled article'), '&', '&amp;'),
                        '<', '&lt;'
                    ),
                    '>', '&gt;'
                ),
                '"', '&quot;'
            ),
            '''', '&#39;'
        ) AS TITLE_HTML,

        -- HTML-safe source
        REPLACE(
            REPLACE(
                REPLACE(
                    REPLACE(
                        REPLACE(COALESCE(T.EMAIL_SOURCE_TYPE, 'Unknown source'), '&', '&amp;'),
                        '<', '&lt;'
                    ),
                    '>', '&gt;'
                ),
                '"', '&quot;'
            ),
            '''', '&#39;'
        ) AS SOURCE_HTML,

        -- HTML-safe archetype
        REPLACE(
            REPLACE(
                REPLACE(
                    REPLACE(
                        REPLACE(COALESCE(T.NEWS_ARCHETYPE, 'General Strategic News'), '&', '&amp;'),
                        '<', '&lt;'
                    ),
                    '>', '&gt;'
                ),
                '"', '&quot;'
            ),
            '''', '&#39;'
        ) AS NEWS_ARCHETYPE_HTML,

        -- HTML-safe V6 summary with section labels emphasized
        REPLACE(
            REPLACE(
                REPLACE(
                    REPLACE(
                        REPLACE(
                            REPLACE(
                                REPLACE(
                                    REPLACE(
                                        REPLACE(
                                            REPLACE(
                                                COALESCE(T.AI_SUMMARY, 'Executive summary pending.'),
                                                '&', '&amp;'
                                            ),
                                            '<', '&lt;'
                                        ),
                                        '>', '&gt;'
                                    ),
                                    '"', '&quot;'
                                ),
                                '''', '&#39;'
                            ),
                            'Executive read:', '<b>Executive read:</b>'
                        ),
                        'Strategic context:', '<br><br><b>Strategic context:</b>'
                    ),
                    'Strategic implication:', '<br><br><b>Strategic implication:</b>'
                ),
                'Watch items:', '<br><br><b>Watch items:</b>'
            ),
            CHAR(10), '<br>'
        ) AS AI_SUMMARY_HTML

    FROM TOP_ITEMS_RAW T
),

METRICS AS (
    SELECT
        COUNT(*) AS TOTAL_TOP_STORIES
    FROM TOP_ITEMS
),

ITEM_CARDS AS (
    SELECT
        DIGEST_RANK,

        '<tr>
            <td style="padding: 0 0 18px 0;">
                <table role="presentation" width="100%" cellspacing="0" cellpadding="0" border="0" style="border: 1px solid #e5e7eb; border-radius: 12px; background-color: #ffffff;">
                    <tr>
                        <td style="padding: 22px 24px 20px 24px;">

                            <div style="font-family: Arial, Helvetica, sans-serif; font-size: 12px; color: #6b7280; letter-spacing: 0.02em; text-transform: uppercase; margin-bottom: 8px;">
                                #' || TO_VARCHAR(DIGEST_RANK) || ' &middot; ' ||
                                SOURCE_HTML || IFF(PUBLISH_DATE IS NOT NULL, ' &middot; ' || TO_VARCHAR(PUBLISH_DATE), '') || '
                            </div>

                            <div style="font-family: Arial, Helvetica, sans-serif; font-size: 20px; line-height: 1.35; font-weight: 700; color: #111827; margin-bottom: 10px;">' ||
                                IFF(
                                    ARTICLE_URL IS NOT NULL AND ARTICLE_URL <> '',
                                    '<a href="' || ARTICLE_URL || '" style="color: #111827; text-decoration: none;">' || TITLE_HTML || '</a>',
                                    TITLE_HTML
                                )
                            || '</div>

                            <div style="margin-bottom: 14px;">
                                <span style="display: inline-block; font-family: Arial, Helvetica, sans-serif; font-size: 12px; color: #374151; background-color: #f3f4f6; border-radius: 999px; padding: 5px 10px; margin: 0 6px 6px 0;">
                                    ' || NEWS_ARCHETYPE_HTML || '
                                </span>' ||
                                IFF(SOURCE_DEPTH IN ('full_article', 'partial_extract'),
                                    '<span style="display: inline-block; font-family: Arial, Helvetica, sans-serif; font-size: 11px; color: #065f46; background-color: #d1fae5; border-radius: 999px; padding: 4px 9px; margin: 0 6px 6px 0;">Full source</span>',
                                    ''
                                ) || '
                            </div>

                            <div style="font-family: Arial, Helvetica, sans-serif; font-size: 14px; line-height: 1.65; color: #111827; background-color: #f9fafb; border-left: 4px solid #2563eb; padding: 14px 16px; border-radius: 8px;">
                                ' || AI_SUMMARY_HTML || '
                            </div>

                        </td>
                    </tr>
                </table>
            </td>
        </tr>' AS ITEM_HTML

    FROM TOP_ITEMS
),

ITEMS_AGG AS (
    SELECT
        LISTAGG(ITEM_HTML, '') WITHIN GROUP (ORDER BY DIGEST_RANK) AS ITEMS_HTML
    FROM ITEM_CARDS
),

EMAIL_ASSEMBLED AS (
    SELECT
        'morten.laugesen@fujifilm.com' AS TO_EMAIL,

        'Daily Pharma News Digest - ' || TO_VARCHAR(CURRENT_DATE()) AS EMAIL_SUBJECT,

        '<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Daily Pharma News Digest</title>
</head>
<body style="margin: 0; padding: 0; background-color: #f3f4f6;">
    <table role="presentation" width="100%" cellspacing="0" cellpadding="0" border="0" style="background-color: #f3f4f6;">
        <tr>
            <td align="center" style="padding: 28px 12px;">
                <table role="presentation" width="680" cellspacing="0" cellpadding="0" border="0" style="width: 680px; max-width: 680px; background-color: #ffffff; border-radius: 16px; overflow: hidden;">

                    <tr>
                        <td style="background-color: #111827; padding: 34px 34px 30px 34px;">
                            <div style="font-family: Arial, Helvetica, sans-serif; font-size: 13px; color: #93c5fd; letter-spacing: 0.08em; text-transform: uppercase; margin-bottom: 10px;">
                                BI &amp; Insights &middot; Strategic News Monitor
                            </div>
                            <div style="font-family: Arial, Helvetica, sans-serif; font-size: 30px; line-height: 1.2; font-weight: 700; color: #ffffff;">
                                Daily Pharma News Digest
                            </div>
                            <div style="font-family: Arial, Helvetica, sans-serif; font-size: 15px; line-height: 1.6; color: #d1d5db; margin-top: 10px;">
                                Executive-context summaries of the most relevant pharma, biopharma, CDMO, pipeline, deal, regulatory and capacity signals.
                            </div>
                        </td>
                    </tr>

                    <tr>
                        <td style="padding: 24px 34px 8px 34px;">
                            <div style="font-family: Arial, Helvetica, sans-serif; font-size: 14px; color: #6b7280; border-bottom: 1px solid #e5e7eb; padding-bottom: 14px;">
                                ' || TO_VARCHAR(M.TOTAL_TOP_STORIES) || ' stories &middot; ' || TO_VARCHAR(CURRENT_DATE()) || '
                            </div>
                        </td>
                    </tr>

                    <tr>
                        <td style="padding: 10px 34px 24px 34px;">
                            <table role="presentation" width="100%" cellspacing="0" cellpadding="0" border="0">
                                ' || COALESCE(A.ITEMS_HTML, '') || '
                            </table>
                        </td>
                    </tr>

                    <tr>
                        <td style="background-color: #f9fafb; padding: 22px 34px; border-top: 1px solid #e5e7eb;">
                            <div style="font-family: Arial, Helvetica, sans-serif; font-size: 12px; line-height: 1.6; color: #6b7280;">
                                Generated by BI &amp; Insights Strategic News Monitor. Summaries are intended for strategic screening and should be fact-checked before external distribution.
                            </div>
                        </td>
                    </tr>

                </table>
            </td>
        </tr>
    </table>
</body>
</html>' AS EMAIL_BODY

    FROM METRICS M
    CROSS JOIN ITEMS_AGG A
)

SELECT
    TO_EMAIL,
    EMAIL_SUBJECT,
    EMAIL_BODY
FROM EMAIL_ASSEMBLED;
