# PharmaNews
<img width="895" height="858" alt="image" src="https://github.com/user-attachments/assets/a5bf9037-59c0-4326-9b90-5f6828e8f3e1" />

<img width="1918" height="869" alt="image" src="https://github.com/user-attachments/assets/9e20eed7-1bbd-4322-a676-28b6c53ce6f7" />


body_best
IF IsNull([body_raw]) OR IsEmpty([body_raw]) THEN [body_preview] ELSE [body_raw] ENDIF

email_source_type
IF Contains(LowerCase([sender_email]), "fierce") OR Contains(LowerCase([sender_name]), "fierce") THEN "FIERCE"
ELSEIF Contains(LowerCase([sender_email]), "endpoints") OR Contains(LowerCase([sender_name]), "endpoints") THEN "ENDPOINTS"
ELSE "OTHER"
ENDIF

parser_version
"v2_article_level_parser"

Python tool
# Alteryx Python Tool - Article-level parser
# Input anchor 1 must contain:
# message_id, sender_name, sender_email, subject_raw, received_ts, body_raw, body_best, email_source_type, parser_version

from ayx import Alteryx
import pandas as pd
from bs4 import BeautifulSoup
from html import unescape

df = Alteryx.read("#1")

def clean_text(x):
    if pd.isna(x):
        return ""
    return " ".join(str(x).split()).strip()

def detect_source(sender_email, sender_name, existing_source):
    if pd.notna(existing_source) and str(existing_source).strip() != "":
        return str(existing_source).strip().upper()
    s_email = str(sender_email).lower() if pd.notna(sender_email) else ""
    s_name = str(sender_name).lower() if pd.notna(sender_name) else ""
    if "fierce" in s_email or "fierce" in s_name:
        return "FIERCE"
    if "endpoints" in s_email or "endpoints" in s_name:
        return "ENDPOINTS"
    return "OTHER"

def should_keep_link(url, title, source):
    url_l = url.lower()
    title_l = title.lower()

    bad_terms = [
        "unsubscribe", "privacy", "preferences", "advertise", "sponsor",
        "webinar", "podcast", "event", "register", "login", "subscribe",
        "facebook", "linkedin", "instagram", "twitter", "youtube"
    ]

    if not url.startswith("http"):
        return False

    if any(term in url_l for term in bad_terms):
        return False

    if any(term in title_l for term in bad_terms):
        return False

    if len(title.strip()) < 20:
        return False

    if source == "FIERCE":
        # For Fierce, the useful article URL is usually in title=, while href is qtx tracking
        return ("fierce" in url_l) or ("qtx.omeclk.com" in url_l)

    if source == "ENDPOINTS":
        # For Endpoints, href is typically the right link/redirect
        return "endpointsnews.com" in url_l

    return True

records = []

for _, row in df.iterrows():
    message_id = row.get("message_id", "")
    sender_name = row.get("sender_name", "")
    sender_email = row.get("sender_email", "")
    subject_raw = row.get("subject_raw", "")
    received_ts = row.get("received_ts", "")
    body_raw = row.get("body_raw", "")
    body_best = row.get("body_best", "")
    parser_version = row.get("parser_version", "v2_article_level_parser")

    source = detect_source(sender_email, sender_name, row.get("email_source_type", ""))

    html = body_raw if pd.notna(body_raw) and str(body_raw).strip() != "" else body_best
    html = "" if pd.isna(html) else str(html)

    soup = BeautifulSoup(html, "html.parser")

    seen = set()
    article_rank = 0

    for a in soup.find_all("a", href=True):
        href = unescape(a.get("href", "")).strip()
        title_attr = unescape(a.get("title", "")).strip()
        link_text = clean_text(a.get_text(" ", strip=True))

        if source == "FIERCE":
            article_url = title_attr if title_attr.startswith("http") else href
            extraction_method = "title" if title_attr.startswith("http") else "href_fallback"
            article_title = link_text if link_text else title_attr

        elif source == "ENDPOINTS":
            article_url = href
            extraction_method = "href"
            article_title = link_text if link_text else title_attr

        else:
            article_url = title_attr if title_attr.startswith("http") else href
            extraction_method = "title_fallback" if title_attr.startswith("http") else "href_fallback"
            article_title = link_text if link_text else title_attr

        article_title = clean_text(article_title)

        if not should_keep_link(article_url, article_title, source):
            continue

        dedupe_key = (message_id, article_url, article_title)
        if dedupe_key in seen:
            continue
        seen.add(dedupe_key)

        article_rank += 1

        article_llm_input = (
            f"Sender: {sender_name} <{sender_email}>"
            f"\nSubject: {subject_raw}"
            f"\nReceived: {received_ts}"
            f"\nSource type: {source}"
            f"\nArticle rank in email: {article_rank}"
            f"\nArticle title: {article_title}"
            f"\nArticle URL: {article_url}"
            f"\nEmail body:\n{body_best}"
        )

        records.append({
            "message_id": message_id,
            "sender_name": sender_name,
            "sender_email": sender_email,
            "subject_raw": subject_raw,
            "received_ts": received_ts,
            "email_source_type": source,
            "article_rank": article_rank,
            "article_title": article_title,
            "article_url": article_url,
            "article_url_extraction_method": extraction_method,
            "body_best": body_best,
            "article_llm_input": article_llm_input,
            "parser_version": parser_version
        })

out_df = pd.DataFrame(records)

Alteryx.write(out_df, 1)



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
