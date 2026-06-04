SHOW NOTIFICATION INTEGRATIONS LIKE 'PHARMA_NEWS_EMAIL_INT';

CALL SYSTEM$SEND_EMAIL(
    'PHARMA_NEWS_EMAIL_INT',
    's235701@dtu.dk',
    'Test email from Snowflake',
    'This is a test email from Snowflake.'
);
