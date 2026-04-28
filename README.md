body_best

Type: V_WString

IF IsNull([body_raw]) OR IsEmpty([body_raw]) THEN [body_preview] ELSE [body_raw] ENDIF

email_source_type

Type: V_WString

IF Contains(LowerCase([sender_email]), "fierce") OR Contains(LowerCase([sender_name]), "fierce") THEN "FIERCE"
ELSEIF Contains(LowerCase([sender_email]), "endpoints") OR Contains(LowerCase([sender_name]), "endpoints") THEN "ENDPOINTS"
ELSE "OTHER"
ENDIF

parser_version

Type: V_WString

"v1_plain_text_article_parser"
