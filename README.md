# PharmaNews

<img width="1918" height="930" alt="image" src="https://github.com/user-attachments/assets/916093b2-088e-43f3-9cd2-9ff38f319474" />

body_best
IF IsNull([body_raw]) OR IsEmpty([body_raw]) THEN [body_preview] ELSE [body_raw] ENDIF

llm_input
"Sender: " + [sender_name] + " <" + [sender_email] + ">" +
"\nSubject: " + [subject_raw] +
"\nReceived: " + [received_ts] +
"\nBody:\n" + [body_best]

parser_version
"v1_fierce_endpoints"

email_source_type
IF Contains(LowerCase([sender_email]), "fierce") OR Contains(LowerCase([sender_name]), "fierce") THEN "FIERCE"
ELSEIF Contains(LowerCase([sender_email]), "endpoints") OR Contains(LowerCase([sender_name]), "endpoints") THEN "ENDPOINTS"
ELSE "OTHER"
ENDIF

article_url_from_title
IF REGEX_CountMatches([body_raw], 'title="https?://[^"]+"') > 0 THEN
    Replace(
        REGEX_Replace(
            [body_raw],
            '(?is).*?title="(https?://[^"]+)".*',
            '$1'
        ),
        '&amp;',
        '&'
    )
ELSE
    ""
ENDIF


article_url_from_href
IF REGEX_CountMatches([body_raw], 'href="https?://[^"]+"') > 0 THEN
    Replace(
        REGEX_Replace(
            [body_raw],
            '(?is).*?href="(https?://[^"]+)".*',
            '$1'
        ),
        '&amp;',
        '&'
    )
ELSE
    ""
ENDIF

article_url
IF [email_source_type] = "FIERCE" AND NOT IsEmpty([article_url_from_title]) THEN [article_url_from_title]
ELSEIF [email_source_type] = "ENDPOINTS" AND NOT IsEmpty([article_url_from_href]) THEN [article_url_from_href]
ELSEIF NOT IsEmpty([article_url_from_title]) THEN [article_url_from_title]
ELSE [article_url_from_href]
ENDIF

article_url_extraction_method
IF [email_source_type] = "FIERCE" AND NOT IsEmpty([article_url_from_title]) THEN "title"
ELSEIF [email_source_type] = "ENDPOINTS" AND NOT IsEmpty([article_url_from_href]) THEN "href"
ELSEIF NOT IsEmpty([article_url_from_title]) THEN "title_fallback"
ELSEIF NOT IsEmpty([article_url_from_href]) THEN "href_fallback"
ELSE "none"
ENDIF








<img width="1919" height="946" alt="image" src="https://github.com/user-attachments/assets/038b8cfc-e3c5-405e-9d5c-ab477aea6ebd" />


hey
frank
<img width="1898" height="817" alt="image" src="https://github.com/user-attachments/assets/f206d114-a13a-47e9-8da3-04f7f41b3458" />
<img width="1919" height="901" alt="image" src="https://github.com/user-attachments/assets/64a29ca0-d412-492e-9fbb-a926afb360fa" />
<img width="698" height="838" alt="image" src="https://github.com/user-attachments/assets/88171881-486a-4857-81d5-8a860d251193" />
<img width="1908" height="871" alt="image" src="https://github.com/user-attachments/assets/3a12babe-7706-4f50-8b62-88c60890f076" />
