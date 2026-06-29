IF IsNull([ENRICHED_TEXT]) THEN 0 ELSE Length([ENRICHED_TEXT]) ENDIF

DateTimeNow()


IF [EXTRACTION_STATUS] = "success" THEN Null()
ELSE "Downloaded but marked as " + [EXTRACTION_STATUS]
ENDIF
<img width="2546" height="1347" alt="image" src="https://github.com/user-attachments/assets/4ca54238-683e-4a81-b829-ed4659033320" />

