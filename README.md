# Crop Yield & Weather Analysis Dashboard

### Dashboard Link: https://app.powerbi.com/links/n18vZhVtNs?ctid=500356b3-ac75-4410-bd8f-decd38beb5a2&pbi_source=linkShare

**Tech stack:** Amazon S3 | Snowflake | Power BI Desktop | Power BI Service

---

## Problem Statement

This dashboard helps agricultural planners, farmers and researchers understand how weather conditions (rainfall, temperature and humidity) relate to crop yield across different locations, seasons and years. By comparing these factors for each crop, users can see which crops, seasons and regions perform best, and which conditions are linked to higher or lower yield.

**Key observations that motivated the dashboard:**
- Average yield varies hugely between crops (Cotton ~51K vs Cocoa ~1K), so crop selection matters.
- Yield differs by season and by location, so planning should be region- and season-specific.
- Rainfall, temperature and humidity may explain part of these differences, so each is analysed on its own page.

---

## Dataset

- **Source:** Course-provided dataset (AWS + Snowflake + Power BI project)
- **Format:** CSV
- **Rows x Columns:** 3,158 rows x 12 columns (14 columns after the two derived columns)
- **Columns:** Year, Location, Area, Rainfall, Temperature, Soil_type, Irrigation, yeilds, Humidity, Crops, price, Season
- **Derived columns (added in Snowflake):** Year_Group (Y1, Y2, Y3) and Rainfall_Groups (Low, Medium, High)
- **Seasons covered:** Kharif, Rabi, Zaid
- **Crops covered (13):** Cotton, Coconut, Ginger, Tea, Blackgram, Coffee, Pepper, Paddy, Groundnut, Arecanut, Cardamum, Cashew, Cocoa
- **Locations covered (11):** Kodagu, Mysuru, Madikeri, Kasaragodu, Raichur, Hassan, Chikmangaluru, Bangalore, Gulbarga, Mangalore, Davangere

---

## Architecture / Data Flow

```
CSV file  -->  Amazon S3 bucket  -->  Snowflake (via Storage Integration)  -->  Power BI Desktop  -->  Power BI Service
```

---

## Steps Followed

### Part 1: AWS (Amazon S3 and IAM)
- **Step 1:** Created an Amazon S3 bucket (`powerbi.project8447`) and uploaded the CSV dataset into it.
- **Step 2:** Created an IAM role (`powerbi.role8447`) that gives Snowflake permission to read from the bucket.
- **Step 3:** Created a Storage Integration in Snowflake (Step 4 below), ran `DESC INTEGRATION` to get the `STORAGE_AWS_IAM_USER_ARN` and `STORAGE_AWS_EXTERNAL_ID`, and updated the IAM role's trust policy with those values.

### Part 2: Snowflake (Load and Transform)

**Step 4: Create the storage integration**
```sql
CREATE OR REPLACE STORAGE INTEGRATION PBI_Integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::<YOUR_AWS_ACCOUNT_ID>:role/powerbi.role8447'
  STORAGE_ALLOWED_LOCATIONS = ('s3://powerbi.project8447/')
  COMMENT = 'Integration between Snowflake and S3 for the Power BI project';

-- Describe the integration (copy the IAM user ARN and external ID into the AWS trust policy)
DESC INTEGRATION PBI_Integration;
```

**Step 5: Create the database, schema and table**
```sql
CREATE DATABASE PowerBI;

CREATE SCHEMA PBI_Data;

CREATE TABLE PBI_Dataset (
    Year         INT,
    Location     STRING,
    Area         INT,
    Rainfall     FLOAT,
    Temperature  FLOAT,
    Soil_type    STRING,
    Irrigation   STRING,
    yeilds       INT,
    Humidity     FLOAT,
    Crops        STRING,
    price        INT,
    Season       STRING
);
```

