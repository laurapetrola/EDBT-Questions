# PostgreSQL and Commercial DB - Unapplied Query Optimization Heuristics

This document provides a set of analytical questions and their corresponding SQL queries, illustrating the impact of various **SQL rewriting heuristics** on query performance across **PostgreSQL** and a **Commercial Database (SQL Server)**. The queries presented here were considered for an accompanying article but were set aside to maintain focus.

The goal is to demonstrate the practical application and performance influence of specific optimization strategies.

## Heuristics Applied
| Heuristic Category | Description |
| :--- | :--- |
| **H1** | Eliminate unnecessary `GROUP BY`. |
| **H2 / H3** | Replace `ALL`/`ANY`/`SOME` subqueries with `MAX`/`MIN` aggregates. |
| **H4** | Move functions applied to indexed columns out of the comparison (e.g., `CAST(col) = 'val'` $\rightarrow$ `col = CAST('val' AS type)`). |
| **H5** | Eliminate unnecessary `DISTINCT`. |
| **H6** | Avoid correlated subqueries, preferring `JOIN`s, `CTE`s, or window functions. |
| **H7** | Propagate filter conditions across `JOIN` keys (e.g., `R.A = S.B AND R.A = val` $\rightarrow$ `R.A = S.B AND R.A = val AND S.B = val`). |
| **H8** | Rewrite `IN (val1, val2, ...)` as multiple `OR` conditions. |

---

## 1. Eliminate unnecessary GROUP BY (H1)

### Question: Show me customer details (ID, name, address, and phone) along with their associated nation and region information grouped.

#### PostgreSQL

| Query Style | Performance (No Index) | Performance (With Index) |
| :--- | :--- | :--- |
| **Without Heuristics (NH)** |  5.1810 seconds |  4.6517 seconds |
| **With Heuristics (CH)** |  7.8777 seconds |  6.8353 seconds |

##### Without Heuristics (NH)
```sql
SELECT 
    c.c_custkey AS customer_id,
    c.c_name AS customer_name,
    c.c_address AS customer_address,
    c.c_phone AS customer_phone,
    n.n_name AS nation_name,
    r.r_name AS region_name
FROM 
    h_customer c
JOIN 
    h_nation n ON c.c_nationkey = n.n_nationkey
JOIN 
    h_region r ON n.n_regionkey = r.r_regionkey
GROUP BY 
    c.c_custkey, c.c_name, c.c_address, c.c_phone, n.n_name, r.r_name
ORDER BY 
    c.c_custkey;
```

##### Contains Heuristics (CH)
```
SELECT 
    c.c_custkey AS customer_id,
    c.c_name AS customer_name,
    c.c_address AS customer_address,
    c.c_phone AS customer_phone,
    n.n_name AS nation_name,
    r.r_name AS region_name
FROM 
    h_customer c
JOIN 
    h_nation n ON c.c_nationkey = n.n_nationkey
JOIN 
    h_region r ON n.n_regionkey = r.r_regionkey;
```

#### Commercial Database
| Query Style | Performance (No Index) | Performance (With Index) |
| :--- | :--- | :--- |
| **Without Heuristics (NH)** |  6.4708 seconds |  6.2918 seconds |
| **With Heuristics (CH)** |  6.0340 seconds |  6.0708 seconds |

##### Without Heuristics (NH)
```
SELECT 
    c.c_custkey AS CustomerID,
    c.c_name AS CustomerName,
    c.c_address AS CustomerAddress,
    c.c_phone AS CustomerPhone,
    n.n_name AS NationName,
    r.r_name AS RegionName
FROM 
    H_Customer c
    INNER JOIN H_Nation n ON c.c_nationkey = n.n_nationkey
    INNER JOIN H_Region r ON n.n_regionkey = r.r_regionkey
GROUP BY 
    c.c_custkey,
    c.c_name,
    c.c_address,
    c.c_phone,
    n.n_name,
    r.r_name
ORDER BY 
    c.c_custkey;
```

