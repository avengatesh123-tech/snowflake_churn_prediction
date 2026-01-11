# 📞 Telecom Churn Prediction using Snowflake & ML

![Snowflake](https://img.shields.io/badge/Snowflake-00BFFF?style=for-the-badge&logo=snowflake&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-4479A1?style=for-the-badge&logo=sql&logoColor=white)
![Machine Learning](https://img.shields.io/badge/Machine%20Learning-F0DB4F?style=for-the-badge&logo=python&logoColor=white)
![Data Engineering](https://img.shields.io/badge/Data%20Engineering-FF6F61?style=for-the-badge)

---

## 🚀 Project Overview
This project demonstrates **Telecom Customer Churn Prediction** using **Snowflake SQL ML**.  
The workflow includes:

1. **Database & Warehouse creation**  
2. **Data ingestion from CSV**  
3. **Feature engineering**  
4. **Train/test split**  
5. **Model training using XGBoost inside Snowflake**  
6. **Prediction and evaluation**  

Goal: Identify customers at risk of churn and provide actionable insights to improve retention.

---

## 🔧 Features
- ✅ Create Snowflake warehouse and database  
- ✅ Load CSV into RAW schema  
- ✅ Encode categorical features and engineer numerical features  
- ✅ Train XGBoost model fully inside Snowflake  
- ✅ Generate predictions and evaluate accuracy  
- ✅ Identify top features contributing to churn  

---

## 💻 Tech Stack
- **Database & ML:** Snowflake  
- **ETL & SQL:** Snowflake SQL  
- **Machine Learning:** XGBoost (SQL-based)  
- **Data Processing:** Feature engineering in SQL  

---

## 🛠 Snowflake Workflow

### 1️⃣ Create Warehouse & Database
```sql

-- Create Warehouse
CREATE OR REPLACE WAREHOUSE ML_WH
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = 300
    AUTO_RESUME = TRUE;

USE WAREHOUSE ML_WH;

-- Create Database and Schemas
CREATE OR REPLACE DATABASE TELCO_ML;
USE DATABASE TELCO_ML;

CREATE OR REPLACE SCHEMA RAW;
CREATE OR REPLACE SCHEMA FEATURE;
CREATE OR REPLACE SCHEMA ML;

-- Create Stage and File Format
CREATE OR REPLACE STAGE TELCO_STAGE;

CREATE OR REPLACE FILE FORMAT TELCO_CSV_FORMAT
    TYPE = CSV
    FIELD_DELIMITER = ','
    SKIP_HEADER = 1
    NULL_IF = ('');

-- Create Raw Table
CREATE OR REPLACE TABLE RAW.TELCO_CUSTOMER_CHURN (
    customerID        STRING,
    gender            STRING,
    SeniorCitizen     INT,
    Partner           STRING,
    Dependents        STRING,
    tenure            INT,
    PhoneService      STRING,
    MultipleLines     STRING,
    InternetService   STRING,
    OnlineSecurity    STRING,
    OnlineBackup      STRING,
    DeviceProtection  STRING,
    TechSupport       STRING,
    StreamingTV       STRING,
    StreamingMovies   STRING,
    Contract          STRING,
    PaperlessBilling  STRING,
    PaymentMethod     STRING,
    MonthlyCharges    FLOAT,
    TotalCharges      FLOAT,
    Churn             STRING
);

-- Load Data
LIST @TELCO_ML.ML.TELCO_STAGE;

COPY INTO RAW.TELCO_CUSTOMER_CHURN
FROM @TELCO_ML.ML.TELCO_STAGE
FILE_FORMAT = TELCO_CSV_FORMAT
ON_ERROR = 'CONTINUE';

LIST @TELCO_ML.ML.TELCO_STAGE;

COPY INTO RAW.TELCO_CUSTOMER_CHURN
FROM @TELCO_STAGE
FILE_FORMAT = TELCO_CSV_FORMAT
ON_ERROR = 'CONTINUE';

-- Feature Engineering
CREATE OR REPLACE TABLE FEATURE.TELCO_FEATURES AS
SELECT
    customerID,
    SeniorCitizen,
    tenure,
    MonthlyCharges,
    TotalCharges,

    -- Encoding categorical features
    CASE WHEN gender = 'Male' THEN 1 ELSE 0 END AS gender_male,
    CASE WHEN Partner = 'Yes' THEN 1 ELSE 0 END AS has_partner,
    CASE WHEN Dependents = 'Yes' THEN 1 ELSE 0 END AS has_dependents,
    CASE WHEN PaperlessBilling = 'Yes' THEN 1 ELSE 0 END AS paperless_billing,

    -- Target label
    CASE WHEN Churn = 'Yes' THEN 1 ELSE 0 END AS churn_label
FROM RAW.TELCO_CUSTOMER_CHURN
WHERE TotalCharges IS NOT NULL;

-- Train/Test Split
CREATE OR REPLACE TABLE ML.TRAIN_DATA AS
SELECT *
FROM FEATURE.TELCO_FEATURES
SAMPLE (80);

CREATE OR REPLACE TABLE ML.TEST_DATA AS
SELECT *
FROM FEATURE.TELCO_FEATURES
MINUS
SELECT *
FROM ML.TRAIN_DATA;

-- Inspect Test Data
SELECT * 
FROM ML.TEST_DATA;

-- Predictions with Confidence Score
SELECT 
    customerID,
    churn_label AS actual_value,
    prediction_results:class::STRING AS predicted_value,
    GET(prediction_results:probability, prediction_results:class::STRING)::FLOAT AS confidence_score
FROM (
    SELECT *, 
           churn_model!PREDICT(INPUT_DATA => OBJECT_CONSTRUCT(*)) AS prediction_results
    FROM ML.TEST_DATA
);

-- Prediction Distribution
SELECT 
    prediction_results:class::STRING AS prediction,
    COUNT(*) AS customer_count
FROM (
    SELECT churn_model!PREDICT(INPUT_DATA => OBJECT_CONSTRUCT(*)) AS prediction_results
    FROM ML.TEST_DATA
)
GROUP BY 1;

-- Confusion Matrix
SELECT 
    actual_label, 
    predicted_label, 
    COUNT(*) AS count
FROM (
    SELECT 
        churn_label AS actual_label,
        churn_model!PREDICT(INPUT_DATA => OBJECT_CONSTRUCT(*)):class::INT AS predicted_label
    FROM ML.TEST_DATA
)
GROUP BY 1, 2
ORDER BY 1, 2;


📊 Outcome

Successfully trained XGBoost model fully inside Snowflake

Predicted churn for test customers

Identified customers at risk and key features affecting churn

🔖 Hashtags

#DataEngineering #MachineLearning #Snowflake #SQL #XGBoost #ETL #BigData #AI #DataAnalytics #ChurnPrediction
