## Sort Keys

When creating a table we can select one or more columns as a sort key. Redshift stores the data on disk sorted according to these keys. How the data is sorted impacts Disk I/O and most notably query performance.

### Best practices for sort keys

* If recent data is queried most frequently, specify the timestamp column as the leading column for the sort key.

* If you do frequent range filtering or equality filtering on one column, specify that column as the sort key.

* If you frequently join a (dimension) table, specify the join column as the sort key.

## To Select Sort Keys

Evaluate queries to find timestamp columns used to filter results. Our lineorder table frequently uses equality filters using the orderdate.

```sql
where lo_orderdate = d_datekey and d_year = 1997
```

Look for columns that are used in range and equality filters. Our lineorder table also uses the orderdate for range filtering.

```sql
where lo_orderdate = d_datekey and d_year >= 1992 and d_year <= 1997
```

From what we've seen `lo_orderdate` is a good candidate for a sort key. Our other tables are dimensions so we can use their primary keys as sort keys.



| Table name | Sort Key	|
| ---------- | -------- |
| LINEORDER	 | lo_orderdate	|
| PART | p_partkey |
| CUSTOMER | c_custkey |
| SUPPLIER |	s_suppkey	|
| DWDATE | d_datekey |

Next we'll move onto distribution styles
