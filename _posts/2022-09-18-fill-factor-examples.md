---
layout: post
title: "Fill factor examples"
date: 2022-09-18
---

# Introduction
In the previous post, I explained what the fill factor is and how it impacts page fullness and fragmentation. In this post, I will show a few examples of indexes with different values of this parameter.

If you haven't read the previous post yet, you can read it [here]({% post_url 2022-08-19-fill-factor %}).

# Checking fragmentation
Fragmentation of indexes on a given table can be checked using the following script (adjust the value of the @tableUsers variable as needed)
```sql
DECLARE @tableName AS varchar(100)
SET @tableName = 'Users' 

SELECT
     IDX.[name] AS Index_Name,
     IDXPS.index_type_desc AS Index_Type,
     IDXPS.alloc_unit_type_desc AS Allocation_Unit_Type,
     IDXPS.avg_fragmentation_in_percent AS Fragmentation_Percentage,
         (case when IDXPS.avg_fragmentation_in_percent > 30 then 'rebuild'
         when IDXPS.avg_fragmentation_in_percent > 10 then 'reorganize'
         else ' - '
         end) AS Recommended_Action,
     IDX.fill_factor AS [FillFactor],
     IDXPS.avg_page_space_used_in_percent AS PageFullness
FROM sys.indexes IDX
LEFT OUTER JOIN sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') as IDXPS
    ON IDX.object_id = IDXPS.object_id AND IDX.index_id = IDXPS.index_id
WHERE OBJECT_NAME(IDXPS.object_id) = @tableName
ORDER BY Fragmentation_Percentage DESC
```

I will use this script in the following examples for showing fragmentation and page fullness after each operation.

# Sample data
I used the Bogus library for seeding database tables with random data. You can find the source code [here].

# Example 1
The first example is an index on an integer column with sequential values. The fill factor has a default value of 100%. 

The table can be created with the following script
```sql
CREATE TABLE [dbo].[Users](
    [Id] [int] IDENTITY(1,1) NOT NULL,
    [FirstName] [varchar](50) NOT NULL,
    [LastName] [varchar](50) NOT NULL,
 CONSTRAINT [PK__Users] PRIMARY KEY CLUSTERED ([Id] ASC) 
 )
```

1. After inserting 10,000 rows of sample data we see low fragmentation of 2.5% and high page fullness of almost 100%. This is what we should expect - as we insert sequential values, the index grows at the end. No page splits are performed, except at the end of the index. 
![Index with 10k records](/assets/images/2022-09-18/example1_1.PNG)

2. We can rebuild the index with the following script
```sql
ALTER INDEX PK__Users ON Users REBUILD
```
After doing that we can see a slight decrease in fragmentation, but overall there is no big impact on the index. 
![Index after rebuild](/assets/images/2022-09-18/example1_2.PNG)

3. Inserting 100 new records, or 1% of new data, increases fragmentation to 2.5% and decreases page fullness by 1% to 98%
![Index after inserting 100](/assets/images/2022-09-18/example1_3.PNG)

4. Inserting 1000 new records, or 10% of new data, has a similar impact on the index. 
![Index after inserting 1000](/assets/images/2022-09-18/example1_4.PNG)

5. Doubling the number of records in the table, by inserting 10,000 new records also has no significant impact on fragmentation and page fullness. 
![Index after inserting 10000](/assets/images/2022-09-18/example1_5.PNG)

As we can see, if the indexed column has sequential values, the default value of the fill factor equal to 100% works fine. 

# Example 2
The second example is similar to the previous one, but this time the index has a fill factor equal to 80%. 

The following script creates a table for this example.
```sql
CREATE TABLE [dbo].[Users](
    [Id] [int] IDENTITY(1,1) NOT NULL,
    [FirstName] [varchar](50) NOT NULL,
    [LastName] [varchar](50) NOT NULL,
 CONSTRAINT [PK__Users] PRIMARY KEY CLUSTERED ([Id] ASC) WITH (FILLFACTOR = 80)
 )
```

