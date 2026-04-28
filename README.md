from ayx import Alteryx
import pandas as pd
from bs4 import BeautifulSoup

df = Alteryx.read("#1")

records = []

for _, row in df.iterrows():
    html = str(row["body_raw"]) if pd.notna(row["body_raw"]) else ""
    source = str(row["email_source_type"]) if pd.notna(row["email_source_type"]) else "OTHER"
    message_id = str(row["message_id"]) if pd.notna(row["message_id"]) else ""

    soup = BeautifulSoup(html, "html.parser")

    rank = 0

    for a in soup.find_all("a", href=True):
        href = a.get("href", "").strip()
        title_attr = a.get("title", "").strip()
        link_text = a.get_text(" ", strip=True)

        if source == "FIERCE":
            article_url = title_attr if title_attr.startswith("http") else href
            extraction_method = "title" if title_attr.startswith("http") else "href_fallback"
        elif source == "ENDPOINTS":
            article_url = href
            extraction_method = "href"
        else:
            article_url = href
            extraction_method = "href_fallback"

        if not article_url.startswith("http"):
            continue

        rank += 1

        records.append({
            "message_id": str(message_id),
            "email_source_type": str(source),
            "article_rank": str(rank),
            "article_title": str(link_text),
            "article_url": str(article_url),
            "article_url_extraction_method": str(extraction_method)
        })

out_df = pd.DataFrame(records)

# Ensure columns always exist even if no rows are found
expected_cols = [
    "message_id",
    "email_source_type",
    "article_rank",
    "article_title",
    "article_url",
    "article_url_extraction_method"
]

if out_df.empty:
    out_df = pd.DataFrame(columns=expected_cols)
else:
    out_df = out_df[expected_cols]

# Explicit Alteryx metadata
columns = {
    "message_id": {"type": "V_WString", "length": 255},
    "email_source_type": {"type": "V_WString", "length": 50},
    "article_rank": {"type": "V_WString", "length": 10},
    "article_title": {"type": "V_WString", "length": 2000},
    "article_url": {"type": "V_WString", "length": 4000},
    "article_url_extraction_method": {"type": "V_WString", "length": 50}
}

Alteryx.write(out_df, 1, columns=columns)