##### Contains Heuristics (CH)
```
SELECT 
    c.c_custkey,
    c.c_name,
    c.c_address,
    c.c_phone,
    n.n_name AS nation_name,
    r.r_name AS region_name
FROM H_Customer c
JOIN H_Nation n ON c.c_nationkey = n.n_nationkey
JOIN H_Region r ON n.n_regionkey = r.r_regionkey;
```

## 2. Replace ALL with MAX (H2)
### Question: List all orders whose total price is greater than the total price of every order with priority '2-HIGH'.
#### PostgreSQL

| Query Style | Performance (No Index) | Performance (With Index) |
| :--- | :--- | :--- |
| **Without Heuristics (NH)** |  7.2391 seconds |  1.1621 seconds |
| **With Heuristics (CH)** |  3.7480 seconds |  0.8196 seconds |

##### Without Heuristics (NH)
```
SELECT o.*
FROM h_order o
WHERE o.o_totalprice > ALL (
    SELECT o2.o_totalprice
    FROM h_order o2
    WHERE o2.o_orderpriority = '2-HIGH'
);
```

##### Contains Heuristics (CH)
```
SELECT o1.*
FROM h_order o1
WHERE o1.o_totalprice > (
    SELECT MAX(o2.o_totalprice)
    FROM h_order o2
    WHERE o2.o_orderpriority = '2-HIGH'
);
```

#### Commercial Database
The Commercial Database optimizer automatically rewrites the ALL clause into a MAX operation internally.

|  Query Style | Performance (No Index) | Performance (With Index) |
|  :--- | :--- | :--- |
| **Original Query (CH)** |   0.4837 seconds |  0.1933 seconds |

##### Original Query (CH)
```
SELECT o1.*
FROM H_Order o1
WHERE o1.o_totalprice > (
    SELECT MAX(o2.o_totalprice)
    FROM H_Order o2
    WHERE o2.o_orderpriority = '2-HIGH'
);
```

## 3. Replace ANY/SOME with MIN (H3)
### Question: Find all orders where the total price is greater than at least one high-priority order.
#### PostgreSQL

| Query Style | Performance (No Index) | Performance (With Index) |
| :--- | :--- | :--- |
| **Without Heuristics (NH)** |  162.3277 seconds |  161.5852 seconds |
| **With Heuristics (CH)** |  28.0556 seconds |  26.3579 seconds |

##### Without Heuristics (NH)
```
SELECT o1.o_orderkey, o1.o_totalprice, o1.o_orderpriority
FROM h_order o1
WHERE o1.o_totalprice > ANY (
    SELECT o2.o_totalprice
    FROM h_order o2
    WHERE o2.o_orderpriority = '1-URGENT'
)
```
##### Contains Heuristics (CH)
```
SELECT o_orderkey, o_totalprice
FROM h_order
WHERE o_totalprice > (
    SELECT MIN(o_totalprice)
    FROM h_order
    WHERE o_orderpriority = '1-URGENT'
)
```

#### Commercial Database
The Commercial Database optimizer  automatically rewrites the ANY clause into a MIN operation internally.

| Query Style | Performance (No Index) | Performance (With Index) |
| :--- | :--- | :--- |
| **Original Query (CH)** |  78.9118 seconds |  81.0501 seconds |

##### Original Query (CH)
```
SELECT o.*
FROM H_Order o
WHERE o.o_totalprice > (
    SELECT MIN(o2.o_totalprice)
    FROM H_Order o2
    WHERE o2.o_orderpriority = '1-URGENT'
);
```
## 4. Move function applied to a column (H4)
### Question: Show me all the rows from the lineitem table where the l_quantity value, when converted to text, is equal to '28'.
#### PostgreSQL
| Query Style | Performance (No Index) | Performance (With Index) |
| :--- | :--- | :--- |
| **Without Heuristics (NH)** |  26.5019 seconds |  23.0724 seconds |
| **With Heuristics (CH)** |  23.7603 seconds |  21.0829 seconds |

