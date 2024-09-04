# Exploratory Data Analysis with SQL

## Investigating Temporal Trends

### Yearly growth over four years:

#### SQL query:
```
  -- Calculating yearly sales and sales difference between each year
  WITH yearly_sales AS (
      SELECT 
          DISTINCT YEAR(Order_Date) AS years, 
          SUM(Sales) AS sales_raw,
          SUM(Sales) - LAG(SUM(Sales)) OVER (
              ORDER BY YEAR(Order_Date)
          ) AS diff
      FROM 
          superstore.orders
      GROUP BY 
          YEAR(Order_Date)
      ORDER BY  
          years
  ),
  
  -- Calculating percent difference for sales between each year
  percent_diff_unrounded AS (
      SELECT 
          years,
          sales_raw,
          diff / LAG(sales_raw) OVER (ORDER BY years) AS percent_diff_raw
      FROM 
          yearly_sales
  )
  
  -- Formatting the numeric values
  SELECT
      years,
      CONCAT('$', FORMAT(sales_raw, 'C')) AS sales,
      CONCAT(ROUND(percent_diff_raw * 100, 2), '%') AS 'percent diff from prev year'
  FROM 
      percent_diff_unrounded
  ORDER BY 
      years;
```
#### Outputted table:

#### Remarks:
- consistent growth in sales over the past four years

### Average sales per month over four years:
  
#### SQL query:
```
  SELECT DISTINCT
      RANK() OVER (
          ORDER BY SUM(Sales) DESC
      ) AS 'rank',
      MONTH(Order_Date) AS 'month',
      CONCAT('$', FORMAT(SUM(Sales)/4, 'C')) AS avg_sales
  FROM 
      superstore.orders
  GROUP BY 
      MONTH(Order_Date)
  ORDER BY 
      avg_sales DESC;
```
#### Outputted table:

#### Remarks:
- the months at the end of the year tend to have the highest average sales
- the month with the highest average sales has more than five times the month with the lowest average sales
    
- **Max and min sales and the months that they occured in**
```
  -- For each year, rank each month according to total sales
  WITH monthly_rank AS (
      SELECT
          YEAR(Order_Date) AS year_,
          MONTH(Order_Date) AS month_,
          SUM(Sales) AS sales,
          RANK() OVER (
              PARTITION BY YEAR(Order_Date)
              ORDER BY SUM(Sales) DESC
          ) AS sales_rank
      FROM `superstore`.`orders`
      GROUP BY year_, month_
      ORDER BY year_, sales_rank
  ),
  
  -- Column for min_month and min_sales
  min_month_table AS (
      SELECT 
          year_,
          month_ AS min_month,
          CONCAT('$', FORMAT(sales, 'C')) AS min_sales
      FROM monthly_rank
      WHERE sales_rank = 12
  ),
  
  -- Column for max_month and max_sales
  max_month_table AS (
      SELECT 
          year_,
          month_ AS max_month,
          CONCAT('$', FORMAT(sales, 'C')) AS max_sales
      FROM monthly_rank
      WHERE sales_rank = 1
  )
  
  -- Combine min and max tables
  SELECT 
      min_month_table.*,
      max_month_table.max_month,
      max_month_table.max_sales
  FROM min_month_table
  JOIN max_month_table
      ON min_month_table.year_ = max_month_table.year_;
```
#### Outputted table and chart:

#### Remarks:
- The months with the lowest sales per year is switches between January and February
- The months with the highest sales always occur in the last third of the year

## Investigating regional trends 

### Regional sales growth 

#### SQL query:
```
  -- Calculate yearly sales
  WITH yearly_regional_sales AS (
  	SELECT 
  		Region, 
  		SUM(CASE WHEN YEAR(Order_Date) = 2019 THEN Sales ELSE 0 END) AS sales_2019,
  		SUM(CASE WHEN YEAR(Order_Date) = 2020 THEN Sales ELSE 0 END) AS sales_2020,
  		SUM(CASE WHEN YEAR(Order_Date) = 2021 THEN Sales ELSE 0 END) AS sales_2021,
  		SUM(CASE WHEN YEAR(Order_Date) = 2022 THEN Sales ELSE 0 END) AS sales_2022
  	FROM `superstore`.`orders`
  	GROUP BY Region
  	ORDER BY Region
  ),
  -- Calculated percent differences
  percent_diff_table AS (
  	SELECT
  		Region,
  		sales_2019,
  		CONCAT(ROUND((sales_2020 - sales_2019) * 100/sales_2019, 1), '%') AS perc_diff_2020,
  		sales_2020,
  		CONCAT(ROUND((sales_2021 - sales_2020) * 100/sales_2020, 1), '%') AS perc_diff_2021, 
  		sales_2021,
  		CONCAT(ROUND((sales_2022 - sales_2021) * 100/sales_2021, 1), '%') AS perc_diff_2022, 
  		sales_2022
  	FROM yearly_regional_sales
  )
  SELECT 
  	Region,
      CONCAT('$', FORMAT(sales_2019, 'C')) AS sales_2019,
      perc_diff_2020, 
      CONCAT('$', FORMAT(sales_2020, 'C')) AS sales_2020,
      perc_diff_2021,
      CONCAT('$', FORMAT(sales_2021, 'C')) AS sales_2021,
      perc_diff_2022,
      CONCAT('$', FORMAT(sales_2022, 'C')) AS sales_2022
  FROM percent_diff_table
```

#### Outputted table and chart:

#### Remarks
- Steady growth in the East, West, and Central regions. Volatile growth in the Southern region.

### Yearly sales by subcategory

#### SQL Query
```
  WITH sales_unformatted AS (
  	SELECT	
  		SubCategory,
  		SUM(CASE WHEN YEAR(Order_Date) = 2019 THEN Sales ELSE 0 END) AS sales_2019,
  		SUM(CASE WHEN YEAR(Order_Date) = 2020 THEN Sales ELSE 0 END) AS sales_2020,
  		SUM(CASE WHEN YEAR(Order_Date) = 2021 THEN Sales ELSE 0 END) AS sales_2021,
  		SUM(CASE WHEN YEAR(Order_Date) = 2022 THEN Sales ELSE 0 END) AS sales_2022,
  	FROM `superstore`.`orders`
  	GROUP BY SubCategory
  )
  SELECT 
  	SubCategory,
      CONCAT('$', FORMAT(sales_2019, 'C')) AS sales_2019, 
      CONCAT('$', FORMAT(sales_2020, 'C')) AS sales_2020,
      CONCAT('$', FORMAT(sales_2021, 'C')) AS sales_2021,
      CONCAT('$', FORMAT(sales_2022, 'C')) AS sales_2022,
      CONCAT('$', FORMAT((sales_2019 + sales_2020 + sales_2021 + sales_2022)/4, 'C')) AS avg_yearly_sales
  FROM sales_unformatted
  ORDER BY (sales_2019 + sales_2020 + sales_2021 + sales_2022)/4 DESC
```
#### Outputted table and chart:

#### Remarks:
- Every subcategory except for supplies category has experienced growth over four years
- Sales growth of subcategories with lower sales tends to be more volatile








