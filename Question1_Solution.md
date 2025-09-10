# Question 1: Performance & Scalability Analysis in Hospital Data

---

## 1️⃣ Reasons for Performance Degradation 

- **Huge data volume**: After 5 years, ~45–50M rows accumulated. Any query scanning this much data inevitably becomes slow.  
- **Missing indexes**: Most queries filter by `HastaId` or `IslemTarihi`, but there are no indexes on these columns. The `Id` primary key is not useful in most queries.  
- **Single table for hot + cold data**: Recent 3-month “hot” data and 5-year-old “cold” data reside in the same table. Even when a user only requests last week’s records, SQL Server scans millions of rows.  
- **Query design issues (commonly seen in real-world usage)**:
  - `SELECT *` → retrieves unnecessary columns. Especially the wide column `Aciklama NVARCHAR(500)` increases I/O.  
  - `YEAR(IslemTarihi) = 2023` or functions like `LOWER()`, `CONVERT()` → prevent index usage and cause full table scans.  
- **No statistics maintenance**: Constant inserts fragment indexes and stale statistics lead to poor query plans.  

---

## 2️⃣ My Proposed Improvements

### 🟢 A) Immediate Actions (0–48 hours)

**Add targeted nonclustered indexes**

```sql
-- Patient-based queries
CREATE NONCLUSTERED INDEX IX_HastaIslemLog_HastaId_Tarih
ON dbo.HastaIslemLog (HastaId, IslemTarihi DESC)
INCLUDE (IslemKodu);

-- Date-based reporting
CREATE NONCLUSTERED INDEX IX_HastaIslemLog_Tarih
ON dbo.HastaIslemLog (IslemTarihi)
INCLUDE (HastaId, IslemKodu);
```

📌 Note: Do not include wide columns such as `Aciklama`, otherwise the index size will grow excessively.

**Make queries sargable**

```sql
-- Bad (non-sargable, disables indexes)
-- SELECT * FROM dbo.HastaIslemLog WHERE YEAR(IslemTarihi) = 2023;

-- Good (uses indexes, selects only needed columns)
SELECT Id, HastaId, IslemTarihi, IslemKodu
FROM dbo.HastaIslemLog
WHERE IslemTarihi >= '2023-01-01'
  AND IslemTarihi <  '2024-01-01';
```

**Update statistics and reorganize indexes**

```sql
UPDATE STATISTICS dbo.HastaIslemLog WITH FULLSCAN;
ALTER INDEX ALL ON dbo.HastaIslemLog REORGANIZE;
```

---

### 🟡 B) Mid-Term Solutions (1–2 weeks)

**Archiving (separating cold data)**

```sql
CREATE TABLE dbo.HastaIslemLog_Archive (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    HastaId INT,
    IslemTarihi DATETIME,
    IslemKodu NVARCHAR(20),
    Aciklama NVARCHAR(500)
);

BEGIN TRANSACTION;

INSERT INTO dbo.HastaIslemLog_Archive (HastaId, IslemTarihi, IslemKodu, Aciklama)
SELECT HastaId, IslemTarihi, IslemKodu, Aciklama
FROM dbo.HastaIslemLog
WHERE IslemTarihi < '2024-01-01';

DELETE FROM dbo.HastaIslemLog
WHERE IslemTarihi < '2024-01-01';

COMMIT TRANSACTION;
```

**Partitioning (time-based)**

```sql
CREATE PARTITION FUNCTION PF_Tarih (DATETIME)
AS RANGE LEFT FOR VALUES ('2023-12-31');

CREATE PARTITION SCHEME PS_Tarih
AS PARTITION PF_Tarih ALL TO ([PRIMARY]);

CREATE TABLE dbo.HastaIslemLog_Partitioned (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    HastaId INT,
    IslemTarihi DATETIME,
    IslemKodu NVARCHAR(20),
    Aciklama NVARCHAR(500)
) ON PS_Tarih(IslemTarihi);
```

📌 Note: Partitioning requires careful planning. During cutover, downtime or ETL adjustments must be considered.

---

### 🔵 C) Long-Term Solutions (6–12 months)

**OLTP–OLAP separation (critical step)**

- OLTP: operational DB for daily transactions (fast queries, small data).  
- OLAP: Data Warehouse for historical reporting and analysis.  

```sql
-- Simple daily ETL logic
INSERT INTO DW.dbo.HastaIslemLog_Fact (HastaId, IslemTarihi, IslemKodu, Aciklama)
SELECT HastaId, IslemTarihi, IslemKodu, Aciklama
FROM dbo.HastaIslemLog
WHERE IslemTarihi >= DATEADD(DAY, -1, CAST(GETDATE() AS DATE))
  AND IslemTarihi <  CAST(GETDATE() AS DATE);
```

**Read replica / secondary server**  
- Reporting queries → replica  
- OLTP writes → primary database  

**Columnstore index (accelerate reporting)**

```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_HastaIslemLog_CS
ON dbo.HastaIslemLog (HastaId, IslemTarihi, IslemKodu, Aciklama);
```

**Data compression (disk + I/O savings)**

```sql
ALTER INDEX ALL ON dbo.HastaIslemLog REBUILD WITH (DATA_COMPRESSION = PAGE);
ALTER INDEX ALL ON dbo.HastaIslemLog_Archive REBUILD WITH (DATA_COMPRESSION = PAGE);
```

**Data lifecycle policy**

Example: “2 years kept online, moved to archive afterwards, deleted after 5 years.”

```sql
DELETE FROM dbo.HastaIslemLog
WHERE IslemTarihi < DATEADD(YEAR, -2, GETDATE());
```

**Monitoring & alerting**  
Use Query Store or custom log tables to track slow queries and index effectiveness.  

---

## 3️⃣ Was using this table like this for 5 years correct?

- **Understandable**: At the beginning, the data volume was small, and going live quickly might have been the priority.  
- **But not correct**: Data growth was not anticipated, and no indexing/partitioning/archiving strategy was planned. This directly caused the current performance crisis.  
- **Best practice**: From the start, a data lifecycle policy should have been defined (e.g., auto-archive >2 years).  
  - `HastaId` and `IslemTarihi` should have been indexed.  
  - `SELECT *` should have been avoided.  
  - Partitioning and archiving should have been applied so that the table always remains manageable.  
