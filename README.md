# For-Fetch---home-exercise
Fetch Rewards Coding Exercise - Analytics Engineer

First: Review Existing Unstructured Data and Diagram a New Structured Relational Data Model
Review the 3 sample data files provided below. Develop a simplified, structured, relational diagram to represent how you would model the data in a data warehouse. The diagram should show each table’s fields and the joinable keys. You can use pencil and paper, readme, or any digital drawing or diagramming tool with which you are familiar. If you can upload the text, image, or diagram into a git repository and we can read it, we will review it!
Relational data model for DW
1. For Users Table
   
Field Name	Data Type	Description

user_id (PK)	STRING	Primary Key and unique ID of table for user

active	BOOLEAN	Indicates if user is active if not false

created_date	DATETIME	Account creation date

last_login	DATETIME	Last login timestamp

role	STRING	User's role

sign_up_source	STRING	The source for sign up

state	STRING	State code of user

2. For Receipts Table
   
Field Name	Data Type	Description

receipt_id (PK)	STRING	Receipt ID 

user_id (FK)	STRING	References Users(user_id)

purchase_date	DATE	Date of purchase

create_date	DATETIME	Date receipt was created

scanned_date	DATETIME	Date scanned

modify_date	DATETIME	Date of last modification

status	STRING	Receipt status 

total_spent	DECIMAL	Total amount spent

points_earned	DECIMAL	Points awarded for receipt

bonus_points_earned	INTEGER	Bonus points awarded

bonus_reason	TEXT	Reason for bonus points

3. Receipt_Items Table
   
Field Name	Data Type	Description

receipt_item_id (PK)	STRING	Auto-generated value

receipt_id (FK)	STRING	References Receipts(receipt_id)

barcode (FK)	STRING	References Brands(barcode)

description	TEXT	Item description

item_price	DECIMAL	Price of the item

final_price	DECIMAL	Final price after any discounts

quantity_purchased	INTEGER	Number of items bought

user_flagged_barcode	STRING	Optional alternate barcode

user_flagged_price	DECIMAL	User-flagged price

user_flagged_quantity	INTEGER	User-flagged quantity

needs_review	BOOLEAN	For manual review

4. Brands Table
   
Field Name	Data Type	Description

barcode (PK)	STRING	Barcode of the item

brand_code	STRING	Brand code

name	STRING	Brand name

category	STRING	Product category

category_code	STRING	Category code

cpg_id	STRING	ID of the Consumer Packaged Goods company

top_brand	BOOLEAN	To indicate if this is a top brand

Entity relationships

Users.user_id ⟶ Receipts.user_id

Receipts.receipt_id ⟶ Receipt_Items.receipt_id

Receipt_Items.barcode ⟶ Brands.barcode

ERD:

