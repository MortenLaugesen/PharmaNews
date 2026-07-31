

USE DATABASE PHARMA_NEWS_SANDBOX;
USE SCHEMA NEWS;

SELECT COUNT(*) AS CURRENT_ROW_COUNT
FROM STG_PHARMA_NEWS_ARTICLES_V1;

SELECT *
FROM STG_PHARMA_NEWS_ARTICLES_V1
LIMIT 10;

SHOW COLUMNS IN TABLE STG_PHARMA_NEWS_ARTICLES_V1;



<img width="976" height="197" alt="image" src="https://github.com/user-attachments/assets/1b0eac6e-2925-4368-827e-6ddc9726cd8e" />


https://fujifilm0.sharepoint.com/:f:/r/teams/msg-FDBD-BusinessIntelligenceandInsights/Shared%20Documents/1.%20Alteryx%20%26%20Tableau%20Works/Tableau%20dashboards?csf=1&web=1&e=zHMWxU

Please use the below list to create your documentations…

1.	WHAT DOES THIS SCRIPT DO? – Script name, database/schema, plain description
2.	DATA SOURCES – Table names, Systems/Excel Files details, read/write, what they contain
3.	HOW OFTEN SHOULD THIS RUN? – Frequency and timing
4.	STEP-BY-STEP BREAKDOWN 
I.	Actual script
II.	Notes
5.	WHO NEEDS THIS & WHY? – Requester, users, business reason
6.	THINGS TO WATCH OUT FOR – Warnings/Assumptions/Restrictions
7.	WHO TO ASK FOR HELP – Primary and backup contacts


Task documentation
Please create and share documentations for the Pharma News Task as well as your workstream in the Scale Analysis – BDO report reverse engineering project.
If any files are saved on SharePoint/Teams, add the location to the document. If not, save your files on SharePoint/Teams and include the location in the document.


<img width="868" height="878" alt="image" src="https://github.com/user-attachments/assets/50ae454d-8558-4650-9b74-6d385b2d5056" />



IF [METRIC] = "% Utilization" THEN

    IF CONTAINS([Value], "%") THEN
        FLOAT(REPLACE(REPLACE([Value], "%", ""), ",", ".")) / 100
    ELSE
        FLOAT(REPLACE([Value], ",", "."))
    END

ELSE

    IF CONTAINS([Value], ",") AND CONTAINS([Value], ".") THEN
        FLOAT(REPLACE(REPLACE([Value], ".", ""), ",", "."))
    ELSEIF CONTAINS([Value], ",") THEN
        FLOAT(REPLACE([Value], ",", "."))
    ELSE
        FLOAT(REPLACE([Value], ".", ""))
    END

END

<img width="1262" height="604" alt="image" src="https://github.com/user-attachments/assets/0e094ce4-b37c-4c59-8e50-b1c4686315d7" />




SUM(
    IF [METRIC] = "Capacity" THEN [C. Value Numeric] END
)
-
SUM(
    IF [METRIC] = "Required" THEN [C. Value Numeric] END
)

<img width="2536" height="1383" alt="image" src="https://github.com/user-attachments/assets/e7224efc-52ba-4f88-bcc1-13958429182c" />

IF CONTAINS([Value], "%") THEN
    FLOAT(REPLACE(REPLACE(REPLACE([Value], "%", ""), ".", ""), ",", ".")) / 100
ELSEIF CONTAINS([Value], ",") THEN
    FLOAT(REPLACE(REPLACE([Value], ".", ""), ",", "."))
ELSE
    FLOAT(REPLACE([Value], ".", ""))
END

------------------------------------------------------------------

Transcript. Use arrow keys to navigate between transcript entries.


Search

AI-generated content may be incorrect

Upasana Paul started transcription

Upasana Paul
0 minutes 3 seconds0:03
Upasana Paul 0 minutes 3 seconds
Yeah.

John Horn
0 minutes 3 seconds0:03
John Horn 0 minutes 3 seconds
Mia, welcome back.
MK

