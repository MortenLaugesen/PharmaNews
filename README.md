from ayx import Alteryx
import pandas as pd
from bs4 import BeautifulSoup

df = Alteryx.read("#1")

debug_rows = []

for _, row in df.iterrows():
    body_raw = str(row["body_raw"]) if pd.notna(row["body_raw"]) else ""
    body_best = str(row["body_best"]) if pd.notna(row["body_best"]) else ""
    source = str(row["email_source_type"]) if pd.notna(row["email_source_type"]) else "OTHER"
    message_id = str(row["message_id"]) if pd.notna(row["message_id"]) else ""

    soup = BeautifulSoup(body_raw, "html.parser")
    links = soup.find_all("a", href=True)

    debug_rows.append({
        "message_id": message_id,
        "email_source_type": source,
        "body_raw_length": str(len(body_raw)),
        "body_best_length": str(len(body_best)),
        "num_links_found": str(len(links)),
        "body_raw_preview": body_raw[:500],
        "body_best_preview": body_best[:500]
    })

out_df = pd.DataFrame(debug_rows)

expected_cols = [
    "message_id",
    "email_source_type",
    "body_raw_length",
    "body_best_length",
    "num_links_found",
    "body_raw_preview",
    "body_best_preview"
]

if out_df.empty:
    out_df = pd.DataFrame(columns=expected_cols)
else:
    out_df = out_df[expected_cols]

columns = {
    "message_id": {"type": "V_WString", "length": 255},
    "email_source_type": {"type": "V_WString", "length": 50},
    "body_raw_length": {"type": "V_WString", "length": 20},
    "body_best_length": {"type": "V_WString", "length": 20},
    "num_links_found": {"type": "V_WString", "length": 20},
    "body_raw_preview": {"type": "V_WString", "length": 2000},
    "body_best_preview": {"type": "V_WString", "length": 2000}
}

Alteryx.write(out_df, 1, columns=columns)
