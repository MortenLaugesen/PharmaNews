-- ============================================================
-- 05 - AI Summaries
-- V6.4 Executive Context Summary Generation
-- Uses stable title-based summary key
-- Refined for more human analyst tone and executive readability
-- ============================================================

USE ROLE SANDBOX_DEVELOPER;
USE WAREHOUSE SANDBOX_WH;
USE DATABASE PHARMA_NEWS_SANDBOX;
USE SCHEMA NEWS;


-- ============================================================
-- Summary cache table
-- ============================================================

CREATE TABLE IF NOT EXISTS PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES (
    SUMMARY_KEY STRING,
    SUMMARY_VERSION STRING,
    AI_SUMMARY STRING,
    CREATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    UPDATED_AT TIMESTAMP_NTZ,
    ARTICLE_TITLE STRING,
    NEWS_ARCHETYPE STRING,
    MODALITY_SIGNAL STRING,
    COMPANY_CONTEXT_SIGNAL STRING,
    MODEL_NAME STRING
);


-- ============================================================
-- Safe ALTERs for existing cache table
-- These preserve old V5 summaries and add missing V6 metadata columns.
-- IMPORTANT:
-- Do not use DEFAULT CURRENT_TIMESTAMP() in ALTER TABLE for UPDATED_AT.
-- ============================================================

ALTER TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES
    ADD COLUMN IF NOT EXISTS UPDATED_AT TIMESTAMP_NTZ;

ALTER TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES
    ADD COLUMN IF NOT EXISTS ARTICLE_TITLE STRING;

ALTER TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES
    ADD COLUMN IF NOT EXISTS NEWS_ARCHETYPE STRING;

ALTER TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES
    ADD COLUMN IF NOT EXISTS MODALITY_SIGNAL STRING;

ALTER TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES
    ADD COLUMN IF NOT EXISTS COMPANY_CONTEXT_SIGNAL STRING;

ALTER TABLE PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES
    ADD COLUMN IF NOT EXISTS MODEL_NAME STRING;


-- ============================================================
-- Refresh V6 executive context summaries
-- Generates missing V6 summaries for current top 10 digest stories.
--
-- Key design:
-- Uses stable title-based SUMMARY_KEY instead of ITEM_KEY.
-- This avoids cache mismatches when ITEM_KEY changes between views.
--
-- V6.4 improvements:
-- - Use "FDB" instead of full company name
-- - More human analyst tone
-- - Less generic AI language
-- - Section renamed from "Relevance to FUJIFILM / CDMO" to "Strategic implication"
-- - Avoid repeated "direct relevance is limited" phrasing
-- - Avoid "CDMOs like FDB"
-- - Focus on the strongest implication, not forced CDMO relevance
-- ============================================================

CREATE OR REPLACE PROCEDURE PHARMA_NEWS_SANDBOX.NEWS.SP_REFRESH_AI_SUMMARIES()
RETURNS STRING
LANGUAGE SQL
AS
$$
DECLARE
    ROWS_INSERTED INTEGER DEFAULT 0;
