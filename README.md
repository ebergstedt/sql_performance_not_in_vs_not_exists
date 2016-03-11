# SQL Server performance - NOT IN vs NOT EXISTS, with NULLABLE fields vs not NULLABLE

This document outlines performance on using NOT IN versus NOT EXISTS clauses, where the compared field is NULLABLE or not NULLABLE.

# Specs

**Database**: SQL Server 2012 R2

# Results

Winner is marked in **bold**.

### Run 1

### Run 2

### Run 3

# Conclusion

# Script

```sql
-- ##
-- ## NOT IN vs NOT EXISTS
-- ##

-- ##
-- ## BASELINE
-- ##
IF EXISTS (SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = 'LargerTable' AND TABLE_SCHEMA = 'TestPerformance')
BEGIN
 DROP TABLE TestPerformance.LargerTable
END
GO

IF EXISTS (SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME = 'SmallerTable' AND TABLE_SCHEMA = 'TestPerformance')
BEGIN
 DROP TABLE TestPerformance.SmallerTable
END
GO

IF EXISTS (SELECT * FROM sys.schemas WHERE name = 'TestPerformance')
BEGIN
 DROP SCHEMA TestPerformance 
END
GO

CREATE SCHEMA TestPerformance
GO

CREATE TABLE TestPerformance.LargerTable (
	Id INT IdENTITY PRIMARY KEY,
	CompareColumn CHAR(4) NOT NULL,
)
 
CREATE TABLE TestPerformance.SmallerTable (
	Id INT IdENTITY PRIMARY KEY,
	LookupColumn CHAR(4) NOT NULL,
)

INSERT INTO 
 TestPerformance.LargerTable (CompareColumn)
	SELECT TOP 
  250000
	CHAR(65+FLOOR(RAND(a.column_Id *5645 + b.object_Id)*10)) + CHAR(65+FLOOR(RAND(b.column_Id *3784 + b.object_Id)*12)) +
	CHAR(65+FLOOR(RAND(b.column_Id *6841 + a.object_Id)*12)) + CHAR(65+FLOOR(RAND(a.column_Id *7544 + b.object_Id)*8))
	FROM 
  master.sys.columns AS a 
  CROSS JOIN 
  master.sys.columns AS b
 

INSERT INTO TestPerformance.SmallerTable (LookupColumn)
	SELECT DISTINCT CompareColumn
	FROM TestPerformance.LargerTable TABLESAMPLE (25 PERCENT)


-- ##
-- ## WITHOUT NULLABILITY
-- ##

SET STATISTICS TIME ON
PRINT 'NOT IN - WITHOUT NULLABILITY'
SELECT 
 Id
 , CompareColumn 
FROM 
 TestPerformance.LargerTable AS bt
WHERE 
 bt.CompareColumn NOT IN (
	 SELECT 
   LookupColumn 
	 FROM 
   TestPerformance.SmallerTable
)
SET STATISTICS TIME OFF
 
SET STATISTICS TIME ON
PRINT 'NOT EXISTS - WITHOUT NULLABILITY'
SELECT 
 Id
 , CompareColumn
FROM TestPerformance.LargerTable AS bt
WHERE NOT EXISTS (
	SELECT 0
	FROM TestPerformance.SmallerTable  AS st
	WHERE st.LookupColumn = bt.CompareColumn
)
SET STATISTICS TIME OFF


-- ##
-- ## ALTER COLUMNS TO NULLABLE
-- ##
ALTER TABLE TestPerformance.LargerTable
ALTER COLUMN CompareColumn CHAR(4) NULL
 
ALTER TABLE TestPerformance.SmallerTable
ALTER COLUMN LookupColumn CHAR(4) NULL


-- ##
-- ## WITH NULLABILITY
-- ##

SET STATISTICS TIME ON
PRINT 'NOT IN - WITH NULLABILITY'
SELECT 
 Id
 , CompareColumn 
FROM 
 TestPerformance.LargerTable AS bt
WHERE 
 bt.CompareColumn NOT IN (
	 SELECT 
   LookupColumn 
	 FROM 
   TestPerformance.SmallerTable
)
SET STATISTICS TIME OFF
 
SET STATISTICS TIME ON
PRINT 'NOT EXISTS - WITH NULLABILITY'
SELECT 
 Id
 , CompareColumn
FROM 
 TestPerformance.LargerTable AS bt
WHERE 
 NOT EXISTS (
	 SELECT 
   0
	 FROM 
   TestPerformance.SmallerTable  AS st
	 WHERE 
   st.LookupColumn = bt.CompareColumn
)
SET STATISTICS TIME OFF

-- Conclusion:
-- NOT NULL columns: equal performance
-- NOT EXISTS is faster if columns has NULL
```

# Credits

http://sqlinthewild.co.za/index.php/2010/03/23/left-outer-join-vs-not-exists/
