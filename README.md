<img width="2542" height="1347" alt="image" src="https://github.com/user-attachments/assets/8d5dc4b2-2880-4bf9-846f-3b01c1df112f" />







////

<Parent Company>

Site: <Company (Subsidiary)>
Location: <City, State>, <Country>
Region: <Region>

Capacity: <SUM(Current Installed Capacity)> L
Type: <Modality> | <Capacity Grade>

////


Competitor Capacity Dashboard

Purpose
The purpose of this dashboard is to provide a clear strategic overview of competitor CDMO capacity. The dashboard should help answer who has the largest installed capacity, how capacity has developed over time, and how capacity is distributed across modality, capacity grade and geography.

The dashboard should separate current installed capacity from future or planned capacity, especially because the dataset includes placeholder future dates such as 2099. This avoids misleading conclusions in the market overview.

Key business questions
1. What is the current installed capacity in the market?
2. How much future or planned capacity is included in the dataset?
3. Which competitors have the largest installed capacity?
4. How has installed capacity developed over time?
5. How is capacity distributed across Commercial vs Clinical and Mammalian vs Microbial?

Recommended dashboard structure

Top filter bar:
Version | Selected Year | Parent Company | Modality | Capacity Grade | Region

KPI row:
Current Installed Capacity | Planned Capacity | Commercial Share | Mammalian Share | Top Competitor | Number of Sites

Main chart:
Cumulative Installed Capacity Over Time

Supporting charts:
Top 20 Competitors by Installed Capacity
Current vs Planned Capacity Split

Deep-dive charts:
Capacity by Modality Over Time
Capacity by Capacity Grade Over Time

Optional:
Capacity by Region or Global Capacity Map
Detail Table with site-level data


Core calculated fields

1. Capacity Status

IF ISNULL([Online Initial Date]) THEN "Unknown"
ELSEIF YEAR([Online Initial Date]) <= YEAR(TODAY()) THEN "Current"
ELSE "Planned"
END


2. Current Installed Capacity

IF [Capacity Status] = "Current"
THEN [Volume Total]
END


3. Planned Capacity

IF [Capacity Status] = "Planned"
THEN [Volume Total]
END


4. Capacity Up To Selected Year

IF NOT ISNULL([Online Initial Date])
AND YEAR([Online Initial Date]) <= [Selected Year]
THEN [Volume Total]
END


5. Commercial Share

SUM(
    IF [Capacity Grade] = "Commercial"
    AND [Capacity Status] = "Current"
    THEN [Volume Total]
    END
)
/
SUM([Current Installed Capacity])


6. Mammalian Share

SUM(
    IF [Modality] = "Mammalian"
    AND [Capacity Status] = "Current"
    THEN [Volume Total]
    END
)
/
SUM([Current Installed Capacity])


7. Competitor Group

IF [Parent Company] IN ("Lonza", "Samsung Biologics", "AGC Biologics", "WuXi Biologics", "FUJIFILM")
THEN [Parent Company]
ELSE "Other"
END


Recommended visualizations

1. Cumulative Installed Capacity Over Time

Purpose:
Shows how total installed market capacity has developed over time.

Setup:
Columns: YEAR(Online Initial Date)
Rows: SUM(Current Installed Capacity)
Table Calculation: Running Total
Marks: Line
Color: Modality or Competitor Group

Recommendation:
Use Modality or Competitor Group as color. Do not use all companies as colors, because the chart becomes too noisy.


2. Top 20 Competitors by Installed Capacity

Purpose:
Shows which competitors have the largest current installed capacity.

Setup:
Rows: Parent Company
Columns: SUM(Current Installed Capacity)
Sort: Descending
Filter: Top 20 by SUM(Current Installed Capacity)
Marks: Bar
Label: SUM(Current Installed Capacity)

Recommendation:
Use Parent Company instead of Company/Subsidiary. Parent Company gives a more strategic competitor view.


3. Current vs Planned Capacity Split

Purpose:
Shows how much of the dataset is current installed capacity versus planned or future capacity.

Setup:
Columns: Capacity Status
Rows: SUM(Volume Total)
Color: Capacity Grade
Marks: Bar
Label: SUM(Volume Total)

Recommendation:
This chart is important because it explains the 2099 data points instead of hiding them.


4. Capacity by Modality Over Time

Purpose:
Shows whether market capacity growth is mainly driven by Mammalian or Microbial capacity.

Setup:
Columns: YEAR(Online Initial Date)
Rows: SUM(Current Installed Capacity)
Table Calculation: Running Total
Marks: Line
Color: Modality

Recommendation:
Only use Modality as color. The chart should show Mammalian vs Microbial over time.


5. Capacity by Capacity Grade Over Time

Purpose:
Shows how Clinical and Commercial capacity have developed over time.

Setup:
Columns: YEAR(Online Initial Date)
Rows: SUM(Current Installed Capacity)
Table Calculation: Running Total
Marks: Line
Color: Capacity Grade

Recommendation:
Only use Capacity Grade as color. The chart should show Clinical vs Commercial over time.


6. Capacity by Region

Purpose:
Shows where capacity is geographically concentrated.

Setup:
Rows: Region
Columns: SUM(Current Installed Capacity)
Color: Modality
Sort: Descending
Marks: Bar
Label: SUM(Current Installed Capacity)

Alternative:
Use a map if the latitude and longitude data is reliable.


KPI row

Recommended KPIs:
1. Current Installed Capacity
2. Planned Capacity
3. Commercial Share
4. Mammalian Share
5. Top Competitor
6. Number of Sites

Formatting:
Use compact number formatting, for example:
11.8M L
1.6M L
83%
Samsung Biologics

Avoid showing large raw numbers such as:
11,776,494
1,595,600


Design rules

1. Use only one color dimension per chart.
2. Use Parent Company for competitor views.
3. Use Current Installed Capacity in historical charts.
4. Keep Planned Capacity separate from Current Capacity.
5. Format capacity values in millions.
6. Use labels only at line ends, not on every data point.
7. Remove unnecessary gridlines.
8. Avoid too many colors.
9. Use short and clear chart titles.
10. Make sure dashboard filters apply to all relevant worksheets.


Recommended chart titles

Current Installed Capacity Over Time
Top 20 Competitors by Installed Capacity
Current vs Planned Capacity
Capacity by Modality Over Time
Capacity by Capacity Grade Over Time
Capacity by Region


Things to avoid

Avoid:
- Showing all companies as different colors in the same line chart
- Mixing both Capacity Grade and Modality in the same line chart
- Including 2099 in current historical capacity views
- Showing labels on every data point
- Using too many KPI cards without a clear purpose
- Using Company/Subsidiary as the main competitor dimension


Executive storyline

The dashboard shows that the CDMO capacity market has grown significantly over time, with current installed capacity concentrated among a limited number of major competitors. Mammalian capacity represents the dominant modality, while commercial capacity accounts for the largest share of current installed volume. Future or planned capacity is separated from current capacity to avoid distortion from future placeholder dates such as 2099.


Final dashboard recommendation

The dashboard should include these six core elements:

1. KPI row
2. Cumulative Installed Capacity Over Time
3. Top 20 Competitors by Installed Capacity
4. Current vs Planned Capacity Split
5. Capacity by Modality Over Time
6. Capacity by Capacity Grade Over Time

The goal is not to show as many charts as possible. The goal is to make the dashboard easy to understand within 30 seconds.

The dashboard should clearly answer:
- Where is the market today?
- Who dominates the market?
- How has capacity developed over time?
- What is current capacity versus planned capacity?
- Which modalities and capacity grades are driving market capacity?