BEGIN

    INSERT INTO PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES (
        SUMMARY_KEY,
        SUMMARY_VERSION,
        AI_SUMMARY,
        CREATED_AT,
        UPDATED_AT,
        ARTICLE_TITLE,
        NEWS_ARCHETYPE,
        MODALITY_SIGNAL,
        COMPANY_CONTEXT_SIGNAL,
        MODEL_NAME
    )

    WITH TOP_STORIES AS (
        SELECT
            D.DIGEST_RANK,
            D.ITEM_KEY,

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
            ) AS SUMMARY_KEY_STABLE,

            D.ARTICLE_TITLE,
            D.EMAIL_SOURCE_TYPE,
            D.PUBLISH_DATE,
            D.PRIORITY_TIER,
            D.SIGNAL_SCORE,
            D.SIGNAL_REASONS,
            D.MATCHED_COMPANIES,
            D.MATCHED_COMPANY_CATEGORIES,
            D.DEAL_VALUE_KEY,
            D.ARTICLE_URL,
            D.ARTICLE_SUMMARY_INPUT

        FROM PHARMA_NEWS_SANDBOX.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2 D
        WHERE D.DIGEST_RANK <= 10
          AND D.ARTICLE_TITLE IS NOT NULL
    ),

    NORMALIZED AS (
        SELECT
            T.*,

            LOWER(COALESCE(T.ARTICLE_TITLE, '')) AS TITLE_LC,

            LOWER(
                COALESCE(T.ARTICLE_TITLE, '') || ' ' ||
                COALESCE(T.SIGNAL_REASONS, '') || ' ' ||
                COALESCE(T.MATCHED_COMPANY_CATEGORIES, '') || ' ' ||
                COALESCE(T.ARTICLE_SUMMARY_INPUT, '')
            ) AS CONTEXT_LC

        FROM TOP_STORIES T
    ),

    CLASSIFIED AS (
        SELECT
            N.*,

            CASE
                WHEN REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(headcount|layoffs|layoff|cuts jobs|cut jobs|job cuts|workforce reduction|restructuring|cost-cutting|cost cutting|shrinks headcount|staff cuts|reduces workforce|reduce workforce)\\b.*'
                )
                THEN 'Workforce / Cost Reduction'

                WHEN REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(antitrust|verdict|lawsuit|litigation|settlement|court|legal|expects to take|charge|write-down|writedown|impairment|hit after)\\b.*'
                )
                THEN 'Legal / Financial Impact'

                WHEN REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(ai|artificial intelligence|machine learning|computational|discovery platform|drug discovery platform|platform deal)\\b.*'
                )
                THEN 'AI / Discovery Platform'

                WHEN REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(partnership|partners|partner|collaboration|collaborates|licensing|license|licensed|pact|alliance|option deal|research deal|r&d deal|co-develop|codevelop)\\b.*'
                )
                THEN 'Partnership / Licensing'

                WHEN REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(acquire|acquires|acquired|acquisition|buyout|takeover|merger|merge|to buy|deal to buy|purchase of|snaps up|scoops up|take out)\\b.*'
                )
                OR REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(inks|strikes|signs)\\b.*\\bdeal\\b.*\\b(to add|add|adds)\\b.*'
                )
                OR REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(makes|launches|unveils)\\b.*\\$?[0-9]+(\\.[0-9]+)?\\s?(b|bn|billion|m|mn|million)\\b.*\\b(move for|bid for|offer for)\\b.*'
                )
                THEN 'M&A / Acquisition'

                WHEN REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(manufacturing|facility|plant|site|campus|capacity|expansion|expand|construction|production|production sites|fill-finish|fill finish|biomanufacturing|cdmo|cmo|supplier)\\b.*'
                )
                THEN 'Capacity / Manufacturing / CDMO'

                WHEN REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(fda|ema|approval|approved|pdufa|complete response|crl|phase 1|phase 2|phase 3|phase i|phase ii|phase iii|trial|clinical|readout|data|filing|submission|regulatory|breakthrough therapy|orphan drug|fast track)\\b.*'
                )
                THEN 'Clinical / Regulatory Catalyst'

                WHEN REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(funding|financing|series a|series b|series c|ipo|private placement|raises|raise|investment round|venture round)\\b.*'
                )
                THEN 'Financing / Funding'

                WHEN REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(launch|commercial|sales|market access|pricing|reimbursement|payer|blockbuster|label expansion|market share)\\b.*'
                )
                THEN 'Commercial / Market Access'

                ELSE 'General Strategic News'
            END AS NEWS_ARCHETYPE,

            CASE
                WHEN REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(headcount|layoffs|layoff|cuts jobs|cut jobs|job cuts|workforce reduction|restructuring|cost-cutting|cost cutting|shrinks headcount|staff cuts|antitrust|verdict|lawsuit|litigation|settlement|court|legal|charge|write-down|writedown|impairment)\\b.*'
                )
                THEN 'Not modality-driven'

                WHEN REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(tki|tyrosine kinase inhibitor|small molecule|oral drug|tablet|capsule)\\b.*'
                )
                THEN 'Small molecule signal'

                WHEN REGEXP_LIKE(
                    N.TITLE_LC,
                    '.*\\b(adc|antibody-drug conjugate|antibody drug conjugate|monoclonal antibody|mab|bispecific|cell therapy|gene therapy|viral vector|protein|enzyme|vaccine|biologic|biologics|rna|mrna|sirna|oligonucleotide|plasmid)\\b.*'
                )
                THEN 'Biologics / advanced modality signal'

                WHEN REGEXP_LIKE(
                    N.CONTEXT_LC,
                    '.*\\b(tki|tyrosine kinase inhibitor|small molecule|oral drug|tablet|capsule)\\b.*'
                )
                THEN 'Small molecule signal'

                WHEN REGEXP_LIKE(
                    N.CONTEXT_LC,
                    '.*\\b(adc|antibody-drug conjugate|antibody drug conjugate|monoclonal antibody|mab|bispecific|cell therapy|gene therapy|viral vector|protein|enzyme|vaccine|biologic|biologics|rna|mrna|sirna|oligonucleotide|plasmid)\\b.*'
                )
                THEN 'Biologics / advanced modality signal'

                ELSE 'Modality unclear or mixed'
            END AS MODALITY_SIGNAL,

            CASE
                WHEN COALESCE(N.MATCHED_COMPANY_CATEGORIES, '') LIKE '%CDMO%'
                THEN 'CDMO / supplier signal'

                WHEN REGEXP_LIKE(
                    N.CONTEXT_LC,
                    '.*\\b(gsk|pfizer|roche|novartis|merck|msd|astrazeneca|lilly|eli lilly|novo nordisk|sanofi|bristol myers|bms|johnson & johnson|j&j|takeda|abbvie|amgen|gilead|bayer|boehringer|biogen|moderna|regeneron|incyte)\\b.*'
                )
                THEN 'Large pharma / Top pharma strategic signal'

                ELSE 'Emerging biotech / other company signal'
            END AS COMPANY_CONTEXT_SIGNAL

        FROM NORMALIZED N
    ),

    PROMPTED AS (
        SELECT
            C.*,

            'You are a biopharmaceutical strategy analyst writing for an executive audience at FDB.

Your job is to turn pharma news into executive intelligence. Do not merely restate the article. Explain the strategic meaning behind the event in a way that sounds like a human analyst wrote it.

Audience:
- Executive-level readers.
- Strong biopharma industry fluency.
- Not necessarily deep expertise in the specific therapeutic or scientific domain.
- They want a concise read on why this matters without needing to read every article.

Core task:
Explain:
1. What happened.
2. Why it matters now.
3. What strategic pressure, portfolio logic, modality trend, capacity signal, customer/competitor movement, or market catalyst it reflects.
4. The strongest strategic implication for FDB, CDMOs, customers, competitors, or the broader manufacturing and development landscape.
5. What should be monitored next.

Important writing rules:
- Use "FDB" when referring to FUJIFILM Diosynth Biotechnologies.
- Do not write "FUJIFILM Diosynth Biotechnologies" in the output.
- Do not write "CDMOs like FDB."
- Write like a concise human analyst, not an AI system.
- Avoid repetitive sentence patterns across sections.
- Avoid generic consulting language such as:
  "this highlights the growing importance of innovation",
  "this underscores the need for strategic focus",
  "this reflects the dynamic nature of the industry",
  "this may lead to increased demand for CDMO services",
  unless tied to a specific asset, modality, company pressure, capability gap, geography, or market event from the source text.
- Do not force a CDMO angle.
- Do not repeatedly say "direct relevance is limited."
- If the FDB/CDMO implication is weak, do not dwell on it. Instead, state the strongest indirect signal in one specific sentence.
- If the article is mainly small molecule, legal, financial, AI discovery, or early research, avoid implying near-term manufacturing demand.
- Do not say "no monetary value was provided." If a monetary value is absent or irrelevant, omit it.
- Do not write a generic conclusion.
- Keep the tone sharp, natural, and useful for a senior business reader.

Accuracy rules:
- Use only the supplied article text and metadata for specific facts.
- You may use general industry knowledge only to frame logic, not to invent facts.
- Do not invent dates, revenue numbers, approval dates, clinical trial names, enrollment numbers, patient population sizes, or deal terms.
- If a claim is an interpretation rather than a sourced fact from the article, phrase it carefully.
- If the supplied article text is limited, produce a shorter answer and do not fill gaps with unsupported detail.

News archetype:
' || C.NEWS_ARCHETYPE || '

Modality signal:
' || C.MODALITY_SIGNAL || '

Company context signal:
' || C.COMPANY_CONTEXT_SIGNAL || '

Metadata:
Headline: ' || COALESCE(C.ARTICLE_TITLE, 'N/A') || '
Source: ' || COALESCE(C.EMAIL_SOURCE_TYPE, 'N/A') || '
Publish date: ' || COALESCE(TO_VARCHAR(C.PUBLISH_DATE), 'N/A') || '
Priority tier: ' || COALESCE(C.PRIORITY_TIER, 'N/A') || '
Signal score: ' || COALESCE(TO_VARCHAR(C.SIGNAL_SCORE), 'N/A') || '
Signal reasons: ' || COALESCE(C.SIGNAL_REASONS, 'N/A') || '
Matched companies: ' || COALESCE(C.MATCHED_COMPANIES, 'N/A') || '
Matched company categories: ' || COALESCE(C.MATCHED_COMPANY_CATEGORIES, 'N/A') || '
Deal value key: ' || COALESCE(C.DEAL_VALUE_KEY, 'N/A') || '

Article/source text:
' || COALESCE(LEFT(C.ARTICLE_SUMMARY_INPUT, 12000), 'No article text available') || '

Archetype guidance:

For M&A / Acquisition:
- Explain what the buyer gains.
- Explain the likely portfolio gap, franchise logic, or capability gap the deal addresses, but only if supported by the article or clearly framed as interpretation.
- Distinguish near-term asset acquisition from broader platform/franchise logic.
- If the deal is small molecule, avoid implying biologics manufacturing demand unless the source supports it.
- Do not overstate synergies unless they are in the source text.

For Partnership / Licensing:
- Explain what each party contributes.
- Identify whether the deal is about platform access, pipeline acceleration, modality entry, risk-sharing, commercial reach, or optionality.
- Do not call it an acquisition.
- For AI/discovery/platform deals, distinguish discovery-stage relevance from downstream manufacturing relevance.

For AI / Discovery Platform:
- Explain what the AI or platform capability is intended to improve.
- Distinguish discovery-stage relevance from downstream development/manufacturing relevance.
- Do not imply near-term CDMO demand unless the source indicates development or manufacturing progression.
- Focus on the indirect signal: faster discovery cycles, more shots on goal, better design quality, or modality expansion.

For Clinical / Regulatory Catalyst:
- Explain what changed in the asset’s development or approval path.
- Explain why the indication, modality, or competitive landscape matters.
- Identify whether the event may accelerate demand, create competitive pressure, or de-risk a program.
- Do not invent trial details not in the source text.

For Capacity / Manufacturing / CDMO:
- Explain what capacity, site, modality, or geography is changing.
- Explain what it signals about demand, outsourcing, internalization, supply risk, or competitive capacity.
- Identify the manufacturing implication clearly.

For Workforce / Cost Reduction:
- Explain what is being reduced and where.
- Explain whether this looks like cost discipline, restructuring, portfolio prioritization, or site-level operational change.
- Focus on indirect signals such as customer cost pressure, portfolio pruning, or shifting investment priorities. Do not imply direct outsourcing demand unless stated.

For Legal / Financial Impact:
- Explain the financial/legal event and scale of impact.
- Explain why it may matter for capital allocation, risk exposure, management attention, or strategic flexibility.
- Avoid forcing an FDB or CDMO angle unless the article links the event to pipeline, manufacturing, or outsourcing decisions.

For Financing / Funding:
- Explain what the funding enables.
- Explain whether the company is likely moving toward clinical development, manufacturing scale-up, or platform validation.
- Avoid implying near-term CDMO opportunity unless the article supports it.

For Commercial / Market Access:
- Explain what changed commercially.
- Explain relevance to market structure, pricing, payer dynamics, adoption, or competitive positioning.

Write the output in exactly this structure:

Executive read:
One concise paragraph. State what happened and include the most important specifics from the article.

Strategic context:
One concise paragraph. Explain the "why behind the what." Tie the news to a specific strategic pressure, modality trend, competitive dynamic, capacity issue, pipeline implication, or portfolio logic. Avoid generic industry commentary.

Strategic implication:
One concise paragraph. Focus on the strongest implication for FDB, CDMOs, customers, competitors, or the broader development/manufacturing landscape. If there is no strong FDB/CDMO implication, focus on the broader strategic signal instead. Do not start with "direct relevance is limited."

Watch items:
Give 1-3 short bullets. Each bullet must be concrete and monitorable.

Target length:
170-280 words total. Shorter is better if the article text is limited.'
            AS EXECUTIVE_PROMPT

        FROM CLASSIFIED C
    ),

    GENERATED AS (
        SELECT
            P.SUMMARY_KEY_STABLE,
            P.ARTICLE_TITLE,
            P.NEWS_ARCHETYPE,
            P.MODALITY_SIGNAL,
            P.COMPANY_CONTEXT_SIGNAL,

            AI_COMPLETE(
                'llama3.3-70b',
                P.EXECUTIVE_PROMPT,
                {
                    'temperature': 0,
                    'max_tokens': 950
                }
            ) AS V6_EXECUTIVE_CONTEXT_SUMMARY

        FROM PROMPTED P
        WHERE NOT EXISTS (
            SELECT 1
            FROM PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES S
            WHERE S.SUMMARY_KEY = P.SUMMARY_KEY_STABLE
              AND S.SUMMARY_VERSION = 'v6_executive_context_summary'
        )
    )

    SELECT
        G.SUMMARY_KEY_STABLE AS SUMMARY_KEY,
        'v6_executive_context_summary' AS SUMMARY_VERSION,
        G.V6_EXECUTIVE_CONTEXT_SUMMARY AS AI_SUMMARY,
        CURRENT_TIMESTAMP() AS CREATED_AT,
        CURRENT_TIMESTAMP() AS UPDATED_AT,
        G.ARTICLE_TITLE,
        G.NEWS_ARCHETYPE,
        G.MODALITY_SIGNAL,
        G.COMPANY_CONTEXT_SIGNAL,
        'llama3.3-70b' AS MODEL_NAME
    FROM GENERATED G;

    ROWS_INSERTED := SQLROWCOUNT;

    RETURN 'V6.4 executive context summaries refreshed. Rows inserted: ' || ROWS_INSERTED;

