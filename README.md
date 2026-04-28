# PharmaNews


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


Requirement already satisfied: beautifulsoup4 in c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages (4.12.3)
Requirement already satisfied: soupsieve>1.2 in c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages (from beautifulsoup4) (2.5)


from ayx import Alteryx 
import pandas as pd

df = Alteryx.read("#1")

print(df.columns.tolist())

out_df = df.head(5).copy()

Alteryx.write(out_df, 1)
from ayx import Alteryx 
import pandas as pd
​
df = Alteryx.read("#1")
​
print(df.columns.tolist())
​
out_df = df.head(5).copy()
​
Alteryx.write(out_df, 1)
Unable to connect to input data (C:\Users\dkmlaug2\AppData\Local\Temp\Engine_6160_92d3cf12f70043ca8e85632d62cc9579_\8990848d0e80417483a0ce4039dc6184\4460abb7be83bae8f01b9bf1238a923c.yxdb)
You must run the workflow first in order to make a cached copy of the incoming data available for development purposes within this Jupyter notebook.
---------------------------------------------------------------------------
FileNotFoundError                         Traceback (most recent call last)
Cell In[1], line 4
      1 from ayx import Alteryx 
      2 import pandas as pd
----> 4 df = Alteryx.read("#1")
      6 print(df.columns.tolist())
      8 out_df = df.head(5).copy()

File c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages\ayx\export.py:35, in read(incoming_connection_name, debug, **kwargs)
     31 def read(incoming_connection_name, debug=False, **kwargs):
     32     """
     33     When running the workflow in Alteryx, this function will convert incoming data streams to pandas dataframes when executing the code written in the Python tool. When called from the Jupyter notebook interactively, it will read in a copy of the incoming data that was cached on the previous run of the Alteryx workflow.
     34     """
---> 35     return __CachedData__(debug=debug).read(incoming_connection_name, **kwargs)

File c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages\ayx\CachedData.py:302, in CachedData.read(self, incoming_connection_name)
    298 input_data_filename = input_data_metadata["filename"]
    299 # create datafile object
    300 # (by not specifying the fileformat paramter, it will assume the file
    301 # type from the file's extension)
--> 302 with Datafile(input_data_filename, debug=self.debug) as db:
    303     msg_action = f'reading input data "{incoming_connection_name}"'
    304     try:
    305         # get the data from the sql db (if only one table exists, no need to specify the table name)

File c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages\ayx\Datafiles.py:157, in Datafile.__enter__(self)
    154 def __enter__(self):
    155     """Open the connection when a Datafile is created
    156     """
--> 157     self.openConnection()
    158     return self

File c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages\ayx\Datafiles.py:191, in Datafile.openConnection(self)
    189     print("Attempting to open connection to {}".format(self.filepath))
    190 try:
--> 191     self.connection = self.__returnConnection()
    192     if self.connection is None:
    193         del self.connection

File c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages\ayx\Datafiles.py:341, in Datafile.__returnConnection(self)
    337             raise PermissionError(
    338                 f"unable to write to filepath: {self.filepath}"
    339             )
    340         return None
--> 341 elif fileExists(
    342     self.filepath, throw_error=not (self.create_new), msg=error_msg
    343 ):
    344     # open connection and attempt to read one row
    345     # to confirm that it is a valid file
    346     connection = pyxdb.AlteryxYXDB()
    347     # test that its a real yxdb file

File c:\program files\alteryx\bin\miniconda3\envs\designerbasetools_venv\lib\site-packages\ayx\helpers.py:100, in fileExists(filepath, throw_error, msg, debug)
     98 else:
     99     if throw_error:
--> 100         raise FileNotFoundError(fileErrorMsg(msg, filepath))
    101     elif debug:
    102         print(fileErrorMsg(msg, filepath))

FileNotFoundError: Unable to connect to input data (C:\Users\dkmlaug2\AppData\Local\Temp\Engine_6160_92d3cf12f70043ca8e85632d62cc9579_\8990848d0e80417483a0ce4039dc6184\4460abb7be83bae8f01b9bf1238a923c.yxdb)

Run `Alteryx.help()` for info about useful functions.  
i.e., `Alteryx.read("#1")`, `Alteryx.write(df,1)`, `Alteryx.getWorkflowConstant("Engine.WorkflowDirectory")`

# List all non-standard packages to be imported by your 
# script here (only missing packages will be installed)
from ayx import Package
#Package.installPackages(['pandas','numpy'])

from ayx import Alteryx 
import pandas as pd

df = Alteryx.read("#1")

print(df.columns.tolist())

out_df = df.head(5).copy()

Alteryx.write(out_df, 1)



from ayx import Alteryx
import pandas as pd

df = Alteryx.read("#1")

print(df.columns.tolist())

out_df = df.head(5).copy()

Alteryx.write(out_df, 1)
<img width="1919" height="1199" alt="image" src="https://github.com/user-attachments/assets/dea7a626-04e3-43e6-96b6-3cc79f7c8f7b" />

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
