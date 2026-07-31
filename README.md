-- ============================================================
-- 07 - Send email procedure
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