**Step 6: Create the stage and load the data from S3**
```sql
CREATE OR REPLACE STAGE pbi_stage
  URL = 's3://powerbi.project8447/'
  STORAGE_INTEGRATION = PBI_INTEGRATION
  FILE_FORMAT = (TYPE = CSV FIELD_DELIMITER = ',' SKIP_HEADER = 1);

-- Check the stage and the files in it
DESC STAGE pbi_stage;
LIST @pbi_stage;

-- Load the data
COPY INTO PBI_Dataset
FROM @pbi_stage
FILE_FORMAT = (TYPE = CSV FIELD_DELIMITER = ',' SKIP_HEADER = 1)
ON_ERROR = 'continue';
```

**Step 7: Verify the load**
```sql
SELECT * FROM PBI_Dataset LIMIT 10;

SELECT COUNT(*) FROM PBI_Dataset;

SELECT year, COUNT(*)
FROM PBI_DATASET
GROUP BY year
ORDER BY year DESC;
```

**Step 8: Create a working copy and transform the data**

A copy of the raw table (`Agriculture`) was created so the original data stays untouched.
```sql
CREATE TABLE Agriculture AS
SELECT * FROM PBI_DATASET;

SELECT * FROM Agriculture;

-- Adjust rainfall and area values
UPDATE Agriculture
SET rainfall = 1.1 * rainfall;

UPDATE Agriculture
SET area = 0.9 * area;
```
> Note: As part of the data transformation, rainfall values were increased by 10% and area values were reduced by 10%. The rainfall values shown on the dashboard are therefore the adjusted values.

**Step 9: Add a Year Group column**
```sql
ALTER TABLE Agriculture
ADD Year_Group STRING;

-- Y1: 2004 to 2009
UPDATE Agriculture
SET year_group = 'Y1'
WHERE year >= 2004 AND year <= 2009;

-- Y2: 2010 to 2015
UPDATE Agriculture
SET year_group = 'Y2'
WHERE year >= 2010 AND year <= 2015;

-- Y3: 2016 to 2019
UPDATE Agriculture
SET year_group = 'Y3'
WHERE year >= 2016 AND year <= 2019;
```

**Step 10: Add a Rainfall Groups column**

Rainfall in the data ranges from a minimum of 255 to a maximum of 4103, so it was grouped as follows:

| Group | Rainfall Range |
|---|---|
| Low | 255 to below 1200 |
| Medium | 1200 to below 2800 |
| High | 2800 and above (up to 4103) |

```sql
ALTER TABLE agriculture
ADD rainfall_groups STRING;

-- Low
UPDATE agriculture
SET rainfall_groups = 'Low'
WHERE rainfall >= 255 AND rainfall < 1200;

-- Medium
UPDATE agriculture
SET rainfall_groups = 'Medium'
WHERE rainfall >= 1200 AND rainfall < 2800;

-- High
UPDATE agriculture
SET rainfall_groups = 'High'
WHERE rainfall >= 2800;

SELECT * FROM agriculture;
```

### Part 3: Power BI (Report Building)
- **Step 11:** Connected Power BI Desktop to Snowflake and imported the transformed `Agriculture` table.
- **Step 12:** Built the **Rainfall Analysis** page with four bar charts: Average Rainfall by Year, Season, Crops and Location.
- **Step 13:** Built the remaining pages using the same layout:
  - **Temperature Analysis**: Average Temperature by Year, Season, Crops and Location
  - **Humidity Analysis**: Average Humidity by Year, Season, Crops and Location
  - **Yield Analysis**: Average Yield by Year, Season, Crops and Location
- **Step 14:** Applied a consistent design across all pages: pink title banner, white card-style visuals with shadows, and a consistent colour per dimension (orange for Year, purple for Season and Location, yellow for Crops).
- **Step 15:** Published the report to Power BI Service.

---

## Dashboard Snapshots

