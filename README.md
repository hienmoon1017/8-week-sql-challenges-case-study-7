# 8 Week SQL Challenges by Danny | Case Study #7 - Balanced Tree
### ERD for Database

![image](https://github.com/user-attachments/assets/d71725ad-8389-4471-89d1-bbb0b00ce59b)


### Check duplication
```sql
SELECT count(*)
FROM 
	(SELECT prod_id, qty, price, discount, member, txn_id, start_txn_time
	,count(*) as records
	FROM balanced_tree.sales
	GROUP BY 1,2,3,4,5,6,7)
WHERE records >1;
```
_Result:_ no duplicated rows

### 1. High Level Sales Analysis
#### 1.1 What was the total quantity sold for all products?
```sql
SELECT sum(qty) as total_quantity_sold
FROM 
	(SELECT distinct * 
	 FROM balanced_tree.sales
	) as unduplicated_sales
;
```

_Result:_

![image](https://github.com/user-attachments/assets/23afab05-7864-4203-bf66-d7a5dc7e9b84)


#### 1.2 What is the total generated revenue for all products before discounts?
```sql
WITH revenue_by_product as
(
	SELECT qty * price as revenue
	FROM 
		(SELECT distinct *
		FROM balanced_tree.sales
		) as unduplicated_sales
)
SELECT sum(revenue) as total_revenue_all_products_before_discounts
FROM revenue_by_product;
```

_Result:_

![image](https://github.com/user-attachments/assets/3736ad62-b256-44f2-868b-f9d4f55fd2dd)


#### 1.3 What was the total discount amount for all products?
```sql
SELECT sum(discount) as total_discounts
FROM
	(SELECT distinct *
	FROM balanced_tree.sales
	 )as unduplicated_sales
;
```

_Result:_

![image](https://github.com/user-attachments/assets/f5545627-b452-41ed-bc57-619b3f55cd0a)

### 2. Transaction Analysis
#### 2.1 How many unique transactions were there?
```sql
SELECT count(distinct txn_id) as total_unique_transactions
FROM balanced_tree.sales;
```

_Result:_

![image](https://github.com/user-attachments/assets/36d47d4a-daf9-4403-8797-e369542b5177)


#### 2.2 What is the average unique products purchased in each transaction?
```sql
WITH unique_products_purchased as
(
	SELECT txn_id	
	,count(distinct prod_id) as unique_product_purchased
	FROM balanced_tree.sales
	GROUP BY 1
)
SELECT round(avg(unique_product_purchased),2) as avg_unique_product_purchased
FROM unique_products_purchased;
```

_Result:_

![image](https://github.com/user-attachments/assets/ef98f4b3-8181-4f9c-a6b0-cf4668b36695)


#### 2.3 What are the 25th, 50th and 75th percentile values for the revenue per transaction?
```sql
WITH revenue_per_transaction as
(
	SELECT txn_id
	,sum(qty * price - discount) as actual_revenue
	FROM balanced_tree.sales
	GROUP BY 1
)
SELECT percentile_cont(0.25) within group (order by actual_revenue) as percentile_25
,percentile_cont(0.5) within group (order by actual_revenue) as percentile_50
,percentile_cont(0.75) within group (order by actual_revenue) as percentile_75
FROM revenue_per_transaction;
```

_Result:_

![image](https://github.com/user-attachments/assets/b2fb84fd-9bf2-4c02-9596-7b8a341b9e55)


#### 2.4 What is the average discount value per transaction?
```sql
WITH discount_per_transaction as
(
	SELECT txn_id
	,sum(discount) as total_discounts
	FROM balanced_tree.sales
	GROUP BY 1
)
SELECT round(avg(total_discounts),2)
FROM discount_per_transaction;
```

_Result:_

![image](https://github.com/user-attachments/assets/f7a77ffc-508d-4086-8051-b3d05ee74f17)


#### 2.5 What is the percentage split of all transactions for members vs non-members?
```sql
WITH transaction_by_member as
(
	SELECT count(distinct txn_id) as total_transaction
	,count(distinct txn_id) filter (where member = true) as txn_members
	,count(distinct txn_id) filter (where member = false) as txn_non_members
	FROM balanced_tree.sales
)
SELECT 
round((txn_members * 100.0 / total_transaction),2) || '%' as percentage_txn_members
,round((txn_non_members * 100.0 / total_transaction),2) || '%' as percentage_txn_non_members
FROM transaction_by_member;
```

_Result:_

![image](https://github.com/user-attachments/assets/42eb94ab-1e1a-477f-83da-b1f5260268bf)


#### 2.6 What is the average revenue for member transactions and non-member transactions?
```sql
WITH transaction_by_member as
(
	SELECT txn_id
	,member
	,sum(qty * price - discount) as actual_revenue
	FROM balanced_tree.sales
	GROUP BY 1,2
)
SELECT member
,round(avg(actual_revenue),2) as avg_revenue
FROM transaction_by_member
GROUP BY 1
ORDER BY 1 DESC;

WITH transaction_by_member as
(
	SELECT 
	member
	,sum(qty * price - discount) as actual_revenue
	FROM balanced_tree.sales
	GROUP BY 1
)
SELECT member
,round(avg(actual_revenue),2) as avg_revenue
FROM transaction_by_member
GROUP BY 1
ORDER BY 1 DESC;
```

_Result:_

![image](https://github.com/user-attachments/assets/a6240bc1-e826-4a7d-a613-47bdef5877b6)


### 3. Product Analysis
#### 3.1 What are the top 3 products by total revenue before discount?
```sql
SELECT prod_id
,sum(qty * price) as total_revenue_before_discount
FROM balanced_tree.sales
GROUP BY 1
ORDER BY 2 DESC
LIMIT 3;
```

_Result:_

![image](https://github.com/user-attachments/assets/42cb49f2-1ef9-458f-8338-e16ee1cd71e3)


#### 3.2 What is the total quantity, revenue and discount for each segment?
```sql
SELECT pd.segment_id
,pd.segment_name
,sum(s.qty) as total_quantity
,sum(s.qty * s.price - s.discount) as total_actual_revenue
,sum(s.discount) as total_discount
FROM balanced_tree.sales s
JOIN balanced_tree.product_details pd on s.prod_id = pd.product_id
GROUP BY 1,2
ORDER BY 4 DESC;
```

_Result:_

![image](https://github.com/user-attachments/assets/d345288f-69c0-4720-83fd-4c88cf69fd8c)


#### 3.3 What is the top selling product for each segment?
```sql
WITH selling_segment as
(
	SELECT pd.segment_name
	,pd.product_name
	,sum(s.qty) as total_quantity
	,row_number() over (partition by pd.segment_name order by sum(s.qty) DESC) as selling_ranking
	FROM balanced_tree.sales s
	JOIN balanced_tree.product_details pd on s.prod_id = pd.product_id
	GROUP BY 1,2
)
SELECT *
FROM selling_segment
WHERE selling_ranking <=3;
```

_Result:_

![image](https://github.com/user-attachments/assets/844fb5bc-64c0-4ad3-b50b-92a9010b2ceb)


#### 3.4 What is the total quantity, revenue and discount for each category?
```sql
SELECT pd.category_name
,sum(s.qty) as total_quantity
,sum(s.qty * s.price - s.discount) as total_actual_revenue
,sum(s.discount) as total_discounts
FROM balanced_tree.sales s
JOIN balanced_tree.product_details pd on s.prod_id = pd.product_id
GROUP BY 1
ORDER BY 3 DESC;
```

_Result:_

![image](https://github.com/user-attachments/assets/1862975d-8f19-484b-8674-a5dcdb952941)

#### 3.5 What is the top selling product for each category?
```sql
WITH selling_category as
(
	SELECT pd.category_name
	,pd.product_name
	,sum(s.qty) as total_quantity
	,row_number () over (partition by pd.category_name order by sum(s.qty) DESC) as selling_ranking
	FROM balanced_tree.sales s
	JOIN balanced_tree.product_details pd on s.prod_id = pd.product_id
	GROUP BY 1,2
)
SELECT *
FROM selling_category
WHERE selling_ranking <= 3;
```

_Result:_

![image](https://github.com/user-attachments/assets/c5e0a043-d61c-48a0-86fb-5f6c11ef411e)



#### 3.6 What is the percentage split of revenue by product for each segment?
```sql
WITH revenue_by_product as
(
	SELECT pd.segment_name
	,s.prod_id
	,sum(s.qty * s.price - s.discount) as total_actual_revenue
	FROM balanced_tree.sales s
	JOIN balanced_tree.product_details pd on s.prod_id = pd.product_id
	GROUP BY 1,2
)
,revenue_by_segment as
(
	SELECT segment_name
	,sum(total_actual_revenue) as total_revenue
	FROM revenue_by_product
	GROUP BY 1
)
SELECT rbs.segment_name
,round((rbp.total_actual_revenue *100 / rbs.total_revenue),2) as percent_revenue
FROM revenue_by_segment rbs
JOIN revenue_by_product rbp on rbs.segment_name = rbp.segment_name
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/58e1cab9-78a3-40d4-a827-f01dd27d99bd)


#### 3.7 What is the percentage split of revenue by segment for each category?
```sql
WITH revenue_by_segment as
(
	SELECT pd.category_name	
	,pd.segment_id
	,sum(s.qty * s.price - s.discount) as total_actual_revenue
	FROM balanced_tree.sales s
	JOIN balanced_tree.product_details pd on s.prod_id = pd.product_id
	GROUP BY 1,2
)
,revenue_by_category as
(
	SELECT category_name	
	,sum(total_actual_revenue) as total_revenue	
	FROM revenue_by_segment
	GROUP BY 1	
)
SELECT rbc.category_name
,round((rbs.total_actual_revenue * 100 / rbc.total_revenue),2) as percent_revenue
FROM revenue_by_category rbc
JOIN revenue_by_segment rbs on rbc.category_name = rbs.category_name
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/e7abab7d-68d1-4a72-ade6-4035ef80a084)


#### 3.8 What is the percentage split of total revenue by category?
```sql
WITH revenue_by_cate as
(
	SELECT pd.category_name
	,sum(s.qty * s.price - s.discount) as total_actual_revenue
	FROM balanced_tree.sales s
	JOIN balanced_tree.product_details pd on s.prod_id = pd.product_id
	GROUP BY 1
)
, total_revenue as
(
	SELECT sum(total_actual_revenue) as grand_total_revenue
	FROM revenue_by_cate
)
SELECT rbc.category_name
,round((total_actual_revenue * 100 / grand_total_revenue),2) as percent_revenue
FROM revenue_by_cate rbc
CROSS JOIN total_revenue tr;
```

_Result:_

![image](https://github.com/user-attachments/assets/7bca1fbf-2ab9-4033-a3b9-9f2eac1cbc82)


#### 3.9 What is the total transaction “penetration” for each product? 
```sql
SELECT prod_id
,count(txn_id) filter (where qty >= 1) / count(txn_id) as penetration
FROM balanced_tree.sales
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/d0ae617c-4dea-43d7-b71d-287a442e50f6)


#### 3.10 What is the most common combination of at least 1 quantity of any 3 products in a 1 single transaction?
```sql
WITH product_combination as
(
	SELECT txn_id
	,array_agg(prod_id) as products
	FROM balanced_tree.sales
	WHERE qty >= 1
	GROUP BY 1
)
SELECT products
,count(*) as combo_count
FROM product_combination
WHERE array_length(products,1) >= 3 -- Ensure at least 3 products; 1 means one-dimensional array
GROUP BY 1
ORDER BY 2 DESC
LIMIT 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/e680cbcd-3c05-47d7-8576-f11697b7fd03)


### 4. Reporting Challenge
#### 4.1. Monthly Sales Analysis Report
```sql
WITH sales_metrics as
(
	SELECT to_char(start_txn_time,'MM-YYYY') as month
	,qty
	,qty * price as revenue_before_discount
	,discount
	FROM balanced_tree.sales		
)
SELECT month
,sum(qty) as total_quantity_sold
,sum(revenue_before_discount) as total_revenues_before_discount
,sum(discount) as total_discounts
FROM sales_metrics
WHERE month = '01-2021' -- Change this dynamically for different months
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/934cbcb9-8210-4a46-88ca-e416af49c608)


#### 4.2 Monthly Transaction Analysis Report
```sql
WITH monthly_metrics as
(
	SELECT to_char(start_txn_time, 'MM-YYYY') as month
	,txn_id
	,member	
	,count(distinct prod_id) as unique_product_purchased
	,sum(qty * price - discount) as actual_revenue
	,sum(discount) as total_discounts
	FROM balanced_tree.sales
	GROUP BY 1,2,3
)
SELECT month
,count(distinct txn_id) as total_unique_transactions
,round(avg(unique_product_purchased),2) as avg_unique_product_purchased
,percentile_cont(0.25) within group (order by actual_revenue) as percentile_25
,percentile_cont(0.5) within group (order by actual_revenue) as percentile_50
,percentile_cont(0.75) within group (order by actual_revenue) as percentile_75
,round(avg(total_discounts),2) as avg_discount
,round((count(distinct txn_id) filter (where member = true) * 100.0 / count(distinct txn_id)),2) || '%' as percentage_txn_members
,round((count(distinct txn_id) filter (where member = false) * 100.0 / count(distinct txn_id)),2) || '%' as percentage_txn_non_members
,round(avg(actual_revenue),2) as avg_revenue
FROM monthly_metrics
WHERE month = '01-2021' -- Change this dynamically for different months
GROUP BY 1
ORDER BY 1;
```

_Result:_

![image](https://github.com/user-attachments/assets/9c403832-e3eb-4f27-9240-7b0b91fd9375)


Thank you for stopping by, and I'm pleased to connect with you, my new friend!

**Please do not forget to FOLLOW and star ⭐ the repository if you find it valuable.**

Wish you a day filled with happiness and energy!

Warm regards,

Hien Moon
