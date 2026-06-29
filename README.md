IF [EXTRACTION_STATUS] != "success" THEN "snippet_only"
ELSEIF Length([ENRICHED_TEXT]) > 500 THEN "full_article"
ELSEIF Length([ENRICHED_TEXT]) > 100 THEN "partial_extract"
ELSE "snippet_only"
ENDIF

IF [EXTRACTION_STATUS] = "success" THEN Null()
ELSE "Downloaded but marked as " + [EXTRACTION_STATUS] + ". Header: " + Left([DownloadHeaders], 100)
ENDIF


![Uploading image.png…]()