Mia Knudsen
0 minutes 5 seconds0:05
Mia Knudsen 0 minutes 5 seconds
Thanks.
Mia Knudsen 0 minutes 6 seconds
Okay.
Mia Knudsen 0 minutes 7 seconds
I'm starting off with writing a very angry e-mail to Sideline.

Upasana Paul
0 minutes 12 seconds0:12
Upasana Paul 0 minutes 12 seconds
Yeah.
Upasana Paul 0 minutes 13 seconds
Yeah.

John Horn
0 minutes 13 seconds0:13
John Horn 0 minutes 13 seconds
Ohh, man.
John Horn 0 minutes 17 seconds
Um...
John Horn 0 minutes 18 seconds
I have to drop at 8:30 to speak to Christiane here, but...
John Horn 0 minutes 24 seconds
What we were gonna talk about, Mia, just for...
John Horn 0 minutes 28 seconds
Since you were out.
MK
Mia Knudsen
0 minutes 28 seconds0:28
Mia Knudsen 0 minutes 28 seconds
I think...

John Horn
0 minutes 30 seconds0:30
John Horn 0 minutes 30 seconds
Oh yeah, go ahead.
MK
Mia Knudsen
0 minutes 30 seconds0:30
Mia Knudsen 0 minutes 30 seconds
Yeah, no, just good.

John Horn
0 minutes 33 seconds0:33
John Horn 0 minutes 33 seconds
We're just going to take a look at the data foundations stuff. So I don't know if you saw that or not, but.
MK
Mia Knudsen
0 minutes 37 seconds0:37
Mia Knudsen 0 minutes 37 seconds
Yeah.
Mia Knudsen 0 minutes 39 seconds
Yeah.
Mia Knudsen 0 minutes 41 seconds
Tobi had some has requested to be off, so she's not here, but she did message Upasana and Remi to just walk you through what they have so far.

John Horn
0 minutes 48 seconds0:48
John Horn 0 minutes 48 seconds
Oh.
John Horn 0 minutes 54 seconds
Okay, that works too.

Upasana Paul
0 minutes 55 seconds0:55
Upasana Paul 0 minutes 55 seconds
Yeah.
Upasana Paul 0 minutes 56 seconds
Yeah, so she told us to utilize this meeting for the video reverse engineering one. So where we are standing, what is expected. So basically, we are almost done with the pipeline for our view. And what we are doing right now, we are actually using two main
Upasana Paul 1 minute 20 seconds
table from Snowflake called BDO Capacity Master and BDO Demand Master. And it seems like it has a 2024 data, not the 2025 in that table. So there is a bit, a little gap between what we have, the count we have in the FDB MAM Excel and what we are calculating.
Upasana Paul 1 minute 41 seconds
This is not a very big difference, because the column we are taking, there are some differences definitely because we are using the phase column from the video demand master and we are getting fewer phase from Snowflake video data what we have.
Upasana Paul 2 minutes 1 second
And let me share my screen.
Upasana Paul 2 minutes 6 seconds
The.
Upasana Paul 2 minutes 8 seconds
So we will have a whole pipeline. There will be many more table in the process. So if someone wants to dig more into detail, so if you want to take a company-wise, if you want to get that data year-wise, everything will be there.
Upasana Paul 2 minutes 28 seconds
But this is the final table we have decided to have for Morten so that it is just a, it is a full customized for the dashboard we are making so that he doesn't need to do the aggregation. So this is an aggregated table what you are looking at.
Upasana Paul 2 minutes 47 seconds
at. So we are going to have the data for a different region for different scale. And these are different metrics and we have the whole equation for each and every matrix. And these are the data year-wise.
Upasana Paul 3 minutes 10 seconds
What we are doing right now, we are actually checking the data quality, what we are calculating here, and what we have in the FDBD. So it seems like it's almost same, but for EHS, I saw that the utilization is a bit for...
Upasana Paul 3 minutes 30 seconds
Asia all capacity, I saw that the utilization is a bit lower than what we have in the FDB. So that's what I'm going through. And for the, is Morten there in the call?
RD
Remi Defina-Sperando
3 minutes 47 seconds3:47
Remi Defina-Sperando 3 minutes 47 seconds
Not yet. That's fine. You can show the, he sent us the visualizations he did earlier. You can share those.

