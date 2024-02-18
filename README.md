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
### 1. Main Query:
    
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

## 2. Linear Regression Using Stored Procedure:

    CREATE PROCEDURE `LINEAR_REGRESSION`(IN tableName varchar(255), IN xColumn varchar(255), IN yColumn varchar(255))
    BEGIN
    
        DECLARE n INT;
        DECLARE sum_x DOUBLE;
        DECLARE sum_y DOUBLE;
        DECLARE sum_xy DOUBLE;
        DECLARE sum_x_squared DOUBLE;
        DECLARE sum_y_squared DOUBLE;
        DECLARE slope DOUBLE;
        DECLARE intercept DOUBLE;
        DECLARE r_squared DOUBLE;
        
        SET @sql = CONCAT('SELECT COUNT(*) INTO @n FROM ', tableName);
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
    
        SET @sql = CONCAT('SELECT SUM(', xColumn, '), SUM(', yColumn, '), SUM(', xColumn, ' * ', yColumn, '), SUM(', xColumn, ' * ', xColumn, '), SUM(', yColumn, ' * ', yColumn, ') INTO @sum_x, @sum_y, @sum_xy, @sum_x_squared, @sum_y_squared FROM ', tableName);
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
    
        SET n = @n;
        SET sum_x = @sum_x;
        SET sum_y = @sum_y;
        SET sum_xy = @sum_xy;
        SET sum_x_squared = @sum_x_squared;
        SET sum_y_squared = @sum_y_squared;
    
        SET slope = ((n * sum_xy) - (sum_x * sum_y)) / ((n * sum_x_squared) - (sum_x * sum_x));
        SET intercept = (sum_y - slope * sum_x) / n;
    
        SET r_squared = POWER((n * sum_xy - sum_x * sum_y) / SQRT((n * sum_x_squared - sum_x * sum_x) * (n * sum_y_squared - sum_y * sum_y)), 2);
    	
        SET @slope := slope;
        SET @intercept := intercept;
        SET @r_squared := r_squared;
        
        select slope, intercept, r_squared;
    END

## Visualization
 Here is the Visualization after Forecasting the Sales Data:
 ![My Image](https://github.com/miteshbhardiya/Sales-Forecasting-Using-SQL/blob/main/graph1.png?raw=true)



    
    
