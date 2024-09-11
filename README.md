# Big Data: Spark Lab and ClickHouse Lab Solutions

This repository contains completed solutions for two labs from the **Big Data** course: **Spark Lab** and **ClickHouse Lab**. The labs were implemented using PySpark and ClickHouse, respectively.

---

## Spark Lab

### General Info

This lab focuses on data analysis using PySpark. The tasks involve analyzing social network (VK) data, including posts, likes, and reposts. The solution includes all tasks described below.

### Tasks Completed

1. **Top posts identified:**
   - by likes count
   - by comments count
   - by reposts count
2. **Top users identified:**
   - by likes received
   - by reposts made
3. **Reposts from the ITMO group extracted.**
4. **Emojis analyzed in user posts:**
   - Top positive emojis
   - Top neutral emojis
   - Top negative emojis
5. **Probable fans identified.**
6. **Probable friends identified.**

The solution is implemented in the notebook `spark-lab.ipynb`.

---

## ClickHouse Lab

### General Info

This lab is centered around building a Data Warehouse (DWH) using ClickHouse. The tasks involve working with Materialized Views (MV) and Distributed tables. The dataset used for this lab consists of 12 million user transactions.

### Tasks Completed

- Two or more Materialized Views (MVs) were implemented.
- Data was uploaded to a ClickHouse cluster using the **MergeTree** family of tables.
- Distributed tables were created using the **Distributed** engine with proper sharding.
- Additional tables were created for specific use cases.

The solution is implemented in `clickhouse-lab.md`.

---

### How to Access the Solutions

To view or run the completed solutions:

1. Clone the repository:

```bash
git clone https://github.com/your-username/bigdata-labs-solutions.git
cd src
