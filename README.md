IF [EXTRACTION_STATUS] != "success" THEN "snippet_only"
ELSEIF Length([ENRICHED_TEXT]) > 500 THEN "full_article"
ELSEIF Length([ENRICHED_TEXT]) > 100 THEN "partial_extract"
ELSE "snippet_only"
ENDIF

IF [EXTRACTION_STATUS] = "success" THEN Null()
ELSE "Downloaded but marked as " + [EXTRACTION_STATUS] + ". Header: " + Left([DownloadHeaders], 100)
ENDIF

<img width="2559" height="1261" alt="image" src="https://github.com/user-attachments/assets/840a3b9b-2d1b-435a-ab2d-c11f5cd08e12" />