![ERD](https://github.com/user-attachments/assets/8ccc8f0f-12b5-45d9-aab7-922c4609b4b4)

Second: Write queries that directly answer predetermined questions from a business stakeholder
Write SQL queries against your new structured relational data model that answer at least two of the following bullet points below of your choosing. Commit them to the git repository along with the rest of the exercise.

Note: When creating your data model be mindful of the other requests being made by the business stakeholder. If you can capture more than two bullet points in your model while keeping it clean, efficient, and performant, that benefits you as well as your team.

1.What are the top 5 brands by receipts scanned for most recent month?

WITH latest_month AS (
  SELECT DATE_TRUNC('month', MAX(scanned_date)) AS month_start
  FROM Receipts
),
recent_receipts AS (
  SELECT r.receipt_id
  FROM Receipts r
  JOIN latest_month lm ON DATE_TRUNC('month', r.scanned_date) = lm.month_start
),
brand_receipts AS (
  SELECT ri.barcode, COUNT(DISTINCT ri.receipt_id) AS receipt_count
  FROM Receipt_Items ri
  JOIN recent_receipts rr ON ri.receipt_id = rr.receipt_id
  GROUP BY ri.barcode
)
SELECT b.name AS brand_name, br.receipt_count
FROM brand_receipts br
JOIN Brands b ON br.barcode = b.barcode
ORDER BY br.receipt_count DESC
LIMIT 5;

2. How does the ranking of the top 5 brands by receipts scanned for the recent month compare to the ranking for the previous month?

WITH months AS (
  SELECT
    DATE_TRUNC('month', scanned_date) AS month_start
  FROM Receipts
  ORDER BY month_start DESC
  LIMIT 2
),
ranked_receipts AS (
  SELECT
    DATE_TRUNC('month', r.scanned_date) AS month,
    ri.barcode,
    COUNT(DISTINCT r.receipt_id) AS receipt_count
  FROM Receipts r
  JOIN Receipt_Items ri ON r.receipt_id = ri.receipt_id
  WHERE DATE_TRUNC('month', r.scanned_date) IN (SELECT month_start FROM months)
  GROUP BY month, ri.barcode
),
ranked_brands AS (
  SELECT
    rb.month,
    b.name AS brand_name,
    rb.receipt_count,
    RANK() OVER (PARTITION BY rb.month ORDER BY rb.receipt_count DESC) AS rank
  FROM ranked_receipts rb
  JOIN Brands b ON rb.barcode = b.barcode
)
SELECT *
FROM ranked_brands
WHERE rank <= 5
ORDER BY month DESC, rank;

3. When considering average spend from receipts with 'rewardsReceiptStatus’ of ‘Accepted’ or ‘Rejected’, which is greater?

SELECT
  status AS rewards_receipt_status,
  AVG(total_spent) AS avg_spend
FROM Receipts
WHERE status IN ('Accepted', 'Rejected')
GROUP BY status;

4. When considering total number of items purchased from receipts with 'rewardsReceiptStatus’ of ‘Accepted’ or ‘Rejected’, which is greater?

SELECT
  r.status AS rewards_receipt_status,
  SUM(ri.quantity_purchased) AS total_items
FROM Receipts r
JOIN Receipt_Items ri ON r.receipt_id = ri.receipt_id
WHERE r.status IN ('Accepted', 'Rejected')
GROUP BY r.status;

5. Which brand has the most spend among users who were created within the past 6 months?

WITH recent_users AS (
  SELECT user_id
  FROM Users
  WHERE created_date >= CURRENT_DATE - INTERVAL '6 months'
),
recent_receipts AS (
  SELECT receipt_id
  FROM Receipts
  WHERE user_id IN (SELECT user_id FROM recent_users)
),
brand_spend AS (
  SELECT ri.barcode, SUM(ri.final_price) AS total_spend
  FROM Receipt_Items ri
  WHERE ri.receipt_id IN (SELECT receipt_id FROM recent_receipts)
  GROUP BY ri.barcode
)
SELECT b.name AS brand_name, bs.total_spend
FROM brand_spend bs
JOIN Brands b ON bs.barcode = b.barcode
ORDER BY bs.total_spend DESC
LIMIT 1;

6. Which brand has the most transactions among users who were created within the past 6 months?

WITH recent_users AS (
  SELECT user_id
  FROM Users
  WHERE created_date >= CURRENT_DATE - INTERVAL '6 months'
),
recent_receipts AS (
  SELECT receipt_id
  FROM Receipts
  WHERE user_id IN (SELECT user_id FROM recent_users)
),
brand_txns AS (
  SELECT ri.barcode, COUNT(DISTINCT ri.receipt_id) AS txn_count
  FROM Receipt_Items ri
  WHERE ri.receipt_id IN (SELECT receipt_id FROM recent_receipts)
  GROUP BY ri.barcode
)
SELECT b.name AS brand_name, bt.txn_count
FROM brand_txns bt
JOIN Brands b ON bt.barcode = b.barcode
ORDER BY bt.txn_count DESC
LIMIT 1;

Third: Evaluate Data Quality Issues in the Data Provided
Using the programming language of your choice (SQL, Python, R, Bash, etc...) identify as many data quality issues as you can. We are not expecting a full blown review of all the data provided, but instead want to know how you explore and evaluate data of questionable provenance.

Commit your code and findings to the git repository along with the rest of the exercise.

1.To find NULL or Duplicate Barcodes in Brands
--NULL BARCODES
SELECT COUNT(*) AS null_barcodes
FROM brands
WHERE barcode IS NULL;

--DUPLICATE BARCODES
SELECT barcode, COUNT(*) AS occurrences
FROM brands
GROUP BY barcode
HAVING COUNT(*) > 1;

2.To identify Receipts with Data Issues
-- Null user_id
SELECT COUNT(*) AS null_user_id
FROM receipts
WHERE user_id IS NULL;

-- Non-numeric total_spent
SELECT COUNT(*) AS null_or_invalid_total_spent
FROM receipts
WHERE total_spent IS NULL OR total_spent !~ '^[0-9]+(\.[0-9]+)?$';  -- PostgreSQL regex syntax

-- Negative total_spent
SELECT COUNT(*) AS negative_total_spent
FROM receipts
WHERE total_spent::NUMERIC < 0;

-- Unrecognized rewards status
SELECT DISTINCT rewards_status
FROM receipts
WHERE rewards_status NOT IN ('Accepted', 'Rejected', 'Pending')

3.Check for User Data Issues
-- Duplicate user IDs
SELECT id, COUNT(*) AS count
FROM users
GROUP BY id
HAVING COUNT(*) > 1;

-- Null creation dates
SELECT COUNT(*) AS null_created_date
FROM users
WHERE created_date IS NULL;

-- Null state field
SELECT COUNT(*) AS null_state
FROM users
WHERE state IS NULL;

-- Malformed date (e.g., missing 'T')
SELECT COUNT(*) AS malformed_date
FROM users
WHERE created_date NOT LIKE '%T%';

Data Quality issues:
1. Brands
   
Issue	                                  Description
Null barcodes	                Some entries are missing unique barcode identifiers
Duplicate barcodes	        Some brands share the same barcode which violates uniqueness
Missing brand names	        A few entries are missing the name field

2. Receipts
   
Issue	                                Description
Null userId	                Some receipts not linked to a user
Null or negative totalSpent	Invalid or corrupt transaction totals
Unknown rewardsReceiptStatus	Some statuses outside Accepted, Rejected, or Pending

4. users.json.gz
Issue	                                Description
Duplicate_id	                        Multiple user records share the same _id
Null createdDate	                Some user records lack account creation dates

4.Fourth: Communicate with Stakeholders
Construct an email or slack message that is understandable to a product or business leader who isn’t familiar with your day to day work. This part of the exercise should show off how you communicate and reason about data with others. Commit your answers to the git repository along with the rest of your exercise.

What questions do you have about the data?
How did you discover the data quality issues?
What do you need to know to resolve the data quality issues?
What other information would you need to help you optimize the data assets you're trying to create?
What performance and scaling concerns do you anticipate in production and how do you plan to address them?

Answer: 
Hi team,

I have completed the first pass of the data review and modeling exercise. Below is a summary of key findings and some follow-up questions to ensure we're aligned on next steps.
I have few questions on the data, please help me here.
Brand-to-product mapping — Can we assume one barcode maps to a single brand, or are shared barcodes expected?
Receipt scanning rules — What conditions determine when a receipt is marked as “Accepted” or “Rejected”?
Time zones and timestamps — Are all timestamps in UTC, or do they reflect local times?
I ran a series of SQL tests and found data quality issues.
Some brand entries are missing barcodes or names, and there are duplicates in the barcode field.
Several receipts lack associated user IDs, have null or negative totalSpend.
The user dataset has multiple user records with same user_ids and some records lack account creation dates.
In order to resolve the above issues, I would need a cleaner version of the dataset.
Performance & Scaling Concerns for production that I anticipate if data is not corrected is that we will need to index large tables on user ID, receipt date, and barcode for analytics performance which is a lot of effort and time spent.

I will be happy to walk through the schema once we clean up the data.

Thanks!
Rakshitha
