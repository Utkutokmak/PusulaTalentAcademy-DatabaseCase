# Question 3 – T-SQL Query Challenge (Hospital Sales Example)

---

## 1️⃣ Task 1: Total Sales Amount and Quantity per Year and Product

**Explanation:** For each year and each product, calculate the total quantity sold and the total sales amount (`Fiyat * Adet`).  

```sql
SELECT 
    YEAR(S.SatisTarihi) AS Year,
    U.UrunAdi,
    SUM(S.Adet * U.Fiyat) AS TotalSalesAmount,
    SUM(S.Adet) AS TotalQuantity
FROM dbo.Satis S
INNER JOIN dbo.Urun U ON S.UrunID = U.UrunID
GROUP BY YEAR(S.SatisTarihi), U.UrunAdi
ORDER BY Year, U.UrunAdi;
```

📌 Note:

-YEAR(SatisTarihi) extracts the year.

-SUM(Adet * Fiyat) calculates the total sales amount.

-SUM(Adet) calculates the total quantity sold.

-GROUP BY groups the results by year and product.


2️⃣ Task 2: Product with the Highest Sales Amount per Year

Explanation: For each year, find the product with the highest total sales amount.


```sql
;WITH YearlySales AS (
    SELECT 
        YEAR(S.SatisTarihi) AS Year,
        U.UrunAdi,
        SUM(S.Adet * U.Fiyat) AS TotalSalesAmount
    FROM dbo.Satis S
    INNER JOIN dbo.Urun U ON S.UrunID = U.UrunID
    GROUP BY YEAR(S.SatisTarihi), U.UrunAdi
)
SELECT Year, UrunAdi, TotalSalesAmount
FROM (
    SELECT 
        Year,
        UrunAdi,
        TotalSalesAmount,
        ROW_NUMBER() OVER (PARTITION BY Year ORDER BY TotalSalesAmount DESC) AS rn
    FROM YearlySales
) t
WHERE rn = 1;
```

📌 Note:

-The first CTE (YearlySales) calculates yearly totals for each product.

-ROW_NUMBER() OVER (PARTITION BY Year ORDER BY TotalSalesAmount DESC) ranks products by sales amount within each year.

-WHERE rn = 1 returns the top product (highest sales amount) for each year.



3️⃣ Task 3: List of Products Never Sold

Explanation: Find products that have no sales records.


```sql
SELECT U.UrunID, U.UrunAdi, U.Fiyat
FROM dbo.Urun U
LEFT JOIN dbo.Satis S ON U.UrunID = S.UrunID
WHERE S.UrunID IS NULL;
```

📌 Note:

-LEFT JOIN ensures all products are included.

-Products with no matching records in the Satis table (NULL) are identified as never sold.
















