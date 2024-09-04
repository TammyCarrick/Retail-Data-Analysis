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
|years|sales   |percent diff from prev year|
|-----|--------|---------------------------|
|2019 |$422,437|NULL                       |
|2020 |$507,864|20.22%                     |
|2021 |$591,224|16.41%                     |
|2022 |$775,676|31.2%                      |

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
#### Outputted table and chart:

|Table 1|Table 2|
|--|--|
|<table> <tr><th>rank</th><th>month</th><th>avg_sales</th><tr><td>1</td><td>11</td><td>$91,666</td></tr><tr><td>2</td><td>12</td><td>$74,884</td></tr><tr><td>3</td><td>9</td><td>$68,642</td></tr><tr><td>4</td><td>7</td><td>$48,513</td></tr><tr><td>5</td><td>10</td><td>$45,239</td></tr><tr><td>6</td><td>4</td><td>$45,227</td></tr><tr><td>7</td><td>8</td><td>$41,661</td></tr><tr><td>8</td><td>3</td><td>$40,798</td></tr><tr><td>9</td><td>5</td><td>$39,626</td></tr><tr><td>10</td><td>6</td><td>$39,316</td></tr><tr><td>11</td><td>1</td><td>$21,040</td></tr><tr><td>12</td><td>2</td><td>$17,688</td></tr> </table>| <table> <tr>![avg monthly sales chart](https://github.com/TammyCarrick/Retail-Data-Analysis/blob/main/images/average%20sales%20by%20month.png?raw=true)</tr></table>|

#### Remarks:
- the months at the end of the year tend to have the highest average sales
- the month with the highest average sales has more than five times the month with the lowest average sales
    
### Max and min sales and the months that they occured in

#### SQL query
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
#### Outputted table:
|year_|min_month|min_sales|max_month|max_sales|
|-----|---------|---------|---------|---------|
|2019 |1        |$12,650  |11       |$69,468  |
|2020 |2        |$11,309  |11       |$115,274 |
|2021 |1        |$18,591  |12       |$80,273  |
|2022 |2        |$21,286  |9        |$106,342 |

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
|Region|sales_2019|perc_diff_2020|sales_2020|perc_diff_2021|sales_2021|perc_diff_2022|sales_2022|
|------|----------|--------------|----------|--------------|----------|--------------|----------|
|Central|$80,843   |25.9%         |$101,800  |47.1%         |$149,699  |12.8%         |$168,897  |
|East  |$138,721  |15.7%         |$160,492  |11.1%         |$178,285  |12.9%         |$201,283  |
|South |$71,753   |44.7%         |$103,812  |-24.7%        |$78,145   |76.6%         |$138,012  |
|West  |$131,121  |8.1%          |$141,759  |30.6%         |$185,094  |44.5%         |$267,484  |

<img src="https://github.com/TammyCarrick/Retail-Data-Analysis/blob/main/images/regional%20sales.png?raw=true" width="700" height="400">

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
|SubCategory|sales_2019|sales_2020|sales_2021|sales_2022|avg_yearly_sales|
|-----------|----------|----------|----------|----------|----------------|
|Phones     |$59,681   |$73,365   |$81,154   |$115,807  |$82,502         |
|Chairs     |$63,055   |$63,668   |$86,111   |$115,615  |$82,112         |
|Storage    |$41,267   |$47,518   |$57,084   |$77,975   |$55,961         |
|Tables     |$42,592   |$42,181   |$52,430   |$69,762   |$51,741         |
|Binders    |$19,343   |$39,325   |$60,211   |$84,534   |$50,853         |
|Machines   |$40,456   |$50,031   |$47,801   |$50,951   |$47,310         |
|Accessories|$36,778   |$36,917   |$40,950   |$52,735   |$41,845         |
|Copiers    |$21,630   |$47,210   |$30,349   |$50,339   |$37,382         |
|Bookcases  |$19,452   |$21,464   |$31,146   |$42,818   |$28,720         |
|Appliances |$20,724   |$26,103   |$28,808   |$31,898   |$26,883         |
|Furnishings|$20,423   |$19,352   |$19,823   |$32,107   |$22,926         |
|Paper      |$17,433   |$17,146   |$17,544   |$26,356   |$19,620         |
|Supplies   |$7,209    |$11,114   |$23,156   |$5,195    |$11,668         |
|Art        |$5,694    |$5,333    |$6,925    |$9,167    |$6,780          |
|Envelopes  |$3,438    |$3,693    |$4,740    |$4,605    |$4,119          |
|Labels     |$2,794    |$2,649    |$2,259    |$4,784    |$3,122          |
|Fasteners  |$469      |$794      |$733      |$1,028    |$756            |

<img src="https://github.com/TammyCarrick/Retail-Data-Analysis/blob/main/images/yearly%20sales%20by%20subcategory.png?raw=true" width = "700" height="400">



#### Remarks:
- Every subcategory except for supplies category has experienced growth over four years
- Sales growth of subcategories with lower sales tends to be more volatile








