Clean failed extraction text

IF [EXTRACTION_STATUS] = "success" THEN Left([ENRICHED_TEXT], 64000)
ELSE Null()
ENDIF
