# Olist project — where we are

## The project in plain words

We are working with a dataset from Olist, a Brazilian e-commerce platform. The dataset has around 100000 real orders made between 2016 and 2018, spread across 9 tables that cover the full life of an order — from purchase to customer review.

The goal of the project is to build a Data Warehouse, run OLAP analyses in Tableau, and apply Data Mining techniques to understand what drives customer satisfaction and delivery delays.

The project has three parts. Part 1 (the proposal) is done. We are now working on Part 2, which is a poster due on April 17. The poster does not require a working system - it is a structured presentation of our plan and ideas. Part 3, the final report and code, is due in May.

---

## The six questions we want to answer

These are the main questions the project is built around.

Q1 - Which regions and product categories have the highest sales volume, the longest average delivery delays, and the lowest review scores? How did these metrics change over time? (Descriptive)

Q2 - To what extent are delivery delays and freight costs associated with a higher chance of a negative review? (Diagnostic)

Q3 - Which sellers perform best and worst in terms of delivery time and customer satisfaction? What factors, like region, category, or order volume, explain those differences? (Diagnostic)

Q4 - Can we predict whether an order will result in a negative review (1 or 2 stars) based on information available before the delivery? (Predictive — supervised classification)

Q5 - Can we estimate how many days a delivery will take, and when the model detects a high risk of delay, recommend a preventive action like alerting the customer? (Prescriptive — regression with decision rule)

Q6 - What groups of orders with similar characteristics emerge from the data, and what risk profiles (delay or negative review) describe each group? (Clustering)

---

## The star schema

The Data Warehouse will be structured as a star schema. This means we take the 9 original Olist tables and reorganize them into one central table (the fact table) surrounded by several context tables (the dimension tables).

### What is the grain?

The grain is the level of detail in the fact table. We chose: one row = one item within one order. This is the most detailed level available and allows us to analyze by product, seller, customer, and time all at once.

### The fact table - Fact_OrderItems

This is the center of the star. Each row represents one product sold in one order. It stores measurable values:

- price
- freight_value
- delivery_delay_days (calculated as actual delivery date minus estimated delivery date)
- is_late (a flag: 1 if the order arrived late, 0 if not)
- is_negative_review (a flag: 1 if the review score was 1 or 2 stars, 0 otherwise)

The fact table also holds foreign keys that connect to each dimension table.

### The Dimension Tables

Dim_Time - when the order was placed. Includes day, day of week, week, month, trimester, year, season, and a weekend flag.

Dim_Customer - who placed the order. Includes zip code prefix, city, state, and region of Brazil.

Dim_Product - what was ordered. Includes category (in English and Portuguese), weight, volume, and number of photos.

Dim_Seller - who sold the product. Includes zip code prefix, city, state, and region.

Dim_Payment - how the order was paid. Includes payment type, number of installments, and total value.

Dim_Review - how the order was rated. Includes review score, whether it was negative, and whether the customer left a written comment.

---

## What comes next for the poster

The star schema is defined. The remaining pieces needed for the poster are:

1. A visual diagram of the star schema
2. Interface mockups showing how a user would interact with the data (at least two types of interaction, for example time-based and location-based)
3. A workflow diagram showing the path from the raw Olist data through ETL to the Data Warehouse and then to the OLAP analyses and Data Mining models
4. Some preliminary charts from the real data to motivate the project and illustrate expected results