John Horn
3 minutes 48 seconds3:48
John Horn 3 minutes 48 seconds
No.

Upasana Paul
3 minutes 49 seconds3:49
Upasana Paul 3 minutes 49 seconds
Out.
Upasana Paul 3 minutes 51 seconds
Yes.
Upasana Paul 3 minutes 54 seconds
Is.
Upasana Paul 3 minutes 56 seconds
Okay, so the visualize one of the visualization will be this.
Upasana Paul 4 minutes 4 seconds
ABS.

John Horn
4 minutes 4 seconds4:04
John Horn 4 minutes 4 seconds
Okay.
John Horn 4 minutes 8 seconds
Global utilization across all capacity scales.

Upasana Paul
4 minutes 13 seconds4:13
Upasana Paul 4 minutes 13 seconds
Yeah.

John Horn
4 minutes 13 seconds4:13
John Horn 4 minutes 13 seconds
is, yeah, 65 going to greater than 100% by 2029.
RD
Remi Defina-Sperando
7 minutes 48 seconds7:48
Remi Defina-Sperando 7 minutes 48 seconds
Yeah. And so for the large scale, what we can add is video has a metric called more than 5 kiloliters, and that would be what you can consider large scale. So that should be a tab that will just come in.
ML
Morten Laugesen
7 minutes 48 seconds7:48
Morten Laugesen 7 minutes 48 seconds
Mhm.

John Horn
7 minutes 56 seconds7:56
John Horn 7 minutes 56 seconds
Yeah.
RD
Remi Defina-Sperando
8 minutes 2 seconds8:02
Remi Defina-Sperando 8 minutes 2 seconds
Um, we counted that in pretty easy.

John Horn
8 minutes 2 seconds8:02
John Horn 8 minutes 2 seconds
Yeah.
ML
Morten Laugesen
8 minutes 5 seconds8:05
Morten Laugesen 8 minutes 5 seconds
Yeah.

John Horn
8 minutes 6 seconds8:06
John Horn 8 minutes 6 seconds
And what I never...
MK
Mia Knudsen
8 minutes 6 seconds8:06
Mia Knudsen 8 minutes 6 seconds
I just started recording, just so you know, so we capture what the what you think on.

IF CONTAINS([Value], "%") THEN
    FLOAT(REPLACE(REPLACE(REPLACE([Value], "%", ""), ".", ""), ",", ".")) / 100
ELSEIF CONTAINS([Value], ",") THEN
    FLOAT(REPLACE(REPLACE([Value], ".", ""), ",", "."))
ELSE
    FLOAT(REPLACE([Value], ".", ""))
END

<img width="1910" height="1133" alt="image" src="https://github.com/user-attachments/assets/e4466604-06be-4052-825e-1aa1419fd622" />


<img width="1915" height="1076" alt="image" src="https://github.com/user-attachments/assets/ae229e6e-d417-44c7-b941-5faf758b7d0f" />


<img width="1916" height="1105" alt="image" src="https://github.com/user-attachments/assets/6f498af6-72fa-4f90-933b-eba130d455f2" />

<img width="1919" height="1141" alt="image" src="https://github.com/user-attachments/assets/0967955d-ae67-4581-9492-9058a44b125e" />


<img width="1919" height="1136" alt="image" src="https://github.com/user-attachments/assets/c6f42e0d-6f61-455e-9880-73bfc20516e1" />



<img width="1919" height="1169" alt="image" src="https://github.com/user-attachments/assets/85a6f6ae-d145-439c-a9d7-b5fe0a3713e1" />

<img width="1915" height="1140" alt="image" src="https://github.com/user-attachments/assets/fcf2cc07-fa4a-4d32-a069-0166d5bf2fad" />

<img width="1914" height="1124" alt="image" src="https://github.com/user-attachments/assets/b8135d9c-6475-4beb-be30-ad593265329a" />