##### Without Heuristics (NH)
```
SELECT *
FROM h_lineitem
WHERE l_quantity::text = '28';
```
##### Contains Heuristics (CH)
```
SELECT *
FROM h_lineitem
WHERE l_quantity = CAST('28' AS DOUBLE PRECISION);
```

#### Commercial Database
| Query Style | Performance (No Index) | Performance (With Index) |
| :--- | :--- | :--- |
| **Without Heuristics (NH)** |  10.2406 seconds |  9.9219 seconds |
| **With Heuristics (CH)** |  8.0105 seconds |  8.6471 seconds |

##### Without Heuristics (NH)
```
SELECT *
FROM H_Lineitem
WHERE CAST(l_quantity AS VARCHAR(20)) = '28';
```
##### Contains Heuristics (CH)
```
SELECT *
FROM H_Lineitem
WHERE l_quantity = CAST('28' AS FLOAT(53));
```

## 5. Eliminate unnecessary DISTINCT (H5)
### Question: Show me a clean list of all orders with no duplicates, including each order's ID, total price, and the customer's name who placed it - sorted by order number.
#### PostgreSQL
|  Query Style | Performance (No Index) | Performance (With Index) |
|:--- | :--- | :--- |
| **Without Heuristics (NH)** |  119.7125 seconds |  118.8553 seconds |
| **With Heuristics (CH)** |  111.4551 seconds |  112.0633 seconds |

##### Without Heuristics (NH)
```
SELECT DISTINCT 
    o.o_orderkey AS order_id,
    o.o_totalprice AS total_price,
    c.c_name AS customer_name
FROM 
    h_order o
JOIN 
    h_customer c ON o.o_custkey = c.c_custkey
ORDER BY 
    o.o_orderkey;
```
##### Contains Heuristics (CH)
```
SELECT o.o_orderkey, o.o_totalprice, c.c_name
FROM h_order o
JOIN h_customer c ON o.o_custkey = c.c_custkey
ORDER BY o.o_orderkey;
```

#### Commercial Database
| Query Style | Performance (No Index) | Performance (With Index) |
| :--- | :--- | :--- |
| **Without Heuristics (NH)** |  45.4891 seconds |  45.0368 seconds |
| **With Heuristics (CH)** |  44.0442 seconds |  45.0508 seconds |

##### Without Heuristics (NH)
```
SELECT DISTINCT 
    o.o_orderkey AS OrderID, 
    o.o_totalprice AS TotalPrice, 
    c.c_name AS CustomerName
FROM H_Order o
INNER JOIN H_Customer c ON o.o_custkey = c.c_custkey
ORDER BY o.o_orderkey;
```
##### Contains Heuristics (CH)
```
SELECT 
    o.o_orderkey,
    o.o_totalprice,
    c.c_name
FROM 
    H_Order o
INNER JOIN 
    H_Customer c ON o.o_custkey = c.c_custkey
ORDER BY 
    o.o_orderkey;
```


## 6. Avoid correlated subqueries (H6)
### Question: For each customer, what is the date of their most recent order and the total price of that order?
#### PostgreSQL
|  Query Style | Performance (No Index) | Performance (With Index) |
| :--- | :--- | :--- |
| **Without Heuristics (NH)** |  174.5616 seconds |  26.5953 seconds |
| **With Heuristics (CH)** |  16.0939 seconds |  7.4452 seconds |