1. Inserting initial 10,000 records results in low fragmentation of 2.5% and high page fullness of almost 100%, same as previously. 
![Index with 10k records](/assets/images/2022-09-18/example2_1.PNG)
2. Rebuilding the index lowers the page fullness to 79%. This is what we should expect with a fill factor of 80%. 
![Index after rebuild](/assets/images/2022-09-18/example2_2.PNG)
3. Inserting 100 records increases page fullness slightly to 80%.
![Index after inserting 100](/assets/images/2022-09-18/example2_3.PNG)
4. Inserting 1000 records further increases page fullness to 81%. 
![Index after inserting 1000](/assets/images/2022-09-18/example2_4.PNG)
5. After inserting 10,000 new records, page fullness reaches 89%. This is because the pages containing the newly inserted data have page fullness close to 100%, and pages containing the initial data still have page fullness of around 80%, resulting in an overall value of about 90%. Since new data is inserted at the end of the index, 20% of space reserved for new records during the index rebuild is not used. 
![Index after inserting 10000](/assets/images/2022-09-18/example2_5.PNG)

For an index on a column containing sequential values, lowering the fill factor gives worse results than the default value of 100%. New records are not inserted into the space reserved during a rebuild. 

# Example 3
The next two examples are indexes on columns with random values (uniqueidentifier, aka. GUID). First, let's consider an index with a default fill factor of 100%. 

This script creates a table `Customers` with an index on autogenerated uniqueidentifier column. 
```sql
CREATE TABLE [dbo].[Customers](
    [Id] [uniqueidentifier] DEFAULT (NEWID()) NOT NULL,
    [FirstName] [varchar](50) NOT NULL,
    [LastName] [varchar](50) NOT NULL,
 CONSTRAINT [PK__Customers] PRIMARY KEY CLUSTERED ([Id] ASC) 
 )
```


1. After inserting an initial 10,000 records index has an extremely high fragmentation of 98%
![Index with 10k records](/assets/images/2022-09-18/example3_1.PNG)
2. Index rebuild solves this problem - there is no fragmentation, and page fullness has a very high value of 99.5%.
![Index after rebuild](/assets/images/2022-09-18/example3_2.PNG)
3. Inserting just 100 new records increases fragmentation to 94%. This is a terrible result! Adding only 1% of new data to the table completely demolished the index. This is because, with random values, new records are inserted not at the end, but all over the index, resulting in many page splits. One hundred new records are enough to cause a page split of almost all pages. 
![Index after inserting 100](/assets/images/2022-09-18/example3_3.PNG)
4. Inserting 1000 records yields similar results. Page fullness is slightly bigger because more records are inserted into the new pages after page splits. 
![Index after inserting 1000](/assets/images/2022-09-18/example3_4.PNG)
5. Inserting 10,000 records has a similar impact, with a bit higher page fullness. 
![Index after inserting 10000](/assets/images/2022-09-18/example3_5.PNG)

An index on a column with random values and a fill factor of 100% works really badly, inserting a small number of new records degrades the index fragmentation completely.


# Example 4
The last example is the same as the previous one, but with a fill factor of 80%. 

The table for this example can be created with the following script.
```sql
CREATE TABLE [dbo].[Customers](
    [Id] [uniqueidentifier] DEFAULT (NEWID()) NOT NULL,
    [FirstName] [varchar](50) NOT NULL,
    [LastName] [varchar](50) NOT NULL,
 CONSTRAINT [PK__Customers] PRIMARY KEY CLUSTERED ([Id] ASC) WITH (FILLFACTOR = 80)
 )
```
1. Inserting initial 10,000 records results in a very high fragmentation, same as previously. 
![Index with 10k records](/assets/images/2022-09-18/example4_1.PNG)
2. Again, a rebuild solves the issue - fragmentation is 0% and page fullness is about 80%
![Index after rebuild](/assets/images/2022-09-18/example4_2.PNG)
3. This time, however, inserting 100 new records doesn't increase the fragmentation at all. All that changes is slightly bigger page fullness - new records are inserted into the reserved space on the existing pages. 
![Index after inserting 100](/assets/images/2022-09-18/example4_3.PNG)
4. Even inserting 1000 new records doesn't cause page splits and fragmentation is still 0. Page fullness grows further as more of the space on the existing pages is filled with new data.
![Index after inserting 1000](/assets/images/2022-09-18/example4_4.PNG)
5. Only after doubling the number of records in the table we can see the same effects as previously - high fragmentation and lowered page fullness. 
![Index after inserting 10000](/assets/images/2022-09-18/example4_5.PNG)

As we can see, changing the fill factor to a lower value largely solved the issue of fragmentation of this index. The problem reappears only after inserting a large amount of data. 

# Summary
In this post, I showed a few practical examples of using the fill factor and how it impacts the index's performance. The last example is a perfect use case for changing this parameter - an index on a column of random values. 
