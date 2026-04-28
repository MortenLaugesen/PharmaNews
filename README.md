from ayx import Alteryx
import pandas as pd
from bs4 import BeautifulSoup

df = Alteryx.read("#1")

records = []

for _, row in df.iterrows():
    html = str(row["body_raw"]) if pd.notna(row["body_raw"]) else ""
    source = str(row["email_source_type"]) if pd.notna(row["email_source_type"]) else "OTHER"
    message_id = row["message_id"]

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
            "message_id": message_id,
            "email_source_type": source,
            "article_rank": rank,
            "article_title": link_text,
            "article_url": article_url,
            "article_url_extraction_method": extraction_method
        })

out_df = pd.DataFrame(records)

Alteryx.write(out_df, 1)
