---
favorited: true
title: Closing Open Vectors
created: '2023-09-14T19:45:16.125Z'
modified: '2023-09-20T15:43:29.642Z'
layout: default
---

# Closing Open Programs (Vectors) for Specific Student Categories
---
I was tasked with closing specific open programs (vectors) for students who are no longer attending preschools. Specifically, these programs remained open for inactive students who had been in preschools and for active students who were not currently enrolled in preschools. This document details the steps I took to execute this task for future reference. All SQL queries used are documented below and are also saved in the following directory: `\\dd-7\DTC-Shared\SIS\SQL Qs\Andrew M\Closing Vectors`

## Problem information
The challenge was that certain programs (vectors) remained open for students even after they had left the preschool environment. This needed rectification. Here's the list of these open programs and the associated building IDs representing the preschools:
- Programs to close:
  - `BRTH3 - ISBE Birth to 3`
  - `CHILD - Early Childhood`
  - `ECP01 - ISBE Early Childhood Preschool for All`
  - `ECP02 - ISBE Early Childhood Preschool for All Expansion`
  - `ECP03 - ISBE Early Childhood Head Start`
  - `ECP04 - ISBE Early Childhood IDEA`
  - `ECP05 - ISBE Early Childhood Preschool Title I`
  - `ECP06 - ISBE Early Childhood Local District`
  - `ECP07 - ISBE Early Childhood Tution Based`
  - `FUND  - Early Childhood Services`

- Building ids to select (Preschools): `(1, 100, 110, 118, 119, 121, 122, 123, 124, 131, 132, 133, 134, 166, 182, 183, 184)` 

## Connecting to the database
- Database Server: `ESP-HAL` (also known as the eschool database)
- Database: `PLA_Test` (Use `PLA_Live` after testing) 
- Sign in using windows credentials

## Pulling Preschool IDs
I decided I didn't want to use that array of building ID's and instead, I wanted to dynamically pull all preschools. The query for this is fairly simple:
```sql
SELECT BUILDING FROM REG_BUILDING WHERE BUILDING_TYPE = 'PK'
```

## Preliminary Identification of Target Students
This query is designed to provide a broad overview of the student IDs who could be impacted. It includes inactive students previously in preschools and active students not currently in preschools. **Note**: This list still requires further filtering to identify students with the specific open vectors.
```sql
SELECT STUDENT_ID
FROM REG
WHERE (BUILDING IN (SELECT BUILDING FROM REG_BUILDING WHERE BUILDING_TYPE = 'PK') AND
        CURRENT_STATUS = 'I')
    OR (BUILDING NOT IN (SELECT BUILDING FROM REG_BUILDING WHERE BUILDING_TYPE = 'PK') AND
        CURRENT_STATUS = 'A')
```

## Viewing a Student's Building History
To view the building history of any student, use the query below. This will also show the building name. Replace `'id'` with the desired student ID:
```sql
SELECT REW.STUDENT_ID, REW.SCHOOL_YEAR, REW.ENTRY_DATE, REW.WITHDRAWAL_DATE, REW.BUILDING, RB.NAME AS School_Name
FROM REG_ENTRY_WITH REW
LEFT JOIN REG_BUILDING RB ON REW.BUILDING = RB.BUILDING
WHERE REW.STUDENT_ID = 'id'
```

## Determining Last Preschool Attendance Date
To determine the last day a student attended a preschool, use the following query. Replace `'id'` with the student's ID:
```sql
SELECT MAX(WITHDRAWAL_DATE)
FROM REG_ENTRY_WITH
WHERE BUILDING in (SELECT BUILDING FROM REG_BUILDING WHERE BUILDING_TYPE = 'PK')
  and REG_ENTRY_WITH.STUDENT_ID = 'id'
```

## Preview of Update
Before applying any updates, it's crucial to preview the changes. The query below identifies which programs will be updated and the proposed new end dates:
```sql
SELECT RP.STUDENT_ID,
       RP.PROGRAM_ID,
       RP.ROW_IDENTITY,
       RP.END_DATE AS current_end_date,
       (SELECT MAX(WITHDRAWAL_DATE)
        FROM REG_ENTRY_WITH
        WHERE BUILDING in (SELECT BUILDING FROM REG_BUILDING WHERE BUILDING_TYPE = 'PK')
          and REG_ENTRY_WITH.STUDENT_ID = RP.STUDENT_ID) AS proposed_new_end_date
FROM REG_PROGRAMS RP
WHERE RP.PROGRAM_ID IN ('BRTH3', 'CHILD', 'ECP01', 'ECP02', 'ECP03', 'ECP04', 'ECP05', 'ECP06', 'ECP07', 'FUND')
  AND RP.END_DATE IS NULL
  AND RP.STUDENT_ID IN (SELECT STUDENT_ID
                        FROM REG
                        WHERE (BUILDING IN (SELECT BUILDING FROM REG_BUILDING WHERE BUILDING_TYPE = 'PK') AND
                               CURRENT_STATUS = 'I')
                           OR (BUILDING NOT IN (SELECT BUILDING FROM REG_BUILDING WHERE BUILDING_TYPE = 'PK') AND
                               CURRENT_STATUS = 'A'))
ORDER BY RP.STUDENT_ID, RP.PROGRAM_ID;
```

## Updating the Database (*Important*)

{: .warning }
Prior to proceeding with this step, ensure a clear understanding of its consequences. The subsequent query will perform changes to the database, aligning with the previously previewed modifications.

```sql
begin transaction;

UPDATE RP
SET RP.END_DATE = (
    SELECT MAX(WITHDRAWAL_DATE)
    FROM REG_ENTRY_WITH
    WHERE BUILDING IN (SELECT BUILDING FROM REG_BUILDING WHERE BUILDING_TYPE = 'PK')
      AND REG_ENTRY_WITH.STUDENT_ID = RP.STUDENT_ID
)
FROM REG_PROGRAMS RP
WHERE RP.PROGRAM_ID IN ('BRTH3', 'CHILD', 'ECP01', 'ECP02', 'ECP03', 'ECP04', 'ECP05', 'ECP06', 'ECP07', 'FUND')
  AND RP.END_DATE IS NULL
  AND RP.STUDENT_ID IN (
        SELECT STUDENT_ID
        FROM REG
        WHERE (BUILDING IN (SELECT BUILDING FROM REG_BUILDING WHERE BUILDING_TYPE = 'PK') AND
               CURRENT_STATUS = 'I')
           OR (BUILDING NOT IN (SELECT BUILDING FROM REG_BUILDING WHERE BUILDING_TYPE = 'PK') AND
               CURRENT_STATUS = 'A')
    );

commit transaction;
```

