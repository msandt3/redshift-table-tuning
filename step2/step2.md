Now we're going to test performance to establish a baseline

First let's record the storage use. The following query measures how many 1MB blocks of disk space are used for each table.


```sql
select stv_tbl_perm.name as table, count(*) as mb
from stv_blocklist, stv_tbl_perm
where stv_blocklist.tbl = stv_tbl_perm.id
and stv_blocklist.slice = stv_tbl_perm.slice
and stv_tbl_perm.name in ('lineorder','part','customer','dwdate','supplier')
group by stv_tbl_perm.name
order by 1 asc;

 table |  mb
--------------------------------------------------------------------------+-------
 customer  |   224
 dwdate    |   160
 lineorder | 34345
 part      |   120
 supplier  |   104
```

Now we'll test the query performance. In order to get an accurate feel for this we need to turn caching off.

```sql
set enable_result_cache_for_session to off;
```

We'll also want to turn on timing so we can effectively benchmark

```sql
\timing
```

Next lets look at sample queries. Run each one twice to take compile time out of the equation. Take the second value and record it - we'll compare it to the query time after we optimize the tables

```sql
-- Query 1
-- Restrictions on only one dimension.
select sum(lo_extendedprice*lo_discount) as revenue
from lineorder, dwdate
where lo_orderdate = d_datekey
and d_year = 1997
and lo_discount between 1 and 3
and lo_quantity < 24;

Time: 5647.682 ms


-- Query 2
-- Restrictions on two dimensions

select sum(lo_revenue), d_year, p_brand1
from lineorder, dwdate, part, supplier
where lo_orderdate = d_datekey
and lo_partkey = p_partkey
and lo_suppkey = s_suppkey
and p_category = 'MFGR#12'
and s_region = 'AMERICA'
group by d_year, p_brand1
order by d_year, p_brand1;

Time: 5819.132 ms

-- Query 3
-- Drill down in time to just one month

select c_city, s_city, d_year, sum(lo_revenue) as revenue
from customer, lineorder, supplier, dwdate
where lo_custkey = c_custkey
and lo_suppkey = s_suppkey
and lo_orderdate = d_datekey
and (c_city='UNITED KI1' or
c_city='UNITED KI5')
and (s_city='UNITED KI1' or
s_city='UNITED KI5')
and d_yearmonth = 'Dec1997'
group by c_city, s_city, d_year
order by d_year asc, revenue desc;

Time: 6761.840 ms
```
