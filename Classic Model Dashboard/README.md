
# Classic Models Sales Overview (MySQL and PowerBI)

## Project Description 

In today's competitive market, data-driven decision-making is crucial for business success. The "Classic Models" database, a repository of scale model car sales data, presents an invaluable resource for understanding and enhancing business performance in this nieche sector. This project aims to utilize MySQL, to conduct an in-depth analysis of sales trends and patterns within the Classic Models dataset and PowerBI to gain clear visual insight on the data. 

In particular I wanted to explore the following questions:

- A general overview about sales and net profit based on product line, cost of sales, sales's office country and customers residence. 
- Deconstruction tree to show which products are generating highest revenue. 
- Analyze which products are frequently purchased together and which ones are rarely purchased together by customer country. 
- Breakdown of average sales value per order and customer credit limit as we expect to find higher sales from customers with a higher credit limit. 
- List all the customers that are exceeding their maximum credit between each orders. 

### Dashboard Link (PDF): https://github.com/ThisIsMirquez/Portfolio/blob/main/Classic%20Model%20Dashboard/ClassicModel_Overview_presentation.pdf
## Dashboard Snapshot 

![snapshot](https://github.com/ThisIsMirquez/Portfolio/blob/main/Classic%20Model%20Dashboard/Pictures/snapshot.png)

## About the Dataset 
This is a classic MySQL database containig the following schemas. 

- **Customers**: stores customer’s data.
- **Products**: stores a list of scale model vehicles.
- **ProductLines**: stores a list of product line categories.
- **Orders**: stores sales orders placed by customers.
- **OrderDetails**: stores sales order line items for each sales order.
- **Payments**: stores payments made by customers based on their accounts.
- **Employees**: stores all employee information as well as the organization structure such as who reports to whom.
- **Offices**: stores sales office data.


With the following relations 

![ralations](https://github.com/ThisIsMirquez/Portfolio/blob/main/Classic%20Model%20Dashboard/Pictures/mysql-sample-database.png)

## Questions 1 & 2
- Load data into MySQL to define the schemas needed to perform the queries. 
- Checking if and where we have null values. 
- Gathering data for the PowerBI analysis, first defining two new columns. 

         sum(QuantityOrdered) * sum(priceEach) as sales_value
         sum(buyPrice) * sum(quantityOrdered) as cost_of_sales
    they will allow us to calculate later the `net_profit` as the difference between `sales_value` and `cost_of_sales`. 


        create or replace view classicmodels.sales_data_for_power_bi  as

        SELECT orderDate,
        ord.orderNumber,
        p.productName,
        p.productLine,
        c.customerName,
        c.city as customer_city,
        c.country as customer_country,
        o.city as office_city,
        o.country as office_country,
        buyPrice, 
        priceEach,
        QuantityOrdered,
        sum(QuantityOrdered) * sum(priceEach) as sales_value,
        sum(buyPrice) * sum(quantityOrdered) as cost_of_sales


        FROM classicmodels.orders ord
        left join classicmodels.orderdetails od
        on ord.orderNumber = od.orderNumber
        left join classicmodels.customers c
        on ord.customerNumber = c.customerNumber
        left join classicmodels.products p
        on od.productCode = p.productCode

        left join classicmodels.employees e
        on c.salesRepEmployeeNumber = e.employeeNumber
        left join classicmodels.offices o
        on e.officeCode = o.officeCode
        group by
        orderdate,
        ord.ordernumber,
        p.productName,
        p.productLine,
        c.customerName,
        c.city,
        c.country,
        o.city,
        o.country,
        buyPrice, 
        priceEach,
        QuantityOrdered;

**The dashboard can be found at the top of this document.**

## Question 3
 - To visualize which product are generally bought together we want to show for    each order that contains more that one product which is the "next different product" after the first, the second, the tird and so on.  
First we join `classicmode.orders` and `classicmodel.productline` and then we create a new table with a left join of the previous joined table. 
```
with prod_sales as (
select orderNumber, t1.productCode, productLine
from orderdetails t1
inner join products t2
on t1.productCode = t2.productCode
)

select distinct t1.ordernumber, t1.productLine as prod_one, t2.productLine as prod_two
from prod_sales t1
left join prod_sales t2
on t1.ordernumber = t2.ordernumber and t1.productLine <> t2.productLine 
```
- We get the following table exporting in Excel 

![Snap_1](https://github.com/ThisIsMirquez/Portfolio/blob/main/Classic%20Model%20Dashboard/Pictures/bought_together.png)

## Question 4
- First we need to group the customers in "credit limit groups". I decided to break them down as: 
    - less than 75k
    - between 75k and 100k
    - between 100k and 150k
    - more than 150k
    and subsequently summed over the `sales_value` for each order. 
    
    So I defined a Common Table Expression (CTE) as 
```
with cte_sales as (
select t1.orderNumber, t1.customerNumber, productCode, quantityOrdered, priceEach, priceEach * quantityOrdered as sales_value,
creditLimit
from orders t1
inner join orderdetails t2
on t1.orderNumber = t2.orderNumber
inner join customers t3
on t1.customerNumber = t3.customerNumber)
```
- I then selected from the CTE customers divided in different credit groups summing up their sales 

```
select ordernumber, customernumber, 
case 
when creditlimit < 75000 then 'a: less than 75k'
when creditlimit between 75000 and 100000 then 'b: between 75k and 100k'
when creditlimit between 100000 and 150000 then 'c: between 100k and 150k'
when creditlimit > 150000  then 'd: over 150k'
else 'Other' end as creditlimit_group,
sum(sales_value) as sales_value
from cte_sales
group by ordernumber, customernumber, creditlimit_group
```

- Exporting into Excel we can then get the total of sales by credit group and counting the number of orders we can calculate the average sales value per order. 

![Snap_2](https://github.com/ThisIsMirquez/Portfolio/blob/main/Classic%20Model%20Dashboard/Pictures/avg_sales_value.png)

## Question 5

- In order to get a list of customers with "negative credit" we need to think how this model is defined. For each order the customer has a maximum credit with the company, so between each order made by the customer we can flag if they owe money calculating the total sales and subtracting the running payment made by the customer, then if the difference between the maximum credit and the owed money is negative then we can report the customer as a debtor. 

    So first and foremost lets calculate the sales value for each order and each payment as cte
```
with cte_sales as (
select
t1.orderdate,
t1.customerNumber,
t1.orderNumber,
customerName,
productCode,
creditLimit,
quantityOrdered * priceEach as sales_value 
from orders t1
inner join orderdetails t2
on t1.orderNumber = t2.orderNumber
inner join customers t3
on t1.customerNumber = t3.customerNumber), 

payments_cte as(
select *
from payments),
```
then we need to collect the date of the following order, for each order made
```
running_total_sales_cte as (
select *, lead(orderdate) over (partition by customernumber order by orderdate) as next_order_date
from 
(
select 
orderdate,
orderNumber,
customerNumber,
customerName,
creditlimit,
sum(sales_value) as sales_value
from cte_sales
group by 
orderdate,
orderNumber,
customerNumber,
customerName,
creditlimit) 
subquery
),
```
we then define the running_total_sales as incremental sum of sales by order_date and the running_total_payments as incremental sum of payments by order_date, for each customer. We can perform a simple calculation to show only the customers with a negative `difference` to show which one is a debtor.
```
main_cte as (
select t1.*,
sum(sales_value) over (partition by t1.customernumber order by orderdate) as running_total_sales,
sum(amount) over (partition by t1.customerNumber order by orderdate) as running_total_payments
from running_total_sales_cte t1
left join payments_cte t2
on t1.customerNumber = t2.customerNumber and t2.paymentdate between t1.orderdate and case when t1.next_order_date is null then current_date else next_order_date end
order by t1.customernumber, orderdate)

select *, running_total_sales - running_total_payments as money_owed,
creditlimit - (running_total_sales - running_total_payments) as difference
from main_cte
where creditlimit - (running_total_sales - running_total_payments) < 0

```
![Snap_3](https://github.com/ThisIsMirquez/Portfolio/blob/main/Classic%20Model%20Dashboard/Pictures/debtor.png)
