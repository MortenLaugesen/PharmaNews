Competitor Overview Dashboard – FUJIFILM Diosynth Biotechnologies

Formålet med dashboardet er at skabe et simpelt og overskueligt competitor overview, der kan bruges til at sammenligne FUJIFILM med relevante CDMO-konkurrenter. Dashboardet skal give et hurtigt overblik over, hvem de vigtigste konkurrenter er, hvilke capabilities de har, hvor deres sites er placeret, hvor stærke de er på kapacitet, og hvilke strategiske moves de foretager sig.

Dashboardet skal ikke kun fungere som en liste over konkurrenter, men som et beslutningsværktøj, der kan hjælpe med at svare på følgende spørgsmål:

1. Hvem er FUJIFILMs vigtigste konkurrenter?
2. Hvor stærke er konkurrenterne sammenlignet med FUJIFILM?
3. Hvilke capabilities tilbyder konkurrenterne?
4. Hvor er deres produktionssites placeret?
5. Hvem ekspanderer eller bevæger sig strategisk?
6. Hvor står FUJIFILM stærkt eller svagt i forhold til markedet?

Dashboardet kan struktureres i fem hovedsider:

1. Executive Overview
2. Capability Deep Dive
3. Site & Capacity Overview
4. News & Strategic Signals
5. Competitor Profile

1. Executive Overview

Den første side skal give et hurtigt overblik over konkurrentlandskabet. Denne side skal kunne forstås på få sekunder og fungere som dashboardets forside.

Siden bør indeholde følgende elementer:

- KPI boxes
- Competitor ranking
- Capability heatmap
- Global site map

KPI boxes kan eksempelvis vise:

- Number of competitors
- Total number of sites
- Top competitor by threat score
- Largest capacity competitor
- Most active competitor
- FUJIFILM position

Formålet med KPI-boksene er at give et hurtigt executive summary. Brugeren skal hurtigt kunne se, hvor mange konkurrenter der analyseres, hvem der vurderes som den største trussel, og hvor FUJIFILM står i forhold til de andre.

Competitor ranking kan laves som et bar chart, hvor konkurrenterne rangeres efter en samlet “Threat Score”. Denne score kan baseres på faktorer som capacity, capabilities, geographic footprint, recent expansions og commercial maturity.

Eksempel på ranking:

- Samsung Biologics
- Lonza
- WuXi Biologics
- Boehringer Ingelheim BioXcellence
- Thermo Fisher / Patheon
- AGC Biologics
- KBI Biopharma
- Rentschler Biopharma
- FUJIFILM as benchmark

Threat Score kan eksempelvis beregnes ud fra:

- Capacity Score
- Capability Score
- Geographic Footprint Score
- Expansion Activity Score
- Commercial Maturity Score

Et simpelt eksempel på beregning:

Threat Score =
0.25 * Capacity Score
+ 0.25 * Capability Score
+ 0.20 * Geographic Footprint Score
+ 0.15 * Expansion Activity Score
+ 0.15 * Commercial Maturity Score

2. Capability Deep Dive

Denne side skal vise, hvilke capabilities de forskellige konkurrenter tilbyder. Dette er en af de vigtigste dele af dashboardet, fordi det gør det muligt at sammenligne FUJIFILM direkte med konkurrenterne.

Den bedste visualisering vil være en capability matrix eller heatmap.

Rows:
- Competitors

Columns:
- Capabilities

Eksempler på capabilities:

- Mammalian cell culture
- Microbial fermentation
- Drug substance manufacturing
- Drug product manufacturing
- Fill-finish
- Analytical development
- Process development
- Cell line development
- Commercial GMP manufacturing
- Clinical GMP manufacturing
- ADC capabilities
- Cell therapy
- Gene therapy
- Viral vectors
- mRNA
- Regulatory support

Hver capability kan scores simpelt fra 0 til 3:

0 = Not present / not found
1 = Low / limited capability
2 = Medium capability
3 = High / strong capability

Eksempel på matrix:

Competitor | Mammalian | Microbial | Drug Product | ADC | Cell Therapy | Commercial GMP
FUJIFILM | High | High | Medium | Medium | Medium | High
Lonza | High | Medium | High | High | High | High
Samsung Biologics | Very High | Low | Medium | Medium | Low | High
WuXi Biologics | High | Medium | High | Medium | Medium | High
AGC Biologics | Medium | High | Medium | Medium | High | Medium

Denne side skal gøre det let at se, hvor FUJIFILM er stærk, og hvor konkurrenterne eventuelt har en fordel.

3. Site & Capacity Overview

Denne side skal vise konkurrenternes globale footprint og kapacitet.

Siden bør indeholde:

- Global map
- Capacity bar chart
- Site table

Map visualiseringen skal vise, hvor konkurrenternes sites er placeret. Hvert site kan vises som en prik på kortet.

Information i tooltip kan være:

- Company
- Site name
- Country
- Region
- Modality
- Known capacity
- Commercial or clinical site
- GMP status
- Source
- Confidence level

Capacity kan vises som enten præcise liter-tal eller som simple kategorier, hvis data er usikker.

Eksempel på capacity categories:

- Low
- Medium
- High
- Very High

Dette er ofte mere realistisk i en første version, fordi offentlig capacity-data kan være usikker eller svær at sammenligne direkte.

Eksempel på capacity overview:

Competitor | Known biologics capacity
Samsung Biologics | Very High
Lonza | High
Boehringer Ingelheim BioXcellence | High
FUJIFILM | High
WuXi Biologics | High
AGC Biologics | Medium
KBI Biopharma | Medium
Rentschler Biopharma | Medium

