# Sales-Forecasting-Using-SQL

## Overview
This project is modeled to showcase an end-to-end approach to sales forecasting, leveraging time series analysis and linear regression techniques. The primary tools used in this project are SQL for data manipulation, analysis, and modeling, with Excel for the sole purpose of visualization.

## Key Features

1. ### Time Series Analysis and Linear Regression
This project leverages time series analysis techniques and linear regression models for sales forecasting. Exploring historical sales data to uncover patterns and trends.

2. ### SQL-Centric Workflow
Utilizes SQL queries for seamless data manipulation, analysis, and model development. Gained hands-on experience with SQL.

3. ### Excel Visualization
Given the absence of a Visualization tool in MySql, I chose MS Excel for Visualization.

## SQL Query


'''sql
    
    CREATE TABLE SALES_DATA.BRANCH_TABLE AS SELECT YEAR_ID, MONTH_ID, SUM(SALES) AS MONTHLY_SALES FROM
    sales_data.salesdataset
    GROUP BY YEAR_ID , MONTH_ID
    ORDER BY YEAR_ID , MONTH_ID;
    
    ALTER TABLE sales_data.branch_table
    ADD COLUMN T INT auto_increment primary key;
    
    CREATE TABLE MOVING_AVERAGETB AS
    select 
    	T, YEAR_ID, MONTH_ID,
    	MONTHLY_SALES,
    		CASE WHEN ROW_NUMBER() OVER(ORDER BY YEAR_ID, MONTH_ID) IN(1, 29) THEN 0 
    		ELSE ROUND(((MONTHLY_SALES + (LAG(MONTHLY_SALES, 1, 0) OVER( ORDER BY YEAR_ID, MONTH_ID)) + (LEAD(MONTHLY_SALES, 1, 0)     OVER(ORDER BY YEAR_ID, MONTH_ID))) / 3), 3)
    		END AS MOVING_AVERAGE
    FROM SALES_DATA.BRANCH_TABLE;
    
    CREATE TABLE DTRENDTB AS
    SELECT 
    	T, YEAR_ID, MONTH_ID, MONTHLY_SALES, MOVING_AVERAGE,
        CASE
    		WHEN row_number() OVER(ORDER BY YEAR_ID, MONTH_ID) IN(1, 29) THEN 0
            ELSE (MONTHLY_SALES - MOVING_AVERAGE)
            END AS DETREND
    FROM MOVING_AVERAGETB;
    
    CREATE TABLE SALES_DATA.SEASONAL_INDEXTB AS SELECT MONTH_ID,
        ROUND(SUM(DETREND) / SUM(CASE
                    WHEN DETREND <> 0 THEN 1
                    ELSE 0
                END),
                2) AS SEASONAL_INDEX FROM
        DTRENDTB
    GROUP BY MONTH_ID;
    
    
    UPDATE branch_table
            JOIN
        dtrendtb ON dtrendtb.T = branch_table.T
            JOIN
        seasonal_indextb ON seasonal_indextb.MONTH_ID = branch_table.MONTH_ID 
    SET 
        branch_table.DETREND = dtrendtb.DETREND,
        branch_table.SEASONAL_INDEX = seasonal_indextb.SEASONAL_INDEX;
    
    call sales_data.LINEAR_REGRESSION('branch_table', 'T', 'MONTHLY_SALES');
    
    SET SQL_SAFE_UPDATES = 0;
    UPDATE branch_table 
    SET 
        TREND = ROUND((@intercept + (@slope * T)), 3);
    
    WITH RESIDUAL_ID AS(
    SELECT T, MONTH_ID, ROUND((MONTHLY_SALES - TREND - SEASONAL_INDEX), 2) AS RESIDUAL
     FROM sales_data.branch_table),
     
     RESIDUALIDX AS(
     SELECT MONTH_ID, AVG(RESIDUAL) AS RESIDUAL_INDEX
     FROM RESIDUAL_ID
     GROUP BY MONTH_ID)
     
    UPDATE sales_data.branch_table
    JOIN RESIDUALIDX ON RESIDUALIDX.MONTH_ID = branch_table.MONTH_ID
    SET sales_data.branch_table.RESIDUAL = RESIDUALIDX.RESIDUAL_INDEX;
    
    ALTER TABLE sales_data.branch_table
    ADD COLUMN FORECAST DOUBLE NOT NULL;
    
    UPDATE sales_data.branch_table 
    SET 
        FORECAST = (TREND + SEASONAL_INDEX + RESIDUAL);
    SELECT 
        *
    FROM
        branch_table;
    
    INSERT INTO branch_table(T, YEAR_ID, MONTH_ID)
    VALUE (30, 2005, 6), (31, 2005, 7), (32, 2005, 8), (33, 2005, 9), (34, 2005, 10), (35, 2005, 11), (36, 2005, 12);
    
    UPDATE branch_table
            JOIN
        seasonal_indextb ON seasonal_indextb.MONTH_ID = branch_table.MONTH_ID 
    SET 
        branch_table.SEASONAL_INDEX = seasonal_indextb.SEASONAL_INDEX;
    
    SELECT 
        *
    FROM
        sales_data.branch_table;

## Visualization
 Here is the Visualization after Forecasting the Sales Data:

    
    
