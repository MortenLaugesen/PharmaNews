<img width="1919" height="1160" alt="image" src="https://github.com/user-attachments/assets/be7db1a9-5f23-47a6-b1f3-4eb7542a36b8" />


-- ============================================================
-- 01 - Set role and warehouse
-- ============================================================

USE ROLE SANDBOX_DEVELOPER;
USE WAREHOUSE SANDBOX_WH;


-- ============================================================
-- 02 - Create your own database
-- ============================================================

CREATE DATABASE IF NOT EXISTS PHARMA_NEWS_SANDBOX;


-- ============================================================
-- 03 - Create schema
-- ============================================================

CREATE SCHEMA IF NOT EXISTS PHARMA_NEWS_SANDBOX.NEWS;


-- ============================================================
-- 04 - Set context
-- ============================================================

USE DATABASE PHARMA_NEWS_SANDBOX;
USE SCHEMA NEWS;


-- ============================================================
-- 05 - Create staging table for Alteryx output
-- ============================================================

CREATE OR REPLACE TABLE PHARMA_NEWS_SANDBOX.NEWS.STG_PHARMA_NEWS_ARTICLES_V1 (
    message_id STRING,
    sender_name STRING,
    sender_email STRING,
    subject_raw STRING,
    received_ts STRING,
    email_source_type STRING,
    article_rank NUMBER,
    article_title STRING,
    article_url STRING,
    article_url_extraction_method STRING,
    body_best STRING,
    article_llm_input STRING,
    parser_version STRING
);


-- ============================================================
-- 06 - Check that it worked
-- ============================================================

SELECT
    CURRENT_ROLE() AS CURRENT_ROLE,
    CURRENT_WAREHOUSE() AS CURRENT_WAREHOUSE,
    CURRENT_DATABASE() AS CURRENT_DATABASE,
    CURRENT_SCHEMA() AS CURRENT_SCHEMA;
