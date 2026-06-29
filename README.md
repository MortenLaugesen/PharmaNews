IF Length([ENRICHED_TEXT]) > 500 THEN "full_article"
ELSEIF Length([ENRICHED_TEXT]) > 100 THEN "partial_extract"
ELSE "snippet_only"
ENDIF


IF IsNull([DownloadData]) OR Length([DownloadData]) < 100 THEN "empty"
ELSEIF Contains(Lowercase([DownloadData]), "subscribe")
   OR Contains(Lowercase([DownloadData]), "sign in")
   OR Contains(Lowercase([DownloadData]), "log in")
   OR Contains(Lowercase([DownloadData]), "premium")
THEN "paywall"
ELSE "success"
ENDIF