##### Without Heuristics (NH)
```
SELECT 
    c.c_custkey,
    c.c_name,
    MAX(o.o_orderdate) AS most_recent_order_date,
    o_totalprice
FROM 
    h_customer c
JOIN 
    h_order o ON c.c_custkey = o.o_custkey
WHERE 
    o.o_orderdate = (
        SELECT MAX(o2.o_orderdate)
        FROM h_order o2
        WHERE o2.o_custkey = c.c_custkey
    )
GROUP BY 
    c.c_custkey, c.c_name, o.o_totalprice;
```
##### Contains Heuristics (CH)
```
WITH customer_recent_orders AS (
    SELECT 
        o_custkey,
        o_orderdate,
        o_totalprice,
        ROW_NUMBER() OVER (PARTITION BY o_custkey ORDER BY o_orderdate DESC) AS rn
    FROM h_order
)
SELECT 
    c.c_custkey,
    c.c_name,
    cro.o_orderdate AS most_recent_order_date,
    cro.o_totalprice AS order_total_price
FROM h_customer c
JOIN customer_recent_orders cro ON c.c_custkey = cro.o_custkey
WHERE cro.rn = 1
ORDER BY c.c_custkey;
```

#### Commercial Database
| Query Style | Performance (No Index) | Performance (With Index) |
|  :--- | :--- | :--- |
| **Without Heuristics (NH)** |  9.8730 seconds |  7.3285 seconds |
| **With Heuristics (CH)** |  0.7893 seconds |  0.5780 seconds	 |

##### Without Heuristics (NH)
```
SELECT 
    c.c_custkey,
    c.c_name,
    MAX(o.o_orderdate) AS most_recent_order_date,
    recent_orders.o_totalprice
FROM 
    H_Customer c
INNER JOIN 
    H_Order o ON c.c_custkey = o.o_custkey
INNER JOIN 
    (
        SELECT 
            o_custkey,
            o_orderdate,
            o_totalprice,
            ROW_NUMBER() OVER (PARTITION BY o_custkey ORDER BY o_orderdate DESC) AS rn
        FROM 
            H_Order
    ) recent_orders ON o.o_custkey = recent_orders.o_custkey 
                    AND o.o_orderdate = recent_orders.o_orderdate
WHERE 
    recent_orders.rn = 1
GROUP BY 
    c.c_custkey,
    c.c_name,
    recent_orders.o_totalprice
ORDER BY 
    c.c_custkey;
```
##### Contains Heuristics (CH)
```
WITH LatestOrders AS (
    SELECT 
        o_custkey,
        MAX(o_orderdate) AS MostRecentOrderDate
    FROM H_Order
    GROUP BY o_custkey
)
SELECT 
    c.c_custkey,
    c.c_name,
    lo.MostRecentOrderDate,
    o.o_totalprice
FROM H_Customer c
INNER JOIN LatestOrders lo ON c.c_custkey = lo.o_custkey
INNER JOIN H_Order o ON lo.o_custkey = o.o_custkey AND lo.MostRecentOrderDate = o.o_orderdate
ORDER BY c.c_custkey;
```

## 7. Propagate filter conditions across JOIN keys (H7)
### Question: Show supplier id and name for suppliers from nation key 15, and include the nation name.
#### PostgreSQL
|  Query Style | Performance (No Index) | Performance (With Index) |
|  :--- | :--- | :--- |
| **Without Heuristics (NH)** |  0.0081 seconds |  N/A |
| **With Heuristics (CH)** |   0.0087 seconds |  N/A |

##### Without Heuristics (NH)
```
SELECT s.s_suppkey, s.s_name, n.n_name
FROM h_supplier s
JOIN h_nation n ON s.s_nationkey = n.n_nationkey
WHERE s.s_nationkey = 15;
```
##### Contains Heuristics (CH)
```
SELECT s.s_suppkey, s.s_name, n.n_name
FROM h_supplier s
JOIN h_nation n ON s.s_nationkey = n.n_nationkey
WHERE s.s_nationkey = 15 AND n.n_nationkey = 15;
```

