# White Paper: Cachefilter —A High-Performance, Precomputed Alternative to Crossfilter

## Introduction

In today’s data-driven environment, organizations often want **interactive dashboards** that allow users to explore **large datasets** with deep drill-down capabilities. Traditionally, in java script libraries such as **[Crossfilter](https://crossfilter.github.io/crossfilter/)** enable slicing and dicing directly in the browser by loading the entire dataset into memory. While Crossfilter shines for datasets up to a few hundred thousand rows, performance quickly degrades as data volume grows into the **millions** or **billions** of rows.

This paper introduces **Cachefilter**, a proposed **open-source** library designed as a **drop-in replacement** for Crossfilter’s _categorical_ filtering features. The core difference lies in Cachefilter’s approach: it uses **precomputed aggregations** (similar to SQL’s `GROUP BY CUBE`) to enable _constant_ dashboard responsiveness regardless of dataset size. By offloading aggregation computation to a preprocessing step, Cachefilter avoids the in-browser overhead of calculating sums and counts on the fly.

- **Goal**: Provide **fast** drill-downs and interactive filtering for large-scale data (hundreds of millions to tens of billions of rows).
- **Approach**: Precompute all **“simple”** aggregations (i.e., full combinations of categorical variables, including “ALL” to represent aggregation across the entire domain) in a step that runs outside the browser (e.g., via Python/Pandas).
- **Key Advantage**: Maintain **constant-time** performance for the majority of typical categorical filters—without needing a large back-end infrastructure or expensive hardware resources.

Whether you are working with public sector budgets, large manufacturing processes, or high-volume retail and ecommerce analytics, Cachefilter’s precomputation model ensures near-instant drill-down to deeper levels of granularity.

---
# 2. Core Concepts & Definitions

## 2.1 “Simple” vs. “Complex” Aggregations

### Simple Aggregations
A “simple aggregation” refers to **full combinations** of all categorical columns (including the “ALL” or wildcard case) in each column. (For readability we will use "ALL" as identifier for this in this paper, in practice the placeholder value for all categories should be an input in the generating function so that the user can set it and avoid naming collisions with columns that natively contain an "ALL" categories before aggregation)
So for example a table with 3 columns Year, Region, Product each of which has 3 possible values and a Revenue column that will be summed:

| Year | Region | Product   | Revenue |
| ---- | ------ | --------- | ------- |
| 2021 | North  | Product A | 1000    |
| 2021 | North  | Product B | 1500    |
| 2021 | North  | Product C | 2000    |
| 2021 | South  | Product A | 2500    |
| 2021 | South  | Product B | 3000    |
| 2021 | South  | Product C | 3500    |
| 2021 | West   | Product A | 4000    |
| 2021 | West   | Product B | 4500    |
| 2021 | West   | Product C | 5000    |
| 2022 | North  | Product A | 1100    |
| 2022 | North  | Product B | 1600    |
| 2022 | North  | Product C | 2100    |
| 2022 | South  | Product A | 2600    |
| 2022 | South  | Product B | 3100    |
| 2022 | South  | Product C | 3600    |
| 2022 | West   | Product A | 4100    |
| 2022 | West   | Product B | 4600    |
| 2022 | West   | Product C | 5100    |
| 2023 | North  | Product A | 1200    |
| 2023 | North  | Product B | 1700    |
| 2023 | North  | Product C | 2200    |
| 2023 | South  | Product A | 2700    |
| 2023 | South  | Product B | 3200    |
| 2023 | South  | Product C | 3700    |
| 2023 | West   | Product A | 4200    |
| 2023 | West   | Product B | 4700    |
| 2023 | West   | Product C | 5200    |
Will yield a simple aggregation table like this:
  
| Year | Region | Product   | Total Revenue |
| ---- | ------ | --------- | ------------- |
| 2021 | North  | Product A | 1000          |
| 2021 | North  | Product B | 1500          |
| 2021 | North  | Product C | 2000          |
| 2021 | North  | ALL       | 4500          |
| 2021 | South  | Product A | 2500          |
| 2021 | South  | Product B | 3000          |
| 2021 | South  | Product C | 3500          |
| 2021 | South  | ALL       | 9000          |
| 2021 | West   | Product A | 4000          |
| 2021 | West   | Product B | 4500          |
| 2021 | West   | Product C | 5000          |
| 2021 | West   | ALL       | 13500         |
| 2021 | ALL    | Product A | 7500          |
| 2021 | ALL    | Product B | 9000          |
| 2021 | ALL    | Product C | 10500         |
| 2021 | ALL    | ALL       | 27000         |
| 2022 | North  | Product A | 1100          |
| 2022 | North  | Product B | 1600          |
| 2022 | North  | Product C | 2100          |
| 2022 | North  | ALL       | 4800          |
| 2022 | South  | Product A | 2600          |
| 2022 | South  | Product B | 3100          |
| 2022 | South  | Product C | 3600          |
| 2022 | South  | ALL       | 9300          |
| 2022 | West   | Product A | 4100          |
| 2022 | West   | Product B | 4600          |
| 2022 | West   | Product C | 5100          |
| 2022 | West   | ALL       | 13800         |
| 2022 | ALL    | Product A | 7800          |
| 2022 | ALL    | Product B | 9300          |
| 2022 | ALL    | Product C | 10800         |
| 2022 | ALL    | ALL       | 27900         |
| 2023 | North  | Product A | 1200          |
| 2023 | North  | Product B | 1700          |
| 2023 | North  | Product C | 2200          |
| 2023 | North  | ALL       | 5100          |
| 2023 | South  | Product A | 2700          |
| 2023 | South  | Product B | 3200          |
| 2023 | South  | Product C | 3700          |
| 2023 | South  | ALL       | 9600          |
| 2023 | West   | Product A | 4200          |
| 2023 | West   | Product B | 4700          |
| 2023 | West   | Product C | 5200          |
| 2023 | West   | ALL       | 14100         |
| 2023 | ALL    | Product A | 8100          |
| 2023 | ALL    | Product B | 9600          |
| 2023 | ALL    | Product C | 11100         |
| 2023 | ALL    | ALL       | 28800         |
| ALL  | North  | Product A | 3300          |
| ALL  | North  | Product B | 4800          |
| ALL  | North  | Product C | 6300          |
| ALL  | North  | ALL       | 14400         |
| ALL  | South  | Product A | 7800          |
| ALL  | South  | Product B | 9300          |
| ALL  | South  | Product C | 10800         |
| ALL  | South  | ALL       | 27900         |
| ALL  | West   | Product A | 12300         |
| ALL  | West   | Product B | 13800         |
| ALL  | West   | Product C | 15300         |
| ALL  | West   | ALL       | 41400         |
| ALL  | ALL    | Product A | 23400         |
| ALL  | ALL    | Product B | 27900         |
| ALL  | ALL    | Product C | 32400         |
| ALL  | ALL    | ALL       | 83700         |

Where the empty fields aggregate over all possible categorical values in that column.
This is akin to performing a SQL `GROUP BY CUBE`. 
### Complex Aggregations
A “complex aggregation” is an **incomplete** selection across one or more columns. For instance, “filter by years 2021 **and** 2022 but **not** 2023,” or “filter by region=North **or** South but **not** West.” These subsets do **not** map cleanly to a single precomputed row in the CUBE because they exclude some category values.

Instead of precomputing every partial subset (which would explode storage requirements), **Cachefilter** derives complex aggregations by **combining** the relevant simple aggregations. For example, summing the precomputed rows for `(year=2021, region=North, product=ALL)` and `(year=2022, region=North, product=ALL)` yields the complex filter for “years 2021 and 2022, region=North,” on the fly.

This approach ensures we only store the “complete” rollups, but can still handle more granular (or partial) filters by compositing multiple simple aggregates.
### Why cache simple aggregations and compute complex aggregations live?

Only precomputing simple aggregations limits the total amount of data needed for storage. 
Say I have $n$ categorical columns and each column has a $k$ expressions/values (eg. North, South, West). Then the total number of rows of the source table would be in the $O(k^n)$. If I add an ALL category to each of those columns and cache all of the resulting possible combinations of aggregations then I effectively will have to cache $O((k+1)^n)$ aggregations. 

The binomial expansion of $(k+1)^n$ is:

$(k+1)^n = \sum_{i=0}^{n} \binom{n}{i} k^{n-i} \cdot 1^i$

Simplifying this, we get:

$(k+1)^n = k^n + \binom{n}{1} k^{n-1} + \binom{n}{2} k^{n-2} + \dots + 1$

Now, keeping the dominant term and simplifying for Big-O notation:

$O((k+1)^n) = O(k^n)$

Thus the simple aggregations that we want to cache are in the same big $O()$ size as the dateset itself.
In short, the key point is: we are _not_ exploding the storage significantly beyond what the dataset inherently has.

---
## 2.2 Storing and Retrieving Precomputed Data

### Precomputation Phase
The precomputation step typically runs outside the browser, such as in **Python** using Pandas:

1. **Load the data** from a database or CSV into a Pandas DataFrame.
2. **Generate the CUBE** by grouping over all categorical columns (plus an “ALL” placeholder where relevant).
3. **Compute summary metrics** (e.g., sums, counts) for each combination.
4. **Create a hash** based on the categorical columns filter column (ignoring the "ALL" columns)
   For example the rows:
```json
{
  "year": "2021",
  "region": "North",
  "product": "Product A",
  "sum_revenue": 1000,
}
{
  "year": "2021",
  "region": "ALL",
  "product": "ALL",
  "sum_revenue": 27000,
}
```
would receive the hash: 
```js
	sha256({"year":"2021","region":"North","product":"Product A"}) = "07b4f74a323888c655b94715afe2d21086ef308a94921514489cc15f82172cfc"
	sha256({"year":"2021"}) = "f3ba6849c137766af8d039a39cc2b30a25359809f42b211c0c96a893b1e8d1a6"
```
1. **Store the row** in a place where it can easily be retrieved based on the hash
### Naive Storage
The simplest way to store this data would be in individual json files with names base on the hash. However that would quickly become a vast number of files which in itself would lead to problems in some operating systems.
### Json clusters
The simplest actually useful way of storing the data would be in jsons that each represent a cluster of roughly 1,000-100,000 rows enough to be read basically instantaneously and have 1,000-100,000 of these files jsons files.
Each Json file would then represent a range of hashes. And have a structure like 
   ```json
{
   "07b4f74a323888c655b94715afe2d21086ef308a94921514489cc15f82172cfc":{
      "year":"2021",
      "region":"North",
      "product":"Product A",
      "sum_revenue":1000
   },
   "f3ba6849c137766af8d039a39cc2b30a25359809f42b211c0c96a893b1e8d1a6":{
      "year":"2021",
      "region":"ALL",
      "product":"ALL",
      "sum_revenue":27000
   }
}
```

### Database based storage
The best way of storing the data for very large dataset would be either a key-value in memory database like valkey (formerly redis) which scales up to the size of the RAM or a traditional database like postgres or snowflake which scale up to the size of disk or functionally infinitely for constant time key value retrival. However this is would make it more difficult to deploy the solution in an enterprise environment than a simple file based solution.
Note that this will also be much faster that directly querying a Database like postgres for an aggregation since fetching a value in a table based on a key is a constant time operation in databases while aggregation over a table is table size dependent.  
### In-Browser Retrieval

On the client side, **Cachefilter** uses a small JavaScript library to:

1. Determine which combination(s) of categories a user has selected.
2. **Compute the file hash prefix** (or otherwise locate the correct JSON) that contains the relevant precomputed rows.
3. Fetch that JSON and perform a **quick lookup** for the row(s) corresponding to the user’s filters.
4. For **complex filters**, sum or merge multiple precomputed rows as needed.

Because each file is relatively small and the number of rows per file is limited, these lookups can be performed **very quickly**, keeping interactive dashboards **responsive** even with billions of underlying records.

By clearly distinguishing between “simple” (fully computed) and “complex” (dynamically derived) aggregations, and by carefully **organizing** these precomputed results, Cachefilter provides constant-time or near-constant-time lookups for most real-world queries—without burdening the browser with raw data at massive scale.

## 2.3 Nested Categories

In practice most source tables will be much smaller than the $O(k^n)$.
This is because in practice it happens very frequently that column B is just a finer grained version of column A (A: date and B: year, A: city and B: country, ect. ) lets call this kinds of columns a nested set. 
A nested set is conceptually ordered from the most coarse column (e.g. country) to the most granular column (e.g. city) potentially with columns with a medium level of granularity (e.g. region in country).
In these cases it is important not to make aggregations combinations that can never happen like Country="Germany" and City="Paris", since those will vastly inflate the original table by a factor of the number of categories in the more coarse column.
If we for example generate a country level aggregation for all countries for each city in our dataset , eg:
Country="Germany" and City="Paris" 
Country="France" and City="Paris"
Country="Poland" and City="Paris"
Country="Ukraine" and City="Paris"
Country="Germany" and City="Berlin"
Country="France" and City="Berlin"
Country="Poland" and City="Berlin"
Country="Ukraine" and City="Berlin"
then the resulting table will have a number of rows that is equal to the number of original rows times the number of countries in the dataset.
And most rows will aggregate over nothing.
If the way in which column content is nested is known we can exclusively the most granular column that the user is currently filtering on for our hash and only generate cached aggregations that fit that pattern.

## Conclusion

Cachefilter offers a practical solution for organizations needing to perform fast, in-browser analysis on massive datasets without sacrificing interactivity. By precomputing “simple” aggregations (via a CUBE-like process) and then combining them on demand for more granular “complex” filters, we avoid the memory and performance bottlenecks commonly associated with large in-browser datasets.

Key takeaways include:

1. **Precomputation for Constant-Time Lookups**  
   - Simple Aggregations are generated in an offline step, reducing the browser’s workload to lightweight fetches and lookups rather than expensive on-the-fly processing.

2. **Flexible Storage Options**  
   - For moderate-scale datasets, JSON “clusters” can be served as static files—requiring no additional backend.  
   - For truly massive or real-time use cases, a key-value store like Valkey (Redis) or a disk-based database can store and deliver precomputed results instantly, preserving constant-time performance.

3. **Simple vs. Complex Aggregations**  
   - “Simple” rollups (e.g., `(year=2021, region=North, product=ALL)`) are directly cached.  
   - “Complex” rollups (e.g., `(year=2021 OR 2022, region=North)`) are derived by summing the relevant precomputed rows in the browser.

4. **Nested Categories and Practical Considerations**  
   - In real-world scenarios, columns are often hierarchical (e.g., country → region → city).  
   - Avoiding impossible category combinations reduces storage needs and query complexity.

By leveraging these principles, Cachefilter remains both efficient and straightforward to deploy—particularly valuable in enterprise environments where setting up new backend services can be cumbersome. With its “drop-in replacement” design for Crossfilter’s categorical filtering, Cachefilter promises near-instant drill-down times, even as data volumes scale into the billions of rows.
