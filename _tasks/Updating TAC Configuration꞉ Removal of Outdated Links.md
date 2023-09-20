---
favorited: true
title: 'Removal of Outdated Links'
created: '2023-09-20T15:20:58.001Z'
modified: '2023-09-20T15:43:37.383Z'
layout: default
---

# Updating TAC Configuration: Removal of Outdated Links
This guide details the steps to remove outdated links from the Teacher Access Center (TAC). Instead of the manual approach of removing links for each school via its webpage, we've adopted an automated SQL approach.

## 1. Connecting to the database
- **Server**: `ESP-HAL` (Also referred to as the eSchool database)
- **Database**: 
  - Test Environment: `PLA_Test`
  - Production Environment: `PLA_Live` (Switch to this post-testing)
- **Authentication**: Use your Windows credentials.


## 2. **Identifying Links for Removal**
I explored various tables to locate the data associated with the outdated links. The pertinent table was identified as `TAC_LINK`.
```sql
SELECT * FROM TAC_LINK
WHERE LINK_URL LIKE 'http://ssrs.psd202.org/Reports/Pages/Report.aspx%'
```
For backup purposes, in case a restoration becomes necessary, the data was saved to:
- Live environment: `\\dd-7\DTC-Shared\SIS\SQL Qs\Andrew M\RemovingLinks\LiveLog.csv`
- Test environment: `\\dd-7\DTC-Shared\SIS\SQL Qs\Andrew M\RemovingLinks\TestLog-2.csv`

## 3. Execution of the Deletion

{: .warning }
Ensure to always backup and test any changes in a safe environment before making modifications to the live database.

Upon confirming that the fetched data matches our deletion criteria, I implemented the following delete query:
```sql
DELETE FROM TAC_LINK
WHERE LINK_URL LIKE 'http://ssrs.psd202.org/Reports/Pages/Report.aspx%'
```
Post-execution, the targeted rows were successfully deleted.




