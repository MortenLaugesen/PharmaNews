# DEBUG PARSER - CHECK HOW MANY TITLE<URL> MATCHES EXIST PER EMAIL

from ayx import Alteryx
import pandas as pd
import re

df = Alteryx.read("#1")

pattern = re.compile(r'([^<\n\r]{8,}?)<((?:https?|mailto):[^>]+)>', re.IGNORECASE)

debug_rows = []

for _, row in df.iterrows():
    message_id = str(row["message_id"]) if pd.notna(row["message_id"]) else ""
    source = str(row["email_source_type"]) if pd.notna(row["email_source_type"]) else "OTHER"
    body_text = str(row["body_best"]) if pd.notna(row["body_best"]) else ""

    matches = pattern.findall(body_text)

    first_title = ""
    first_url = ""

    if len(matches) > 0:
        first_title = " ".join(matches[0][0].split()).strip()
        first_url = matches[0][1].strip()

    debug_rows.append({
        "message_id": message_id,
        "email_source_type": source,
        "body_length": str(len(body_text)),
        "match_count": str(len(matches)),
        "first_title": first_title,
        "first_url": first_url
    })

out_df = pd.DataFrame(debug_rows)

expected_cols = [
    "message_id",
    "email_source_type",
    "body_length",
    "match_count",
    "first_title",
    "first_url"
]

if out_df.empty:
    out_df = pd.DataFrame(columns=expected_cols)
else:
    out_df = out_df[expected_cols]

columns = {
    "message_id": {"type": "V_WString", "length": 255},
    "email_source_type": {"type": "V_WString", "length": 50},
    "body_length": {"type": "V_WString", "length": 20},
    "match_count": {"type": "V_WString", "length": 20},
    "first_title": {"type": "V_WString", "length": 2000},
    "first_url": {"type": "V_WString", "length": 4000}
}

Alteryx.write(out_df, 1, columns=columns)
