# Practice 4: E-commerce Orders (Sales)


**Folder:** `practice-4-ecommerce-orders/`

## The request from the manager

> "Our revenue looks fine, but we give a lot of discounts and I don't think we're earning what we should. Where do we make money, where do we lose it, and who are our best customers? Also, what should we do before next year's sales season?"

## The scenario

An online shop sells 12 products in 4 categories to customers in 6 cities. You have three files for the year 2025:

- `orders.csv`: one row per order (one product per order)
- `products.csv`: one row per product (category, price, cost)
- `customers.csv`: one row per customer (signup date, city, acquisition channel)

## Columns

**orders.csv**
- `order_id`, `order_date`, `customer_id`, `product_id`
- `quantity`: units ordered
- `discount_pct`: discount on the order
- `status`: Delivered, Returned or Cancelled
- `payment_method`

**products.csv**
- `product_id`, `name`, `category`
- `price`: selling price per unit before discount
- `cost`: what the shop pays per unit

**customers.csv**
- `customer_id`, `signup_date`, `city`
- `channel`: how the customer found the shop (Organic, Ads, Referral)

## Setup (run once)

Creates the three CSV files (about 6,300 orders). The data has messy dates, text, discount formats, impossible values, unknown products, missing customers and duplicates.

```python
import random
import numpy as np
import pandas as pd

random.seed(5)
np.random.seed(5)

products = pd.DataFrame({
    "product_id": [f"P{i:02d}" for i in range(1, 13)],
    "name": ["Headphones", "Phone Case", "Smart Speaker", "Desk Lamp", "Cookware Set", "Vacuum Bottle",
             "T-Shirt", "Jacket", "Sneakers", "Yoga Mat", "Dumbbell Set", "Fitness Band"],
    "category": ["Electronics"] * 3 + ["Home"] * 3 + ["Clothing"] * 3 + ["Sports"] * 3,
    "price": [79.0, 15.0, 59.0, 35.0, 90.0, 25.0, 20.0, 85.0, 70.0, 30.0, 60.0, 45.0],
    "cost": [48.0, 5.0, 38.0, 29.0, 76.0, 20.0, 7.0, 40.0, 33.0, 12.0, 35.0, 24.0],
})

cities = ["Berlin", "Rome", "Madrid", "Paris", "Vienna", "Warsaw"]
customers = pd.DataFrame({
    "customer_id": [f"C{i:04d}" for i in range(1, 401)],
    "signup_date": pd.to_datetime("2023-01-01") + pd.to_timedelta(np.random.randint(0, 1100, 400), unit="D"),
    "city": [random.choice(cities) for _ in range(400)],
    "channel": [random.choice(["Organic", "Ads", "Referral"]) for _ in range(400)],
})
customers["signup_date"] = customers["signup_date"].dt.strftime("%Y-%m-%d")

weights = np.random.lognormal(0, 1, 400)
weights = weights / weights.sum()
pop = np.array([3, 4, 2, 3, 1, 3, 4, 2, 3, 3, 2, 2], dtype=float)
pop = pop / pop.sum()

rows = []
oid = 1000
for d in pd.date_range("2025-01-01", "2025-12-31"):
    lam = 14 * (1.8 if d.month in (11, 12) else 1.0) * (1.25 if d.dayofweek >= 5 else 1.0)
    for _ in range(np.random.poisson(lam)):
        oid += 1
        c = np.random.choice(customers["customer_id"], p=weights)
        p = np.random.choice(products["product_id"], p=pop)
        cat = products.loc[products["product_id"] == p, "category"].iloc[0]
        qty = int(np.random.choice([1, 2, 3, 4], p=[0.6, 0.25, 0.1, 0.05]))
        if cat == "Home" and d.month >= 7:
            disc = random.choice([0.2, 0.25, 0.3])
        else:
            disc = random.choice([0, 0, 0, 0.05, 0.1])
        r_ret = 0.15 if cat == "Clothing" else 0.04
        u = random.random()
        status = "Returned" if u < r_ret else ("Cancelled" if u < r_ret + 0.04 else "Delivered")
        pay = random.choice(["Card", "Card", "PayPal", "Transfer"])
        rows.append([f"O{oid}", d.strftime("%Y-%m-%d"), c, p, qty, disc, status, pay])

orders = pd.DataFrame(rows, columns=["order_id", "order_date", "customer_id", "product_id",
                                     "quantity", "discount_pct", "status", "payment_method"])
n = len(orders)
orders["discount_pct"] = orders["discount_pct"].astype(object)
for i in random.sample(range(n), 70):
    orders.loc[i, "order_date"] = pd.to_datetime(orders.loc[i, "order_date"]).strftime("%d.%m.%Y")
for i in random.sample(range(n), 90):
    s = orders.loc[i, "status"]
    orders.loc[i, "status"] = random.choice([s.lower(), s.upper(), " " + s])
for i in random.sample(range(n), 120):
    orders.loc[i, "discount_pct"] = f"{int(float(orders.loc[i, 'discount_pct']) * 100)}%"
for i in random.sample(range(n), 12):
    orders.loc[i, "quantity"] = -1
for i in random.sample(range(n), 45):
    orders.loc[i, "customer_id"] = None
for i in random.sample(range(n), 15):
    orders.loc[i, "product_id"] = "P99"
orders = pd.concat([orders, orders.sample(25, random_state=2)]).sample(frac=1, random_state=9).reset_index(drop=True)

for i in random.sample(range(400), 40):
    customers.loc[i, "city"] = random.choice([customers.loc[i, "city"].upper(), " " + customers.loc[i, "city"].lower()])
products["price"] = products["price"].astype(object)
products.loc[products["product_id"] == "P05", "price"] = "90,00"

products.to_csv("products.csv", index=False)
customers.to_csv("customers.csv", index=False)
orders.to_csv("orders.csv", index=False)
```

