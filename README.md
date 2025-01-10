# White Paper: Cachefilter —A High-Performance, Precomputed Alternative to Crossfilter

## 1 Introduction

In today’s data-driven environment, organizations often want **interactive dashboards** that allow users to explore **large datasets** with deep drill-down capabilities. Traditionally, in java script libraries such as **[Crossfilter](https://crossfilter.github.io/crossfilter/)** enable slicing and dicing directly in the browser by loading the entire dataset into memory. While Crossfilter shines for datasets up to a few hundred thousand rows, performance quickly degrades as data volume grows into the **millions** or **billions** of rows.

This paper introduces **Cachefilter**, a proposed **open-source** library designed as a **drop-in replacement** for Crossfilter’s _categorical_ filtering features. The core difference lies in Cachefilter’s approach: it uses **precomputed aggregations** (similar to SQL’s `GROUP BY CUBE`) to enable _constant_ dashboard responsiveness regardless of dataset size. By offloading aggregation computation to a preprocessing step, Cachefilter avoids the in-browser overhead of calculating sums and counts on the fly.

- **Goal**: Provide **fast** drill-downs and interactive filtering for large-scale data (hundreds of millions to tens of billions of rows).
- **Approach**: Precompute all **“simple”** aggregations (i.e., full combinations of categorical variables, including “ALL” to represent aggregation across the entire domain) in a step that runs outside the browser (e.g., via Python/Pandas).
- **Key Advantage**: Maintain **constant-time** performance for the majority of typical categorical filters—without needing a large back-end infrastructure or expensive hardware resources.

Whether you are working with public sector budgets, large manufacturing processes, or high-volume retail and ecommerce analytics, Cachefilter’s precomputation model ensures near-instant drill-down to deeper levels of granularity.

---
## 2 “Simple” vs. “Complex” Aggregations

### 2.1 Simple Aggregations
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
### 2.2 Complex Aggregations
A “complex aggregation” is an **incomplete** selection across one or more columns. For instance, “filter by years 2021 **and** 2022 but **not** 2023,” or “filter by region=North **or** South but **not** West.” These subsets do **not** map cleanly to a single precomputed row in the CUBE because they exclude some category values.

Instead of precomputing every partial subset (which would explode storage requirements), **Cachefilter** derives complex aggregations by **combining** the relevant simple aggregations. For example, summing the precomputed rows for `(year=2021, region=North, product=ALL)` and `(year=2022, region=North, product=ALL)` yields the complex filter for “years 2021 and 2022, region=North,” on the fly.

This approach ensures we only store the “complete” rollups, but can still handle more granular (or partial) filters by compositing multiple simple aggregates.
### 2.3 Why cache simple aggregations and compute complex aggregations live?

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
## 3 Storing and Retrieving Precomputed Data

### 3.1 Hash per row Format

We will start by outlining a storage format with a single hash for each row in the simple aggregation table this will not be the final format we are suggesting for storage but it will be useful to explain some of the concepts that we want to use.

#### 3.1.1 Precomputation Phase
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
5. **Store the row** in a place where it can easily be retrieved based on the hash
#### 3.1.2 Naive Storage
The simplest way to store this data would be in individual json files with names base on the hash. However that would quickly become a vast number of files which in itself would lead to problems in some operating systems.
#### 3.1.3 Json clusters
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

#### 3.1.4 Database based storage
The best way of storing the data for very large dataset would be either a key-value in memory database like valkey (formerly redis) which scales up to the size of the RAM or a traditional database like postgres or snowflake which scale up to the size of disk or functionally infinitely for constant time key value retrival. However this is would make it more difficult to deploy the solution in an enterprise environment than a simple file based solution.
Note that this will also be much faster that directly querying a Database like postgres for an aggregation since fetching a value in a table based on a key is a constant time operation in databases while aggregation over a table is table size dependent. 

### 3.2 The Problems with the hash per row Format and a proposed solution

In visualizations it is very often the case that one has one display per categorical column.
In a way similar to this visualization of the change of the global burden of disease between 2000 and 2019 along the country, sex and age categorical columns:
![Pasted image 20250110114901](https://github.com/user-attachments/assets/9e8e153a-d46f-4017-9303-ab3f94604a3e)

Note that getting all of the entries of a categorical variable like country that has a of different values will take a lot of fetch requests in the one has per row format.
Also note that a filter on a value in column A in crossfilter never changes the values that are displayed along column A (a behavior that is the same in other popular dashboarding software like PowerBI).
For example here filtering on the United States of America lead to the numbers displayed for Sex and age to be filtered down to the values for the US but in the country table all other country values are still visible (they are greyed out a bit but that is a visual change that happens in the way it is displayed not in the crossfilter library). 
![Pasted image 20250110125755](https://github.com/user-attachments/assets/1d5b781c-1769-4d13-b63d-68a7efd7c11e)

Because of this we would like to store our data in the following way:
For each categorical column have one hash for each filtering outside that column and under that hash all of the entries for all of the values of that column such that we get hashes and that look like this:

```json
{
    "<sha256({'Product':{'Year':'2021','Region':'North'}})>": {
        "Product A": 1000,
        "Product B": 1500,
        "Product C": 2000
    },
    "<sha256({'Product':{'Year':'2021'}})>": {
        "Product A": 7500,
        "Product B": 9000,
        "Product C": 10500
    },
    "<sha256({'Product':{'Region':'North'}})>": {
        "Product A": 3300,
        "Product B": 4800,
        "Product C": 6300
    },
    "<sha256({'Product':{}})>": {
        "Product A": 23400,
        "Product B": 27900,
        "Product C": 32400
    }
}
```

This will generate more data since we essentially generate an dataset that is in size in $O(k^n)$ for every categorical column so we will then be in the $O(n*k^n)$ however we believe that the reading speedup due to a reduction in reading operations will be worth extra storage space, since usually the number of categorical columns is between 5 and 30 in most applications.
## 5 Nested Categories

In practice most source tables will be much smaller than the $O(k^n)$.

This is because in practice it happens very frequently that column B is just a finer grained version of column A (A: date and B: year, A: city and B: country, A: Region of the world and B: country, ect. ) lets call this kinds of columns a nested set.

Here an illustrative example of a nested set involving the World Region, Sub Region and country:

![Pasted image 20250110115322](https://github.com/user-attachments/assets/8096879b-b315-46ac-b911-3aead73f5cdb)

A nested set is conceptually ordered from the most coarse column (e.g. Region) to the most granular column (e.g. country) potentially with columns with a medium level of granularity (e.g. Subregion).

In these cases it is important not to blindly save the sql `group by cube()` result for this data since that would lead to a lot of duplicate rows:

For example this table:

| Region   | Sub Region                      | Population    |
| -------- | ------------------------------- | ------------- |
| Asia     | Southern Asia                   | 1,432,568,832 |
| Asia     | Eastern Asia                    | 1,524,904,419 |
| Africa   | Sub-Saharan Africa              | 649,185,308   |
| Asia     | South-eastern Asia              | 529,934,839   |
| Americas | Latin America and the Caribbean | 518,515,484   |
| Americas | Northern America                | 311,104,242   |
| Europe   | Eastern Europe                  | 307,676,547   |
| Asia     | Western Asia                    | 196,217,782   |
| Africa   | Northern Africa                 | 170,708,511   |
| Europe   | Western Europe                  | 184,200,174   |
| Europe   | Southern Europe                 | 145,662,355   |
| Europe   | Northern Europe                 | 94,913,606    |
| Asia     | Central Asia                    | 55,811,615    |
| Oceania  | Australia and New Zealand       | 22,671,343    |
| Oceania  | Melanesia                       | 7,012,381     |
| Oceania  | Micronesia                      | 511,666       |
| Oceania  | Polynesia                       | 685,968       |

If cubed would lead to this table 

| Region   | Sub Region                      | Total Population | usefull/not usefull row |
| -------- | ------------------------------- | ---------------- | ----------------------- |
| ALL      | ALL                             | 6,152,285,072    | usefull                 |
| Europe   | Western Europe                  | 184,200,174      | usefull                 |
| Asia     | Western Asia                    | 196,217,782      | usefull                 |
| Oceania  | Micronesia                      | 511,666          | usefull                 |
| Americas | Northern America                | 311,104,242      | usefull                 |
| Asia     | South-eastern Asia              | 529,934,839      | usefull                 |
| Europe   | Eastern Europe                  | 307,676,547      | usefull                 |
| Oceania  | Australia and New Zealand       | 22,671,343       | usefull                 |
| Oceania  | Polynesia                       | 685,968          | usefull                 |
| Africa   | Northern Africa                 | 170,708,511      | usefull                 |
| Europe   | Northern Europe                 | 94,913,606       | usefull                 |
| Asia     | Southern Asia                   | 1,432,568,832    | usefull                 |
| Americas | Latin America and the Caribbean | 518,515,484      | usefull                 |
| Europe   | Southern Europe                 | 145,662,355      | usefull                 |
| Africa   | Sub-Saharan Africa              | 649,185,308      | usefull                 |
| Asia     | Central Asia                    | 55,811,615       | usefull                 |
| Oceania  | Melanesia                       | 7,012,381        | usefull                 |
| Asia     | Eastern Asia                    | 1,524,904,419    | usefull                 |
| Oceania  | ALL                             | 30,881,358       | usefull                 |
| Americas | ALL                             | 829,619,726      | usefull                 |
| Africa   | ALL                             | 819,893,819      | usefull                 |
| Asia     | ALL                             | 3,739,437,487    | usefull                 |
| Europe   | ALL                             | 732,452,682      | usefull                 |
| ALL      | Australia and New Zealand       | 22,671,343       | not usefull             |
| ALL      | Western Europe                  | 184,200,174      | not usefull             |
| ALL      | South-eastern Asia              | 529,934,839      | not usefull             |
| ALL      | Melanesia                       | 7,012,381        | not usefull             |
| ALL      | Eastern Europe                  | 307,676,547      | not usefull             |
| ALL      | Polynesia                       | 685,968          | not usefull             |
| ALL      | Eastern Asia                    | 1,524,904,419    | not usefull             |
| ALL      | Sub-Saharan Africa              | 649,185,308      | not usefull             |
| ALL      | Northern Europe                 | 94,913,606       | not usefull             |
| ALL      | Central Asia                    | 55,811,615       | not usefull             |
| ALL      | Latin America and the Caribbean | 518,515,484      | not usefull             |
| ALL      | Micronesia                      | 511,666          | not usefull             |
| ALL      | Northern Africa                 | 170,708,511      | not usefull             |
| ALL      | Southern Asia                   | 1,432,568,832    | not usefull             |
| ALL      | Southern Europe                 | 145,662,355      | not usefull             |
| ALL      | Western Asia                    | 196,217,782      | not usefull             |
| ALL      | Northern America                | 311,104,242      | not usefull             |

And a lot of aggregation rows will aggregate over just 1 row essentially duplicating the data
If the way in which column content is nested is known we can exclusively the most granular column that the user is currently filtering on for our hash and only generate cached aggregations that fit that pattern.

## 6 Conclusion

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
