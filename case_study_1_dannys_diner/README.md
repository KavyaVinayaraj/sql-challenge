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

**3. What was the first item from the menu purchased by each customer?**

```sql
with sales_rank as
 (select s.customer_id, m.product_name, s.order_date,
  dense_rank() over (partition by s.customer_id order by s.order_date) as date_rank
  from sales s
  join menu m on m.product_id = s.product_id
  )
  select distinct customer_id, product_name, order_date
  from sales_rank
  where date_rank = 1;
 ```
**Action:** With a CTE ranked the customers order by date, keeping the earliest. Used `DENSE_RANK()` instead of `ROW_NUMBER()` so that all the all the orders bought on the same day shows up, rather than one picked arbitrarily 
 
**Results:** A: sushi and curry B: curry C: ramen

**Insights:** A's first visit had two items on the same day, so there is no single "first" item. The data genuinely can't separate them, and the query reflects that rather than hiding it.

**4. What is the most purchased item on the menu and how many times was it purchased by all customers?**
```sql
select product_name, count(product_name) as product_count
  from menu m
  join sales s on s.product_id = m.product_id
  group by m.product_name
  order by product_count desc
  limit 1;
```
**Action:** Joined the `menu` and `sales` and counted sales per item across all customers, ordered descending, and took the topmost

**Results:** ramen, ordered 8 times

**Insights**: ramen is the clear favourite

**5. Which item was the most popular for each customer?**
```sql
  with popular_dish as
  (select s.customer_id, m.product_name, count(m.product_id) as purchase_count, 
  dense_rank() over (partition by s.customer_id order by count(m.product_id) desc) as product_count
  from sales s 
  join menu m on m.product_id = s.product_id
  group by s.customer_id, product_name, m.product_id
  )
  select customer_id, product_name , purchase_count
  from popular_dish
  where product_count =1;
  ```
**Action:** `DENSE_RANK` on each customer's order the count per item for each customer, ordered descending, took the top. counted each customer's orders per item, then ranked within each customer with `DENSE_RANK`.

**Result:** A - ramen (3), B - curry, sushi, ramen (2 each), C - ramen (3)

**Insights:** B has no real favourite, all three items are tied at two orders. A and C both lean heavily on ramen.

**6. Which item was purchased first by the customer after they became a member?**
```sql
with cte as (select s.customer_id, m.product_name, s.order_date, mm.join_date,
  dense_rank() over (partition by s.customer_id order by s.order_date) as rnk
  from sales s
  join menu m on s.product_id = m.product_id
  join members mm on s.customer_id = mm.customer_id
  where mm.join_date < s.order_date
  )
  select customer_id, product_name
  from cte
  where rnk =1;
```

**Action:** joined `members`, kept orders placed after `join_date` and took the earliest

**Result:** A: ramen, B: sushi

**Note:** I read "after they became a member" as strictly after the join date, so an order on the join day itself does not count. If the join day is instead counted as "after", A's answer becomes curry. I went with the stricter reading. Either is defensible, and it is exactly the kind of definition I would confirm with a stakeholder before reporting it.

**Insight:** only A and B appear, since C never joined. That is the join behaving as intended, not C being lost by accident.

**7. Which item was purchased just before the customer became a member?**
```sql
with cte as (select s.customer_id, m.product_name, s.order_date,
  dense_rank() over (partition by s.customer_id order by s.order_date desc) as rnk
  from sales s
  join menu m on m.product_id = s.product_id
  join members mm on mm.customer_id = s.customer_id
  where mm.join_date > s.order_date)
  select customer_id, product_name
  from cte 
  where rnk = 1;
```
**Action:** joined `members` and ranked the orders in descending by `order_date` and kept only the orders before the `join_date` and took the top

**Result:** A: sushi and curry B: sushi

**Insights:** A ordered two items on the last pre-membership day and B ordered one item.

**8. What is the total items and amount spent for each member before they became a member?**
``` sql
select s.customer_id, sum(m.price) as total_amount, count(product_name) as total_items 
  from sales s 
  join menu m on s.product_id = m.product_id
  join members mm on mm.customer_id = s.customer_id
  where mm.join_date > order_date
  group by s.customer_id;
```
**Action:** summed price and counted items for orders placed before each member's join date

**Results:** A: $25 (2 items) B: $40 (3 items)

**Insights:** both were already spending before they joined. That baseline matters: would be needed before claiming the loyalty programme caused any later change in behaviour

**9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?**
```sql
with points as 
    ( select *, 
      case when product_name = 'sushi' then price * 20
      when product_name != 'sushi' then price * 10
      end as item_points
    from  menu )
    select s.customer_id, sum(item_points) as points
    from sales s 
    join points p on p.product_id = s.product_id
    group by s.customer_id;
```
**Action:** built a `CASE` that doubles points for sushi and applies the standard rate to everything else, then summed per customer

**Result:** A: 860, B: 940 C: 360

**Note:**  Q9 is a hypothetical points calculation across all customers, so C is included even though C never joined the programme. The membership condition only matters in Q10.

**Insights:** B leads on points despite spending slightly less than A, because B buys more sushi and the multiplier rewards that. A points scheme quietly changes which customer looks "best".

**10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi how many points do customer A and B have at the end of January?**
```sql
select s.customer_id, sum(
case when s.order_date between mm.join_date and date_add(mm.join_date, interval 6 day) then price * 20
when m.product_name = 'sushi' then price*20
when m.product_name != 'sushi' then price *10
end 
) as total_points	
from sales s
join menu m on m.product_id = s.product_id
join members mm on mm.customer_id = s.customer_id
where s.order_date <= '2021-01-31' and 
s.customer_id in ('A','B')  
group by s.customer_id;
```
**Action:** every January order earns points. Orders inside the customer's first member week earn double on everything. All other orders follow the normal rule, sushi double and everything else standard. I deliberately did not filter out pre-join orders, because those still earn normal points and the question asks for the full January total.

**Result:** A: 1370, B: 820

**Insights:** A leads, because A's bonus week orders had three ramen orders and a curry, all doubled. The timing of the bonus relative to a customer's ordering pattern matters more than the bonus itself, two customers can get very different value from the same offer.
