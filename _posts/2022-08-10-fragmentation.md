---
layout: post
title: "Index fragmentation and page fullness"
date: 2022-08-10
---

# Introduction
This is the third post in a series on SQL databases. In this post, I will cover index fragmentation and page fullness: what they are, what issues are related to them, and how to solve those issues.

# What is index fragmentation
Index fragmentation is a measure of the difference between the logical ordering of index pages and the physical ordering of index pages. It's expressed as a percentage value: 0 means that logical and physical orders are the same, and 100 means that they are completely different. Typically, when talking about index fragmentation there is no need to interpret the exact value, like "what exactly does it mean that fragmentation is equal to 17.4% ?", instead it's understood as "low fragmentation good, high fragmentation bad" with some rule of thumb for what is too high. Index fragmentation changes all the time, as new records are inserted into the indexed table, existing records are updated or deleted, and corresponding changes are made to the index. 

# Page fullness
Page fullness is a measure of the percentage of index data pages on average filled with actual data. 100% means that all pages are filled with index data, and 50% means that half of the space in memory pages is unused. A 0 would mean that all space is unused. Similarly, as with index fragmentation, we generally do not care about exact values, just general interpretations like "100% good, low values bad". Page fullness also changes as new records are inserted into and deleted from the indexed table. A newly created (or rebuild) index typically has page fullness close to 100%, often not exactly 100% but 98-99%. This is because the last data page is filled partially. 

# What causes high fragmentation and low page fullness
Typical issues with indexes are high fragmentation and low page fullness. 
They are most commonly caused by page splits. When a new row is inserted into a database table, a new entry has to be inserted into the index. But sometimes the memory page where it should be inserted is already full. In that case, the DB engine does the following
1. allocates a new page
2. inserts it (in logical ordering) after the original one
3. moves half of existing entries to the new page to achieve more equal data distribution
4. inserts the new entry

The results of that are that:
- The new page, which was physically allocated somewhere else, is now logically placed right next to the old one. So in logical ordering, they are next to each other, but physically they are not. This is exactly what fragmentation is. The more page splits are done, the higher fragmentation of the whole index becomes. 
- While the old page was full, after the split each of the two pages is filled roughly in half. 

## Page Split
Let's consider the following example. We have an index on a column of type int. The physical order and the logical order of pages are the same, so fragmentation is 0%. All data pages are filled with entries, therefore page fullness is 100%. 

Since this is a nonclustered index, entries in leaf pages contain addresses of nodes in the clustered index. For readability, only some nodes of the index tree are shown. 
![Index before a page split](/assets/images/2022-08-10/page-split1.png)


A new entry with a key equal to 61 needs to be inserted. The DB engine searches for an appropriate place to insert it:
- first compares it with values in the root node and finds a subtree containing values between 57 and 87
- then descends to the level below, compares it with values in the internal node, and finds a page containing values between 57 and 62
- finally tries to insert it at the end of the page, but there isn't enough space - the page is already full

![Index insert](/assets/images/2022-08-10/page-split2.png)


In that case, the DB engine performs a page split:
- a new page is allocated (shown in green)
- half of the existing entries (59 and 60) are moved to the new page
- a new entry is inserted in the parent node (value of 59 pointing to the new page)
- pointers creating a linked list of leaf nodes are added (green arrows)
- the new value of 61 is inserted on the new page (blue entry)

![Index after a page split](/assets/images/2022-08-10/page-split3.png)

As we can see, after this operation:
- page 57 is half empty (low page fullness)
- page 59 also contains some empty space (low page fullness)
- page 59 is logically located between pages 57 and 62, but physically located somewhere else (increased fragmentation)


# Why are high fragmentation and low page fullness an issue
The main reason fragmentation is an issue is that for many storage technologies, Hard Drive Disks (HDD) being one such example, accessing adjacent blocks of memory is a lot faster than reading the same number of memory blocks allocated randomly. Performing one read of 10 sequential memory pages is a lot faster than performing 10 reads of 1 page each. For queries involving few read operations, this difference may be negligible. However the bigger part of the index is scanned, the more significant the difference becomes. 

If the page fullness value is low, the DB engine has to read more memory pages to access a given number of index entries. Consider the following example: one page can contain at most 100 entries, and 600 entries have to be read to execute the query. If page fullness is at 100%, 6 pages will be loaded. However, If page fullness is at 75%, then the query executor will load 600 / (100 * 0.75) = 8 pages, or 33% more than in the previous case. By the same logic, low page fullness negatively impacts caching - only 75% of a page stored in the cache is used. 

# How to lower fragmentation
Two solutions may help us to solve those issues. The first is performing index maintenance operations: a reorganize or a rebuild. The second is decreasing the fill factor. 

## Reorganize and Rebuild
Reorganize is an index maintenance operation that changes the order of leaf nodes in the B-tree structure of the index. It's performed online, which means that insert, update and delete operations can be run on the indexed table while it is in progress. 

Rebuild drops and recreates the index. This operation can change the whole structure of the index, including inner nodes, not only the leaf nodes. Because of that, in some cases, it can lower fragmentation more than reorganize. The downside is that it can have a more negative impact on the performance of queries executed on the indexed table while it is in progress. Rebuild can be done as an online or offline operation.

The trade-off between rebuild and reorganize is a choice between the impact the operation can have on index structure once it's done (bigger for rebuild) and the negative impact on executed queries while it is performed (lower for reorganize).

## Change fill factor
The second possible solution is to change the index's fill factor. This solution applies primarily to indexes in which the key is a randomly generated, non-sequential value. I will cover it in detail in the next post. 