### Published Report (Power BI Service)
![Power BI Service Snapshot](https://github.com/user-attachments/assets/b6ec9f67-ffcf-4d82-b96f-b13a09a57fcb)

### Rainfall Analysis
![Rainfall Analysis](https://github.com/user-attachments/assets/82942cff-e147-4f58-9c29-50df4eaeaf2d)

### Temperature Analysis
![Temperature Analysis](https://github.com/user-attachments/assets/ad921f0b-f9cd-41c1-aec2-32538966102f)

### Humidity Analysis
![Humidity Analysis](https://github.com/user-attachments/assets/29a0ae55-5e15-40e6-a42a-f2afd34e52f6)

### Yield Analysis
![Yield Analysis](https://github.com/user-attachments/assets/e9eab193-b0b5-4cfe-87fa-2f30fed30e26)

---

## Insights

A four-page report was created in Power BI Desktop and published to Power BI Service. The following inferences can be drawn from the dashboard.

### 1. Yield Analysis

**By Season**

| Season | Average Yield |
|---|---|
| Rabi | 24.9K |
| Zaid | 22.0K |
| Kharif | 20.2K |

**Takeaway:** Rabi has the highest average yield and Kharif the lowest.

**By Crop**

| Rank | Crop | Average Yield |
|---|---|---|
| Highest | Cotton | 51K |
| 2nd | Coconut | 34K |
| 3rd | Ginger | 26K |
| Lowest | Cocoa | 1K |
| 2nd lowest | Cashew | 3K |

**Takeaway:** Cotton gives by far the highest average yield, followed by Coconut and Ginger. Cocoa and Cashew are at the bottom.

**By Location**

| Rank | Location | Average Yield |
|---|---|---|
| Highest | Kodagu | 28.7K |
| 2nd | Mysuru | 27.6K |
| 3rd | Madikeri | 24.9K |
| Lowest | Davangere | 11.8K |

**Takeaway:** Kodagu and Mysuru lead, while Davangere is far behind every other location (11.8K vs 20K+ for the rest).

**By Year:** Average yield ranges from about 16.4K (lowest) to 28.7K (highest) across the years, so it fluctuates noticeably from year to year.

### 2. Rainfall Analysis

| Dimension | Highest | Lowest |
|---|---|---|
| Season | Rabi (3416) | Zaid (3377) |
| Crop | Paddy (3.8K) | Cotton (3.2K) |
| Location | Bangalore (4.2K) | Mysuru (3.2K) |

**Takeaways:**
- Average rainfall is almost identical across seasons (3377 to 3416), so season alone does not explain rainfall differences.
- Paddy sees the most rainfall among crops, while Cotton (the highest-yielding crop) sees the least.
- Across years, average rainfall stays between roughly 3.3K and 3.5K, with one year dipping to about 3.0K.

### 3. Temperature Analysis

| Dimension | Highest | Lowest |
|---|---|---|
| Season | Kharif and Zaid (72 each) | Rabi (61) |
| Crop | Ginger (79) | Paddy (49) |
| Location | Bangalore (186) | Chikmangaluru (33) |

**Takeaways:**
- Rabi is the coolest season (61) compared with Kharif and Zaid (72).
- Ginger is grown at the highest average temperatures, Paddy at the lowest.
- Across years, average temperature ranges from about 41 to 73.

> **Note:** The location-level temperature values (33 to 186) are much larger than the season and crop values (49 to 79). Check the aggregation (Average vs Sum) and the units for this visual before publishing.

### 4. Humidity Analysis

- Average humidity is almost constant at **55 to 56** across every year, season, crop and location.
- **Takeaway:** Humidity barely varies in this dataset, so it is unlikely to explain differences in yield on its own.

### 5. Overall Takeaways
- Crop type is the biggest driver of yield differences, followed by location.
- Rabi is the best-performing season for yield and has the lowest average temperature.
- Rainfall and humidity are nearly uniform across seasons, while temperature is the weather factor that varies most between seasons.

---

## Recommendations

1. Prioritise high-yield crops (Cotton, Coconut, Ginger) in regions whose weather suits them.
2. Investigate why Davangere's average yield is much lower than the other locations.
3. Review Kharif and Zaid practices, since both have lower average yield than Rabi.
4. Add slicers (Year, Season, Crop, Location) and a scatter plot of weather vs yield for deeper analysis.

---

## Tools Used

- Amazon S3 and AWS IAM
- Snowflake (SQL)
- Power BI Desktop
- Power BI Service

## Author

**Shaswat Kumar** | [LinkedIn](https://www.linkedin.com/in/shaswat-kumar07) | [shaswat844@gmail.com](mailto:shaswat844@gmail.com)