## Problems

### 0. Plan first
Before any code, write:
- 3 precise questions from the manager's request (with numbers, time period and what you will measure)
- the formulas you will use for revenue and profit, and which order statuses count (write one sentence on why)

### 1. Find the problems
Load all three files. List every data problem you can find (at least 8). For each one, write how you found it and how many rows are affected.

### 2. Clean the tables
Write `clean_orders(df)`, `clean_products(df)` and `clean_customers(df)`. For every decision (drop, fill, convert), write one sentence on why. Pay special attention to:
- the discount column (it has more than one format)
- the date column
- orders with unknown products or no customer
- duplicates

Check:
- `order_date` is datetime, no missing dates
- `discount_pct` is numeric between 0 and 1
- no negative `quantity`
- `status` has exactly 3 unique values
- no duplicate `order_id`

### 3. Build one analysis table
Merge the three tables. Add columns `revenue` (after discount), `cost_total`, `profit` and `margin` (profit divided by revenue). Decide how to treat Returned and Cancelled orders, and write one sentence on why.

### 4. Revenue over time
Monthly revenue and monthly profit in one chart. Describe the pattern in one sentence. Then compare weekdays vs weekends: is the difference big?

### 5. Where do we make and lose money?
Profit and margin by category. Then split each category into first half (January to June) and second half (July to December) of the year. Does anything change? Support it with numbers and one chart.

### 6. The discount question
- What is the average discount per category?
- Is there a category where discounts push the margin to zero or below? Show it with a chart.
- Roughly how much profit did the shop lose because of that? Explain how you calculated it.

### 7. Returns
Return rate by category and by city. Which is highest? Is the difference big enough to matter, or could it be chance? (Think about how many orders each group has.)

### 8. Best customers
- Top 10 customers by total profit (not revenue). Do they differ from the top 10 by revenue?
- What share of total profit comes from the top 10% of customers?
- Customers per acquisition channel: average profit per customer, and average number of orders per customer.

### 9. Repeat customers
For each customer, compute number of orders, first order date and last order date. What percentage of customers ordered more than once? Does the repeat rate differ by channel? Draw one chart.

### 10. Report function, dashboard and recommendation
Write `category_report(df, category)` that returns a dict (revenue, profit, margin, return rate, average discount, number of orders) and saves a chart `report_<category>.png`. Loop over all categories and collect the results into one DataFrame.

Then make a 2×2 figure saved as `dashboard.png`, and write a short report (5 to 8 sentences) for the manager:
- where we make and lose money
- who our best customers are
- one recommendation for next year's sales season, with a number to support it

## New tools you will meet

- Cleaning mixed formats in one column (`.astype(str)`, `.str.replace`, `.where`)
- Calculated columns and conditional logic
- `groupby` with two keys and `.unstack()`
- `.nlargest()` and share-of-total calculations
- `groupby("customer_id").agg(...)` with named aggregations
- Thinking about sample size when comparing rates

## Rules

- Spend about 20 minutes on a problem before asking for help.
- Ask for a hint, and I will guide you step by step.
- Do not read the setup code before you finish. Treat the data as if a company gave it to you.
- Keep the notebook in one folder with one heading per problem.
- Write your conclusion sentence under every chart.

## Notion tasks (short)

- [ ] Plan and find data problems
- [ ] Clean three tables
- [ ] Build analysis table
- [ ] Revenue and profit over time
- [ ] Categories, discounts, returns
- [ ] Customers and repeat rate
- [ ] Report, dashboard and recommendation

## Checklist

- [ ] Setup: generate the three CSV files
- [ ] 0. Plan and formulas
- [ ] 1. Find data problems
- [ ] 2. Clean the tables
- [ ] 3. Build analysis table
- [ ] 4. Revenue over time
- [ ] 5. Where we make and lose money
- [ ] 6. The discount question
- [ ] 7. Returns
- [ ] 8. Best customers
- [ ] 9. Repeat customers
- [ ] 10. category_report() + dashboard + recommendation