# Example
In this example, I will show how to check the index's fragmentation and page fullness. 

We will start by creating a sample database table and a nonclustered index. The table will contain some basic information about employees, such as name and date of birth, an autogenerated identity column, and a column named EmployeeNumber. In this example, the employee number is a GUID (unique identifier), but it can be understood as an equivalent of a Social Security Number, Personal ID Number, or another real-world identifier for a person. A common requirement is to search for a person in a DB based not on the table's ID column, but based on this kind of real-world ID number, e.g. get an employee by his SSN. 

The following script creates a table and a nonclustered index on the EmployeeNumber column.
```sql
CREATE TABLE [dbo].[Employees](
    [Id] [int] IDENTITY(1,1) NOT NULL,
    [FirstName] [varchar](50) NOT NULL,
    [LastName] [varchar](50) NOT NULL,
    [DateOfBirth] [date] NOT NULL,
    [Salary] [int] NOT NULL,
    [Position] [varchar](50) NOT NULL,
    [EmployeeNumber] [uniqueidentifier] NOT NULL,
    CONSTRAINT [PK__Employee] PRIMARY KEY CLUSTERED ([Id] ASC)
)


CREATE NONCLUSTERED INDEX [IDX_EmployeeNumber] ON [dbo].[Employees]
(
    [EmployeeNumber] ASC
)
```

After running this script you should see the following objects in the SQL Management Studio Object Explorer

![Object explorer](/assets/images/2022-08-10/object-explorer.png)



For seeding the table I wrote a script using the [Bogus package](https://github.com/bchavez/Bogus) and [Dapper](https://github.com/DapperLib/Dapper)
```csharp
static void Main()
{
    Randomizer.Seed = new Random(1234);

    var testEmployees = new Faker<Employee>()
        .RuleFor(e => e.EmployeeNumber, f => f.Random.Uuid())
        .RuleFor(e => e.FirstName, f => f.Name.FirstName())
        .RuleFor(e => e.LastName, f => f.Name.LastName())
        .RuleFor(e => e.DateOfBirth, f => f.Date.Past())
        .RuleFor(e => e.Salary, f => f.Random.Number(1000, 10000))
        .RuleFor(e => e.Position, f => f.Name.JobTitle());

    var employees = testEmployees.Generate(10000);


    using (var connection = new SqlConnection(@"Server=DESKTOP-BK;Initial Catalog=TestDb1;Integrated Security=true;"))
    {
        connection.Open();

        var identity = connection.Insert(employees);
    }
}
```
You can see the full source code [here](https://github.com/BartekSTRX/blog-examples). If you want to run it locally, remember to change the server name and the DB name in the connection string. 

## Measuring fragmentation and page fullness
Values of index fragmentation and page fullness can be checked both in the UI and by running an SQL script

### The UI
Right-click on the index name in the Object Explorer and go to Properties, select Fragmentation in the left-hand side menu, and in the main pane, you will see the Page fullness and Total fragmentation statistics. 
![UI fragmentation](/assets/images/2022-08-10/properties-fragmentation.PNG)

In this case, the index has an extremely high fragmentation of 97.78% and page fullness of 71.36%.

### The script
The following script retrieves basic data about all indexes for a given table. It uses the sys.indexes view and the sys.dm_db_index_physical_stats dynamic management view. 

```sql
SELECT
     IDX.[name] AS Index_Name,
     IDXPS.index_type_desc AS Index_Type,
     IDXPS.alloc_unit_type_desc AS Allocation_Unit_Type,
     IDXPS.avg_fragmentation_in_percent AS Fragmentation_Percentage,
         (case when IDXPS.avg_fragmentation_in_percent > 30 then 'rebuild'
         when IDXPS.avg_fragmentation_in_percent > 10 then 'reorganize'
         else 'ok'
         end) AS Recommended_Action,
     IDX.fill_factor AS [FillFactor],
     IDXPS.avg_page_space_used_in_percent AS PageFullness
FROM sys.indexes IDX
LEFT OUTER JOIN sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') as IDXPS
    ON IDX.object_id = IDXPS.object_id AND IDX.index_id = IDXPS.index_id
WHERE OBJECT_NAME(IDXPS.object_id) = 'Employees'
ORDER BY Fragmentation_Percentage DESC
```

After running this script you should see a result like below
![Script fragmentation](/assets/images/2022-08-10/script-fragmentation.PNG)

It shows statistics for all indexes for the Employees table, both clustered and nonclustered. It also shows a "recommended action" for each of them. In this case for indexes with fragmentation above 10%, it recommends a reorganize, and for those above 30%, it recommends a rebuild.

## Rebuild and reorganize
The following script shows SQL commands for executing a reorganize and a rebuild of the IDX_EmployeeNumber index on the Employees table. 

```sql
ALTER INDEX IDX_EmployeeNumber ON Employees REORGANIZE;

ALTER INDEX IDX_EmployeeNumber ON Employees REBUILD
```

Executing the previous script again shows that after running a rebuild fragmentation is close to 0% and page fullness is close to 100%.

![Script fragmentation - rebuild](/assets/images/2022-08-10/script-fragmentation-rebuild.PNG)


# Summary
In this post, I explained what index fragmentation and page fullness are, how to check their values, and how to solve performance issues they can cause. In the next post, I will cover a related topic of fill factor. 

# References
- Microsoft Docs - [reorganize and rebuild](https://docs.microsoft.com/en-us/sql/relational-databases/indexes/reorganize-and-rebuild-indexes?view=sql-server-ver16)
- [Bogus](https://github.com/bchavez/Bogus)
- [Dapper](https://github.com/DapperLib/Dapper)