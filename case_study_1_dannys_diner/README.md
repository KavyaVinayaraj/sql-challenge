# **Case Study #1 - Danny's Diner**
Solved as part of [8 Week SQL Challenge](https://8weeksqlchallenge.com/case-study-1/).

## **The Problem**
Danny runs a small Japanese restaurant with three menu items and a young loyalty programme. He wants to understand his customers: how much they spend, how often they visit, what they order, and whether membership changes their behaviour. The data is three tables, sales, menu, and members.

## **Tools**
MySQL

---
**1. What is the total amount each customer spent?**

```sql
select s.customer_id, sum(m.price) as total_spent_per_customer
from sales s
join menu m on m.product_id = s.product_id
group by s.customer_id
order by total_spent_per_customer desc;
```
**Approach:** `sales` records what was ordered but not the price, so I joined it to `menu` and summed price per customer.

**Result**: A:$76, B:$74, C:$36

**Insight:** A and B spend almost identically. C spends roughly half compared to the other customers.

**2. How many days has each customer visited the restaurant?**
```sql
select customer_id, count(distinct order_date) as number_of_visits
from sales
group by customer_id
order by customer_id;
```
**Approach:** counted distinct `order_date` as per visit, so that multiple orders on the same day are counted as one visit

**Result:** A:4, B:6, C:2

**Insight:** B visits most but A spends most, so visits and value don't line up. Worth remembering before treating "number of visits" as a proxy for how valuable a customer is.
