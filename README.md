
-- Check V2 queue without date filter
SELECT
    PRIORITY_TIER,
    COUNT(*) AS CNT
FROM BI.NEWS.V_PHARMA_NEWS_DIGEST_QUEUE_V2
GROUP BY PRIORITY_TIER
ORDER BY CNT DESC;

<img width="1411" height="515" alt="image" src="https://github.com/user-attachments/assets/ca8799fd-ec7d-4dc7-a305-dbf501ac1109" />