#### Commercial Database
|  Query Style | Performance (No Index) | Performance (With Index) |
|  :--- | :--- | :--- |
| **Without Heuristics (NH)** |  0.0123 seconds |  0.0108 seconds |
| **With Heuristics (CH)** |  0.0135 seconds |  0.0105 seconds	 |

##### Without Heuristics (NH)
```
SELECT 
    s.s_suppkey AS supplier_id,
    s.s_name AS supplier_name,
    n.n_name AS nation_name
FROM 
    H_Supplier s
INNER JOIN 
    H_Nation n ON s.s_nationkey = n.n_nationkey
WHERE 
    s.s_nationkey = 15;
```
##### Contains Heuristics (CH)
```
SELECT 
    s.s_suppkey,
    s.s_name,
    n.n_name
FROM 
    H_Supplier s
JOIN 
    H_Nation n ON s.s_nationkey = n.n_nationkey
WHERE 
    s.s_nationkey = 15 
    AND n.n_nationkey = 15;
```

## 8. Rewrite IN as OR conditions (H8)
### Question: Which customers are from either 'GERMANY', 'FRANCE', or 'BRAZIL'?
#### PostgreSQL
|  Query Style | Performance (No Index) | Performance (With Index) |
|  :--- | :--- | :--- |
| **Without Heuristics (NH)** |  1.0642 seconds |  0.6241 seconds |
| **With Heuristics (CH)** |   0.9135 seconds |  0.6383 seconds |

##### Without Heuristics (NH)
```
SELECT c.c_custkey, c.c_name, c.c_address, c.c_phone, c.c_acctbal, c.c_mktsegment, n.n_name AS country
FROM h_customer c
JOIN h_nation n ON c.c_nationkey = n.n_nationkey
WHERE n.n_name IN ('GERMANY', 'FRANCE', 'BRAZIL')
ORDER BY c.c_custkey;
```
##### Contains Heuristics (CH)
```
SELECT c.*
FROM h_customer c
JOIN h_nation n ON c.c_nationkey = n.n_nationkey
WHERE n.n_name = 'GERMANY              ' OR n.n_name = 'FRANCE                      ' OR n.n_name = 'BRAZIL                      ';
```

#### Commercial Database
|  Query Style | Performance (No Index) | Performance (With Index) |
|  :--- | :--- | :--- |
| **Without Heuristics (NH)** |  1.3299 seconds |  0.9238 seconds |
| **With Heuristics (CH)** |  1.1021 seconds |   0.8467 seconds	 |

##### Without Heuristics (NH)
```
SELECT 
    c.c_custkey,
    c.c_name,
    c.c_address,
    c.c_nationkey,
    c.c_phone,
    c.c_acctbal,
    c.c_mktsegment,
    c.c_comment
FROM H_Customer c
JOIN H_Nation n ON c.c_nationkey = n.n_nationkey
WHERE n.n_name IN ('GERMANY', 'FRANCE', 'BRAZIL');
```
##### Contains Heuristics (CH)
```
SELECT 
    c.c_custkey,
    c.c_name,
    c.c_address,
    c.c_nationkey,
    c.c_phone,
    c.c_acctbal,
    c.c_mktsegment,
    c.c_comment
FROM 
    H_Customer c
    INNER JOIN H_Nation n ON c.c_nationkey = n.n_nationkey
WHERE 
    n.n_name = 'GERMANY' OR 
    n.n_name = 'FRANCE' OR 
    n.n_name = 'BRAZIL';
```
These examples provide additional evidence that explicitly applying SQL rewriting heuristics can often, though not always, lead to significant performance gains, especially in PostgreSQL where the optimizer is less aggressive in certain rewrites (like H2 and H3). In Commercial Databases, internal optimizations may negate the benefit of some rewrites (H2, H3), but others (H6, H4) still show a clear performance edge.

For maximum efficiency and cross-database compatibility, it is generally recommended to use the heuristic-based approach as it often results in clearer, more direct execution paths for the database query planner.
