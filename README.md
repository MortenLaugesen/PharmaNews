IF IsNull([ENRICHED_TEXT]) THEN 0 ELSE Length([ENRICHED_TEXT]) ENDIF

DateTimeNow()


IF [EXTRACTION_STATUS] = "success" THEN Null()
ELSE "Downloaded but marked as " + [EXTRACTION_STATUS]
ENDIF