<img width="1907" height="1074" alt="image" src="https://github.com/user-attachments/assets/b75e8869-1564-4a9e-812d-eacc291a8afb" />

<img width="1919" height="999" alt="image" src="https://github.com/user-attachments/assets/3b114474-d8fd-4201-be7d-1a1e28bbffe0" />

[BDO1.1_reverse.xlsm](https://github.com/user-attachments/files/30412231/BDO1.1_reverse.xlsm)


[Uploading BDO reverse eng_2026-07-27-1209.csv…]()


[BDO1.1_demand.xlsm](https://github.com/user-attachments/files/30406818/BDO1.1_demand.xlsm)

[BDO1.1_supply.xlsm](https://github.com/user-attachments/files/30406814/BDO1.1_supply.xlsm)



https://www.cnbc.com/2026/07/23/rheinmetall-gunpowder-ammunition-europe-rearm-ceo.html

<img width="471" height="428" alt="image" src="https://github.com/user-attachments/assets/a04643ae-b268-4236-9cf2-16d3661fbb86" />

<img width="498" height="435" alt="image" src="https://github.com/user-attachments/assets/09de89e8-66b2-4073-8147-cc556dc1f928" />

zs86899.west-europe.azure.snowflakecomputing.com

[{"host":"zs86899.west-europe.azure.snowflakecomputing.com","port":443,"type":"SNOWFLAKE_DEPLOYMENT"},{"host":"datahub-business_intelligence_insights.snowflakecomputing.com","port":443,"type":"SNOWFLAKE_DEPLOYMENT_REGIONLESS"},{"host":"3efb11sfcb1stg.blob.core.windows.net","port":443,"type":"STAGE"},{"host":"sfc-repo.snowflakecomputing.com","port":443,"type":"SNOWSQL_REPO"},{"host":"client-telemetry.snowflakecomputing.com","port":443,"type":"OUT_OF_BAND_TELEMETRY"},{"host":"ocsp.snowflakecomputing.com","port":80,"type":"OCSP_CACHE"},{"host":"api-00b5905e.duosecurity.com","port":443,"type":"DUO_SECURITY"},{"host":"uw2.devicemanagement.duosecurity.com","port":443,"type":"DUO_SECURITY"},{"host":"ec1.devicemanagement.duosecurity.com","port":443,"type":"DUO_SECURITY"},{"host":"snowconvert.snowflake.com","port":443,"type":"SNOWCONVERT"},{"host":"DATAHUB-BUSINESS-INTELLIGENCE-INSIGHTS.registry.snowflakecomputing.com","port":443,"type":"SPCS_REGISTRY_REGIONLESS"},{"host":"DATAHUB-BUSINESS_INTELLIGENCE_INSIGHTS.registry.snowflakecomputing.com","port":443,"type":"SPCS_REGISTRY_REGIONLESS"},{"host":"ZS86899.snowpark.amszqu.snowflakecomputing.com","port":443,"type":"SNOWPARK_CONNECT"},{"host":"ocsp.digicert.com","port":80,"type":"OCSP_RESPONDER"},{"host":"ocsp.sca1b.amazontrust.com","port":80,"type":"OCSP_RESPONDER"},{"host":"crl3.digicert.com","port":80,"type":"CRL_DISTRIBUTION_POINT"},{"host":"ocsp.r2m01.amazontrust.com","port":80,"type":"OCSP_RESPONDER"},{"host":"ocsp.r2m03.amazontrust.com","port":80,"type":"OCSP_RESPONDER"},{"host":"ocsp.rootca1.amazontrust.com","port":80,"type":"OCSP_RESPONDER"},{"host":"www.microsoft.com","port":80,"type":"CRL_DISTRIBUTION_POINT"},{"host":"ocsp.r2m02.amazontrust.com","port":80,"type":"OCSP_RESPONDER"},{"host":"ocsp.rootg2.amazontrust.com","port":80,"type":"OCSP_RESPONDER"},{"host":"ocsp.r2m04.amazontrust.com","port":80,"type":"OCSP_RESPONDER"},{"host":"oneocsp.microsoft.com","port":80,"type":"OCSP_RESPONDER"},{"host":"crl4.digicert.com","port":80,"type":"CRL_DISTRIBUTION_POINT"},{"host":"crl2.microsoft.com","port":80,"type":"CRL_DISTRIBUTION_POINT"},{"host":"app.snowflake.com","port":443,"type":"SNOWSIGHT_DEPLOYMENT"},{"host":"apps-api.c1.westeurope.azure.app.snowflake.com","port":443,"type":"SNOWSIGHT_DEPLOYMENT"},{"host":"datahub-business-intelligence-insights.openflow.amszqu.snowflakecomputing.com","port":443,"type":"OPENFLOW"},{"host":"datahub-business-intelligence-insights.telemetry.amszqu.snowflakecomputing.com","port":443,"type":"EXTERNAL_TELEMETRY"}]

<img width="939" height="695" alt="image" src="https://github.com/user-attachments/assets/61338219-7821-48a3-9d87-87be3b91fe8a" />

<img width="812" height="693" alt="image" src="https://github.com/user-attachments/assets/355359e3-2885-435c-9430-04f7f9c16e59" />

DATAHUB-BUSINESS_INTELLIGENCE_INSIGHTS.snowflakecomputing.com
<img width="2548" height="1337" alt="image" src="https://github.com/user-attachments/assets/d4fe1444-f45d-4e60-8f72-1a7581c4d323" />

<img width="1666" height="1145" alt="image" src="https://github.com/user-attachments/assets/06a296fe-f3ce-45ed-b506-0617f1ca549b" />


<img width="2540" height="1435" alt="image" src="https://github.com/user-attachments/assets/7529fb80-65d8-4241-b8f1-775b6b4b8166" />

<img width="1193" height="663" alt="image" src="https://github.com/user-attachments/assets/7241bb03-6550-4fbf-a4bf-003ddfce9bec" />


<img width="1933" height="1127" alt="image" src="https://github.com/user-attachments/assets/d9c30a8c-64e2-44ee-9379-41b8e5a44c7b" />

<img width="2542" height="1402" alt="image" src="https://github.com/user-attachments/assets/2b3aa7f5-ff60-494e-bfe0-6f1217ffa9d7" />

<img width="2531" height="1362" alt="image" src="https://github.com/user-attachments/assets/160ec731-3fc8-4fea-8b5f-09aa9f6c47f6" />


<img width="2545" height="1432" alt="image" src="https://github.com/user-attachments/assets/83e28fd9-e963-4692-bd45-93b1198776e7" />

<img width="1628" height="919" alt="image" src="https://github.com/user-attachments/assets/8c814d03-e543-4d6d-9723-2e86a0d3cc63" />

<img width="987" height="917" alt="image" src="https://github.com/user-attachments/assets/5aa85fdd-27e6-499a-ab9e-1b4f1eaf4c88" />

<img width="1646" height="951" alt="image" src="https://github.com/user-attachments/assets/d8c6b6c7-6dd1-443d-bb5e-7158e59bef93" />




<img width="1776" height="1097" alt="image" src="https://github.com/user-attachments/assets/cfe24faa-de58-4152-9a34-9111549faed7" />


Role: REPORTING_READER
Warehouse: SANDBOX_WH
Database: REPORTING
Schema: PUBLIC


SELECT
    CURRENT_USER(),
    CURRENT_ROLE(),
    CURRENT_WAREHOUSE(),
    CURRENT_DATABASE(),
    CURRENT_SCHEMA();


SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_DATABASE(), CURRENT_SCHEMA();
<img width="1242" height="856" alt="image" src="https://github.com/user-attachments/assets/f342f66b-4d41-4f80-a56f-cd1a023727ea" />


<img width="1464" height="913" alt="image" src="https://github.com/user-attachments/assets/375d0d07-1644-4eaf-89ef-2813c9db6399" />

<img width="737" height="544" alt="image" src="https://github.com/user-attachments/assets/5cb3145c-ce3c-4da5-83b3-efa4c50abdda" />


An error occurred while communicating with data source 'Untitled Data Source'
Authentication failed.
Error Code: 14333AEC
Snowflake URL validation failed. For more information see: https://kb.tableau.com/articles/Issue/error-validation-failed-for-your-input-oauth-custom-domain-when-connecting-to-snowflake

<img width="2543" height="1370" alt="image" src="https://github.com/user-attachments/assets/b8165d55-afdf-4ab0-a6bc-99b90103f1b9" />


<img width="521" height="1108" alt="image" src="https://github.com/user-attachments/assets/ba03a41f-2fd9-4b2a-83ec-d5b3309d134f" />



<img width="2540" height="1253" alt="image" src="https://github.com/user-attachments/assets/ae74f875-822d-4b34-a86d-e653d1baa86d" />


IF [PROJECTED_BIOREACTOR_FIT_VOLUME] < 2000 THEN "Small Scale"
ELSEIF [PROJECTED_BIOREACTOR_FIT_VOLUME] <= 5000 THEN "Mid Scale"
ELSE "Large Scale"
END

<img width="2502" height="1349" alt="image" src="https://github.com/user-attachments/assets/946c18f8-db2d-4c7e-9384-72ea47dcbd64" />

<img width="2552" height="1317" alt="image" src="https://github.com/user-attachments/assets/72cb486b-ffe6-4966-b44f-3d733d8d2e9c" />



<img width="1475" height="864" alt="image" src="https://github.com/user-attachments/assets/842d9daf-d804-4a56-8a8c-1b3ec4fab4f6" />

<img width="1491" height="876" alt="image" src="https://github.com/user-attachments/assets/3dd96fe4-be58-49f4-b7fd-690dfa3e0919" />

<img width="1489" height="875" alt="image" src="https://github.com/user-attachments/assets/a41146fe-6997-4e9b-b521-67cb171d5275" />


![Uploading image.png…]()




<img width="2556" height="1407" alt="image" src="https://github.com/user-attachments/assets/938b830e-0cc3-4454-be56-d4bea393aa32" />


<img width="2523" height="1366" alt="image" src="https://github.com/user-attachments/assets/17a89bde-a261-4c58-b40e-e615082bed9e" />


<img width="2519" height="1384" alt="image" src="https://github.com/user-attachments/assets/1681a2b3-718e-4bd9-b4a7-a533b4ba2be6" />

[BDO_Tableau_Long_Output.csv](https://github.com/user-attachments/files/30116840/BDO_Tableau_Long_Output.csv)

[Uploading BDO_Tableau_Long_Output.csv…]()

<img width="2540" height="1373" alt="image" src="https://github.com/user-attachments/assets/12423adb-4247-4bda-aa00-7c31a4e4f50a" />

<img width="2077" height="1200" alt="image" src="https://github.com/user-attachments/assets/bd33657d-9185-434b-b375-3dd3749dde47" />
<img width="2548" height="1380" alt="image" src="https://github.com/user-attachments/assets/c3bf363d-e615-4b81-817c-203a670fc77e" />


[BDO.Reverse.Eng.-.Data.Sample.1.1.xlsx](https://github.com/user-attachments/files/30113823/BDO.Reverse.Eng.-.Data.Sample.1.1.xlsx)
[BDO_demo_3tabs_supply_demand_joined (1).xlsx](https://github.com/user-attachments/files/30113710/BDO_demo_3tabs_supply_demand_joined.1.xlsx)

<img width="1083" height="434" alt="image" src="https://github.com/user-attachments/assets/c20dda72-e6ed-4d11-98ac-28f15a152078" />
[BDO Reverse Eng - Data Sample 1 (1).xlsx](https://github.com/user-attachments/files/30113814/BDO.Reverse.Eng.-.Data.Sample.1.1.xlsx)

<img width="2077" height="1200" alt="image" src="https://github.com/user-attachments/assets/13ccc0c4-3d8b-4f53-be0d-ed89da46b756" />