END;
$$;


-- ============================================================
-- One-time cleanup before regenerating V6.4 with refined prompt
-- Run manually once. This only deletes V6 summaries and preserves V5.
-- ============================================================

-- DELETE FROM PHARMA_NEWS_SANDBOX.NEWS.PHARMA_NEWS_AI_SUMMARIES
-- WHERE SUMMARY_VERSION = 'v6_executive_context_summary';


-- ============================================================
-- Manual refresh command
-- Run manually in Snowflake when you want to generate missing V6.4 summaries.
-- ============================================================

-- CALL PHARMA_NEWS_SANDBOX.NEWS.SP_REFRESH_AI_SUMMARIES();


-- ============================================================
-- Preview V6.4 summary coverage using stable title-based summary key
-- ============================================================

SELECT
    D.DIGEST_RANK,
    D.ARTICLE_TITLE,
    D.EMAIL_SOURCE_TYPE,
    D.PUBLISH_DATE,
    D.PRIORITY_TIER,
    S.SUMMARY_VERSION,
    S.NEWS_ARCHETYPE,
    S.MODALITY_SIGNAL,
    S.COMPANY_CONTEXT_SIGNAL,
    CASE
        WHEN S.AI_SUMMARY IS NULL THEN 'NO V6 SUMMARY YET'
        ELSE 'V6 SUMMARY AVAILABLE'
    END AS V6_SUMMARY_STATUS,
    LEFT(S.AI_SUMMARY, 1800) AS V6_SUMMARY_PREVIEW
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
WHERE D.DIGEST_RANK <= 10
ORDER BY D.DIGEST_RANK;
