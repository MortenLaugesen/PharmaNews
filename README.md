from ayx import Alteryx
import pandas as pd
import re

df = Alteryx.read("#1")

records = []

bad_terms = [
    "unsubscribe", "privacy", "preferences", "advertise", "contact support",
    "facebook", "linkedin", "twitter", "youtube", "webinar", "event",
    "register", "subscribe", "read in browser", "enterprise subscription",
    "brand logo", "linkedin logo", "facebook logo", "twitter logo", "youtube logo",
    "questex signature"
]

pattern = re.compile(r'([^<\n\r]{8,}?)<((?:https?|mailto):[^>]+)>', re.IGNORECASE)

for _, row in df.iterrows():
    message_id = str(row["message_id"]) if pd.notna(row["message_id"]) else ""
    sender_name = str(row["sender_name"]) if pd.notna(row["sender_name"]) else ""
    sender_email = str(row["sender_email"]) if pd.notna(row["sender_email"]) else ""
    subject_raw = str(row["subject_raw"]) if pd.notna(row["subject_raw"]) else ""
    received_ts = str(row["received_ts"]) if pd.notna(row["received_ts"]) else ""
    source = str(row["email_source_type"]) if pd.notna(row["email_source_type"]) else "OTHER"
    parser_version = str(row["parser_version"]) if pd.notna(row["parser_version"]) else "v1_plain_text_article_parser"
    body_best = str(row["body_best"]) if pd.notna(row["body_best"]) else ""

    matches = pattern.findall(body_best)

    seen = set()
    rank = 0

    for title, url in matches:
        article_title = " ".join(title.split()).strip()
        article_url = url.strip()

        title_lower = article_title.lower()
        url_lower = article_url.lower()

        # Basic filters
        if len(article_title) < 15:
            continue

        if any(term in title_lower for term in bad_terms):
            continue

        if any(term in url_lower for term in ["unsubscribe", "privacy", "facebook", "linkedin", "twitter", "youtube", "mailto:"]):
            continue

        dedupe_key = (message_id, article_title, article_url)
        if dedupe_key in seen:
            continue
        seen.add(dedupe_key)

        rank += 1

        if source == "FIERCE":
            extraction_method = "plain_text_tracking_link"
        elif source == "ENDPOINTS":
            extraction_method = "plain_text_href"
        else:
            extraction_method = "plain_text_generic"

        article_llm_input = (
            f"Sender: {sender_name} <{sender_email}>"
            f"\nSubject: {subject_raw}"
            f"\nReceived: {received_ts}"
            f"\nSource type: {source}"
            f"\nArticle rank in email: {rank}"
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
            "article_rank": str(rank),
            "article_title": article_title,
            "article_url": article_url,
            "article_url_extraction_method": extraction_method,
            "body_best": body_best,
            "article_llm_input": article_llm_input,
            "parser_version": parser_version
        })

out_df = pd.DataFrame(records)

expected_cols = [
    "message_id",
    "sender_name",
    "sender_email",
    "subject_raw",
    "received_ts",
    "email_source_type",
    "article_rank",
    "article_title",
    "article_url",
    "article_url_extraction_method",
    "body_best",
    "article_llm_input",
    "parser_version"
]

if out_df.empty:
    out_df = pd.DataFrame(columns=expected_cols)
else:
    out_df = out_df[expected_cols]

columns = {
    "message_id": {"type": "V_WString", "length": 255},
    "sender_name": {"type": "V_WString", "length": 255},
    "sender_email": {"type": "V_WString", "length": 255},
    "subject_raw": {"type": "V_WString", "length": 2000},
    "received_ts": {"type": "V_WString", "length": 100},
    "email_source_type": {"type": "V_WString", "length": 50},
    "article_rank": {"type": "V_WString", "length": 10},
    "article_title": {"type": "V_WString", "length": 2000},
    "article_url": {"type": "V_WString", "length": 4000},
    "article_url_extraction_method": {"type": "V_WString", "length": 100},
    "body_best": {"type": "V_WString", "length": 32000},
    "article_llm_input": {"type": "V_WString", "length": 32000},
    "parser_version": {"type": "V_WString", "length": 100}
}

Alteryx.write(out_df, 1, columns=columns)
