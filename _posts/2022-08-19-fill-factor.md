---
layout: post
title: "Fill factor "
date: 2022-08-19
---

# Introduction
In the previous post, I explained what is index fragmentation, and how to decrease it by reorganizing or rebuilding the index. In this post, I will present a second solution to the problem of high fragmentation: changing the fill factor. 

# Fill factor
The fill factor is a parameter that determines what percentage of data pages in the index's leaf nodes will be filled with data, and what percentage will be left unused after the index is rebuilt. The default value of the fill factor is 100%, meaning all available space is filled with data and none is left unused. 

To illustrate this concept, let's compare two indexes that differ only by fill factor. For the sake of this explanation, we will assume that every data page can contain up to 10 entries. 

The picture below shows an index with the default fill factor of 100%. Every page, except the last one, is filled with data. The last page isn't full, as the index grows, new entries will be added there. 
![Index with fill factor 100](/assets/images/2022-08-19/ff100-sequential1.png)

This is an index with a fill factor set to 80%. Note that every page (except the last) contains 8 entries, leaving enough empty space for two more. 
![Index with fill factor 80](/assets/images/2022-08-19/ff80-sequential1.png)


# Why change fill factor
As mentioned before, the default value of the fill factor is 100%. Why would we want to change it? 

In the previous post I explained, that if there isn't enough space on the data page to insert a new entry, a page split will be performed, resulting in high fragmentation. If the indexed column has sequential values, such as automatically-generated incremental integer IDs, this will not happen. New entries will be inserted at the end of the index, and page splits will happen only there. However, if the indexed column has random values, such as GUID (globally unique identifier, also known as UUID, universally unique identifier), new entries will be inserted into all existing data pages, resulting in many page splits. 


Result of inserting new entries into an index on a column of sequential values
![Index with fill factor 100 with new values inserted](/assets/images/2022-08-19/ff100-sequential2.png)
As you can see, new entries are inserted into the last data page, not causing increased fragmentation.


Result of the same operation with a lower fill factor
![Index with fill factor 80 with new values inserted](/assets/images/2022-08-19/ff80-sequential2.png)
In this case, fragmentation is also low, but the empty space reserved for new entries is not used, lowering page fullness. Therefore, this is not a good use case for a lower fill factor. 


However, if the indexes column contains random values (like a GUID), the results will be completely different. 

The following picture shows an index on a GUID column (unique identifier in SQL Server naming convention) with the default fill factor of 100%. For readability, only the first three digits are shown. 
![Index with fill factor 100 with random values](/assets/images/2022-08-19/ff100-random1.png)
It looks more or similar to the index on the sequential integer column. 

However, when we try to insert new entries into it, the values are inserted into random data pages, not the page at the last node. Remember: the values are random, so their place in the index is also random. 
![Index with fill factor 100 with random values - insert](/assets/images/2022-08-19/ff100-random2.png)

This will result in many page splits, and very high fragmentation. Also as you can see, after page splits, every page is roughly half empty, lowering the index's page fullness. 
![Index with fill factor 100 with random values - page splits](/assets/images/2022-08-19/ff100-random3.png)
Because of high fragmentation, the diagram is now a mess - arrows connecting leaf nodes are all over it. 


An index on a column with random values and a fill factor of 80% is shown below. Same as previously with sequential values, 20% of space in data pages is left empty for new entries. 
![Index with fill factor 80 with random values](/assets/images/2022-08-19/ff80-random1.png)

If we add new entries, they will be inserted into the reserved space. No page splits, or very few, will be done. 
![Index with fill factor 80 with random values - insert](/assets/images/2022-08-19/ff80-random2.png)
This is the main use for the fill factor parameter: reserving space in an index on a column of random values to limit page splits and resulting index fragmentation.


# Downsides of changing the fill factor
While a lower fill factor can help to solve some issues, there are also disadvantages of changing it:
- Some space in the data pages is left unused, so more data pages are required to store the same amount of data. As a result, the index size is bigger.
- More IO operations are required to read the same number of entries since every page read will fetch fewer of them. 

Those are essentially the same issues that low page fullness causes. Make sure that changing the fill factor does not have a negative impact on page fullness in your case. 

# How to check the current value of the fill factor
The current value of the fill factor can be checked both in the UI and by executing a script.

## The UI
In Microsoft SQL Server Management Studio go to Object Explorer, right-click on the index name, and select Properties.

![Fill factor parameter](/assets/images/2022-08-19/fill-factor-object-explorer.png)

In the left-hand side menu select Options. The fill factor is displayed in the main pane, under Storage. Note that 0 means 100% - this is a special case, other values are displayed as you would expect, e. g. 80% as 80.

## The script
The fill factor value can be accessed by the sys.indexes view. The following script gets the fill factor by index name.

```sql
select fill_factor
from sys.indexes
where [name] = 'IDX_EmployeeNumber'
```
Keep in mind that the index name doesn't have to be unique across the whole database.


# How to change the fill factor
The fill factor can be specified both during index creation and during a rebuild.

## The UI
Go to the index's Properties as described above, set the desired value and click OK. You will see a confirmation dialog saying that the index has to be rebuilt. 

## The script
To provide the fill factor value during index creation, add the WITH clause to the CREATE INDEX statement

```sql
CREATE INDEX IDX_EmployeeNumber ON Employees (EmployeeNumber) WITH (FILLFACTOR = 80);
```

It can also be specified when rebuilding an index, like in the following script

```sql
ALTER INDEX IDX_EmployeeNumber ON Employees REBUILD WITH (FILLFACTOR = 80);
```

## What are reasonable values
The default value of the fill factor is 100%. 

Values between 99% and 80% can solve fragmentation issues in many common scenarios. You can experiment by setting the value to 95%, monitoring performance, then trying e.g. 90%. 

If you want to set the fill factor lower than 80%, you really need to know what you are doing. At this point, costs may outweigh the benefits, and changing the fill factor may not be a good solution to the problem at hand. 

# Summary
In this post,  I explained what the fill factor is, and how to use it to decrease the index fragmentation. I also showed how it impacts indexes using columns with random values and with sequential values. 
Changing the fill factor value is a bit more advanced technique. It can have an adverse impact on performance and should be used sparingly. 

# References
Microsoft Docs - [fill factor](https://docs.microsoft.com/en-us/sql/relational-databases/indexes/specify-fill-factor-for-an-index?view=sql-server-ver16)

