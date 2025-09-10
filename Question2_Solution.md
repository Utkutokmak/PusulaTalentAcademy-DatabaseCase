# Question 2 – Index Strategy & Query Optimization 

---

## 1️⃣ What performance problems can arise from this query?

- **Function usage** (`LOWER(AdSoyad)`, `YEAR(KayitTarihi)`)  
  → When a function runs on a column, the index is ignored, leading to table scans.  

- **LIKE '%ahmet%' expression**  
  → Because `%` is at the beginning, indexes cannot be used; a full scan is required.  

- **SELECT ***  
  → Fetches all unnecessary columns, increasing I/O and network load.  

- **Large data volume**  
  → Even with 1–2 million rows, these issues cause serious slowdowns.  

---

## 2️⃣ How would you optimize this query and/or the table structure?

### 🟢 Immediate Solutions (0–48 Hours)

**1. Make the date filter sargable**

```sql
-- Bad
SELECT * FROM HastaKayit 
WHERE YEAR(KayitTarihi) = 2024;

-- Good
SELECT Id, AdSoyad, KayitTarihi
FROM HastaKayit
WHERE KayitTarihi >= '2024-01-01'
  AND KayitTarihi <  '2025-01-01';
```

This way, an index on `KayitTarihi` can be used.  

---

**2. Remove unnecessary SELECT \***  

```sql
-- Bad
SELECT * FROM HastaKayit ...

-- Good (only required columns)
SELECT Id, AdSoyad, KayitTarihi
FROM HastaKayit ...
```

---

**3. Add a persisted computed column for lowercase values**

```sql
-- Add new column
ALTER TABLE dbo.HastaKayit
ADD AdSoyad_Lower AS LOWER(AdSoyad) PERSISTED;

-- Create index
CREATE NONCLUSTERED INDEX IX_HastaKayit_AdSoyadLower
ON dbo.HastaKayit (AdSoyad_Lower);
```

Now the query becomes:

```sql
SELECT Id, AdSoyad, KayitTarihi
FROM dbo.HastaKayit
WHERE AdSoyad_Lower LIKE 'ahmet%'   -- if prefix search is applied
  AND KayitTarihi >= '2024-01-01'
  AND KayitTarihi <  '2025-01-01';
```

---

### 🟡 Mid-Term Solutions (1–2 Weeks)

**1. Full-Text Index for substring searches**

If `%ahmet%` searches (contains) must be supported, the correct approach is **Full-Text Index**.

```sql
-- Full-text catalog
CREATE FULLTEXT CATALOG ftCatalog AS DEFAULT;

-- Full-text index
CREATE FULLTEXT INDEX ON dbo.HastaKayit(AdSoyad LANGUAGE 1055) -- 1055 = Turkish
KEY INDEX PK_HastaKayit;

-- Query
SELECT Id, AdSoyad, KayitTarihi
FROM dbo.HastaKayit
WHERE CONTAINS(AdSoyad, 'ahmet')
  AND KayitTarihi >= '2024-01-01'
  AND KayitTarihi <  '2025-01-01';
```

**2. Composite index to speed up queries**

```sql
-- Date + name combination
CREATE NONCLUSTERED INDEX IX_HastaKayit_Tarih_AdSoyadLower
ON dbo.HastaKayit (KayitTarihi, AdSoyad_Lower)
INCLUDE (Id);
```

This allows both date and name filters to run faster.  

**3. Statistics and index maintenance**

```sql
-- Update statistics
UPDATE STATISTICS dbo.HastaKayit WITH FULLSCAN;

-- Index maintenance
ALTER INDEX ALL ON dbo.HastaKayit REORGANIZE;
```

---

### 🔵 Long-Term Solutions (6–12 Months)

**OLTP–OLAP Separation**  
- Daily operations remain in the OLTP table.  
- Search and reporting queries are moved to a separate OLAP (Data Warehouse).  

**Read Replica**  
- Reporting and search load → replica server.  
- OLTP writes → primary server.  

**Columnstore Index (accelerates reporting)**

```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_HastaKayit_CS
ON dbo.HastaKayit (HastaId, KayitTarihi, AdSoyad);
```

Boosts performance dramatically in large reports.  

**Data Lifecycle Policy**  
“Records older than 2 years are moved to archive.”  

```sql
DELETE FROM dbo.HastaKayit
WHERE KayitTarihi < DATEADD(YEAR, -2, GETDATE());
```

---

## 3️⃣ What improvements can be made on the application side?

**Improve Search Behavior**  
- Offer users a “starts with” option (`'ahmet%'`).  
- `%ahmet%` → forces full table scan (slow).  
- `'ahmet%'` → can use index (very fast).  

**Preprocessing (Client-Side Lowercase)**  
- Convert the search term to lowercase in the application (e.g., `.toLower()` in C#, Java, Python).  
- Avoids running `LOWER(AdSoyad)` inside the database → makes indexes usable.  

**Pagination**  
- Instead of returning huge results, paginate (e.g., first 50 rows).  

```sql
SELECT Id, AdSoyad, KayitTarihi
FROM dbo.HastaKayit
WHERE AdSoyad_Lower LIKE 'ahmet%'
  AND KayitTarihi >= '2024-01-01'
  AND KayitTarihi <  '2025-01-01'
ORDER BY KayitTarihi DESC
OFFSET 0 ROWS FETCH NEXT 50 ROWS ONLY;
```

**Caching**  
- Frequently searched queries (e.g., “ahmet”) are cached in memory (Redis or in-app cache).  
- Same query answered instantly without hitting the DB.  

**Debouncing / Throttling**  
- When typing in the search box, don’t fire a query on every keystroke.  
- Instead, wait 300–500ms before sending a query.  
- Reduces unnecessary DB load.  