Formålet med denne side er at vise, hvem der har fysisk produktionskapacitet, hvor kapaciteten er placeret, og hvordan FUJIFILM står geografisk i forhold til konkurrenterne.

4. News & Strategic Signals

Denne side skal gøre dashboardet dynamisk og fremadskuende. I stedet for kun at vise statisk information skal dashboardet også vise, hvilke konkurrenter der bevæger sig strategisk.

Eksempler på signaltyper:

- Expansion
- Acquisition
- Partnership
- New technology
- New facility
- Regulatory issue
- New customer/project
- Portfolio change
- Investment
- Commercial milestone

Tabellen kan indeholde følgende felter:

- Date
- Competitor
- Signal type
- Headline
- Short description
- Region
- Modality
- Strategic impact
- Source

Strategic impact kan scores simpelt:

Low = Limited relevance
Medium = Relevant competitor movement
High = Important strategic move

Eksempel:

Date | Competitor | Signal Type | Description | Impact
2026 | FUJIFILM | Expansion | Expansion of Hillerød site | High
2025 | Samsung Biologics | Expansion | Increased manufacturing footprint | High
2025 | Lonza | Portfolio move | Focus on pure-play CDMO strategy | Medium/High
2024 | Catalent | Acquisition | Acquired by Novo Holdings | High

Denne side skal hjælpe med at identificere konkurrenter, der ekspanderer, investerer eller på anden måde ændrer deres strategiske position.

5. Competitor Profile

Denne side skal fungere som en one-pager for hver konkurrent. Brugeren skal kunne vælge én konkurrent og få et hurtigt overblik over virksomhedens position.

Siden kan bygges med en parameter eller filter i Tableau, hvor man vælger competitor.

Profilen bør indeholde:

- Company name
- Headquarters
- Ownership
- Main modalities
- Key sites
- Key capabilities
- Strengths
- Weaknesses
- Recent news
- Relevance vs FUJIFILM
- Threat level

Eksempel:

Competitor: Samsung Biologics

Strengths:
Samsung Biologics has very large-scale mammalian manufacturing capacity and is a strong competitor for large commercial biologics manufacturing.

Weaknesses:
The company appears less diversified across microbial and certain advanced modalities compared with broader CDMO competitors.

Relevance vs FUJIFILM:
Samsung Biologics is especially relevant as a competitor in large-scale mammalian commercial manufacturing. FUJIFILM may differentiate itself through its EU/US footprint, end-to-end offering, microbial and mammalian breadth, and customer proximity.

Threat Level:
High

Recommended data structure

For at bygge dashboardet i Tableau anbefales det at strukturere data i nogle simple tabeller.

1. Competitor table

Fields:

- Competitor ID
- Competitor name
- Headquarters
- Ownership
- Competitor tier
- Company type
- Website
- Notes

2. Site table

Fields:

- Site ID
- Competitor ID
- Site name
- City
- Country
- Region
- Latitude
- Longitude
- Site type
- GMP flag
- Commercial flag

3. Capability table

Fields:

- Competitor ID
- Site ID
- Capability group
- Capability name
- Modality
- Lifecycle stage
- Score 0-3
- Evidence text
- Source URL
- Confidence level

4. Capacity table

Fields:

- Competitor ID
- Site ID
- Modality
- Capacity liters
- Capacity category
- Bioreactor count
- Bioreactor size
- Capacity status
- Year available
- Source
- Confidence level

5. News signals table

Fields:

- Signal ID
- Competitor ID
- Date
- Signal type
- Headline
- Description
- Region
- Modality
- Strategic impact score
- Source URL

Recommended first version / MVP

For a first version, the dashboard should be kept simple. The MVP should include four main elements:

1. Competitor ranking
2. Capability matrix
3. Site overview map
4. Recent activity table

This will be enough to create a strong first version without making the dashboard too complex.

The first version should answer:

- Who are the competitors?
- How strong are they?
- What capabilities do they have?
- Where are their sites?
- Who is expanding?
- Where does FUJIFILM stand strong or weak?

Recommended Tableau pages

Page 1: Executive Overview

Content:
- KPI boxes
- Competitor ranking
- Capability heatmap
- Global map

Purpose:
To give a quick management-level overview of the competitor landscape.

Page 2: Capability Deep Dive

Content:
- Competitor x capability heatmap
- Filter by modality
- Filter by lifecycle stage
- Capability score

Purpose:
To compare FUJIFILM and competitors across key CDMO capabilities.

Page 3: Site & Capacity Overview

Content:
- Global site map
- Capacity chart
- Site table

Purpose:
To understand competitor footprint and manufacturing capacity.

Page 4: News & Strategic Signals

Content:
- Recent competitor moves
- Expansion timeline
- Signal type filter
- Impact score

Purpose:
To track which competitors are moving strategically.

Page 5: Competitor Profile

Content:
- One competitor at a time
- Strengths
- Weaknesses
- Key sites
- Capabilities
- Recent news
- Relevance vs FUJIFILM

Purpose:
To provide a simple one-page competitor summary.

Key message of the dashboard

The dashboard should not only show data. It should help answer the strategic question:

“Where does FUJIFILM have a right-to-win, and where are competitors building pressure?”

The dashboard should make it clear where FUJIFILM is strong, where competitors are stronger, and where there may be white-space opportunities in the market.

In short, the dashboard should contain:

1. A competitor ranking
2. A capability matrix
3. A global site and capacity overview
4. A recent news and strategic signals tracker
5. A detailed competitor profile page

This structure will create a simple but useful competitor intelligence dashboard that can support strategic decision-making.
