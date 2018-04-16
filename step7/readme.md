## Retest System After Performance Tuning

First lets take a look at storage usage and see how it compares to the last time.

```sql
select stv_tbl_perm.name as "table", count(*) as "blocks (mb)"
from stv_blocklist, stv_tbl_perm
where stv_blocklist.tbl = stv_tbl_perm.id
and stv_blocklist.slice = stv_tbl_perm.slice
and stv_tbl_perm.name in ('customer', 'part', 'supplier', 'dwdate', 'lineorder')
group by stv_tbl_perm.name
order by 1 asc;

                                  table                                   | blocks (mb)
--------------------------------------------------------------------------+-------------
 customer                                                                 |         604
 dwdate                                                                   |         160
 lineorder                                                                |       26349
 part                                                                     |         200
 supplier                                                                 |         236
```

As you can see we've brought down the space consumption of `lineorder` greatly.

### Distribution Skew

Lets examine the distribution skew of tables to see if any slices are doing more work than others.

```sql
select trim(name) as table, slice, sum(num_values) as rows, min(minvalue), max(maxvalue)
from svv_diskusage
where name in ('customer', 'part', 'supplier', 'dwdate', 'lineorder')
and col =0
group by name, slice
order by name, slice;

   table   | slice |   rows   |   min    |    max
-----------+-------+----------+----------+-----------
 customer  |     0 |  3000000 |        1 |   3000000
 customer  |     2 |  3000000 |        1 |   3000000
 customer  |     4 |  3000000 |        1 |   3000000
 customer  |     6 |  3000000 |        1 |   3000000
 dwdate    |     0 |     2556 | 19920101 |  19981230
 dwdate    |     2 |     2556 | 19920101 |  19981230
 dwdate    |     4 |     2556 | 19920101 |  19981230
 dwdate    |     6 |     2556 | 19920101 |  19981230
 lineorder |     0 | 75029991 |        3 | 599999975
 lineorder |     1 | 75059242 |        7 | 600000000
 lineorder |     2 | 75238172 |        1 | 599999975
 lineorder |     3 | 75065416 |        1 | 599999973
 lineorder |     4 | 74801845 |        3 | 599999975
 lineorder |     5 | 75177053 |        1 | 599999975
 lineorder |     6 | 74631775 |        1 | 600000000
 lineorder |     7 | 75034408 |        1 | 599999974
 part      |     0 |   175006 |       15 |   1399997
 part      |     1 |   175199 |        1 |   1399999
 part      |     2 |   175441 |        4 |   1399989
 part      |     3 |   175000 |        3 |   1399995
 part      |     4 |   175018 |        5 |   1399979
 part      |     5 |   175091 |       11 |   1400000
 part      |     6 |   174253 |        2 |   1399969
 part      |     7 |   174992 |       13 |   1399996
 supplier  |     0 |  1000000 |        1 |   1000000
 supplier  |     2 |  1000000 |        1 |   1000000
 supplier  |     4 |  1000000 |        1 |   1000000
 supplier  |     6 |  1000000 |        1 |   1000000
(28 rows)
```

You can see here that our dist style of ALL for `customer` puts it on only one slice per node, the same holds true for `dwdate` and `supplier`


### QUERY PLANS

Lets re-run the explain to see if we've eliminated some of the network activity

```sql
explain
select sum(lo_revenue), d_year, p_brand1
from lineorder, dwdate, part, supplier
where lo_orderdate = d_datekey
and lo_partkey = p_partkey
and lo_suppkey = s_suppkey
and p_category = 'MFGR#12'
and s_region = 'AMERICA'
group by d_year, p_brand1
order by d_year, p_brand1;
```

```sql
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 XN Merge  (cost=1000014259964.14..1000014259964.84 rows=280 width=20)
   Merge Key: dwdate.d_year, part.p_brand1
   ->  XN Network  (cost=1000014259964.14..1000014259964.84 rows=280 width=20)
         Send to leader
         ->  XN Sort  (cost=1000014259964.14..1000014259964.84 rows=280 width=20)
               Sort Key: dwdate.d_year, part.p_brand1
               ->  XN HashAggregate  (cost=14259952.05..14259952.75 rows=280 width=20)
                     ->  XN Hash Join DS_DIST_ALL_NONE  (cost=30673.83..14224628.62 rows=4709791 width=20)
                           Hash Cond: ("outer".lo_orderdate = "inner".d_datekey)
                           ->  XN Hash Join DS_DIST_ALL_NONE  (cost=30641.88..14118626.38 rows=4709791 width=20)
                                 Hash Cond: ("outer".lo_suppkey = "inner".s_suppkey)
                                 ->  XN Hash Join DS_DIST_NONE  (cost=17640.00..13758507.64 rows=24001516 width=24)
                                       Hash Cond: ("outer".lo_partkey = "inner".p_partkey)
                                       ->  XN Seq Scan on lineorder  (cost=0.00..6000378.88 rows=600037888 width=16)
                                       ->  XN Hash  (cost=17500.00..17500.00 rows=56000 width=16)
                                             ->  XN Seq Scan on part  (cost=0.00..17500.00 rows=56000 width=16)
                                                   Filter: ((p_category)::text = 'MFGR#12'::text)
                                 ->  XN Hash  (cost=12500.00..12500.00 rows=200750 width=4)
                                       ->  XN Seq Scan on supplier  (cost=0.00..12500.00 rows=200750 width=4)
                                             Filter: ((s_region)::text = 'AMERICA'::text)
                           ->  XN Hash  (cost=25.56..25.56 rows=2556 width=8)
                                 ->  XN Seq Scan on dwdate  (cost=0.00..25.56 rows=2556 width=8)
```

You can see here there was no re-distribution done to the tables. WOO!

### RUNNING TEST QUERIES

First turn off the caching for this session

```sql
set enable_result_cache_for_session to off;
```

We can then re-run our test queries

```sql
-- Query 1
-- Restrictions on only one dimension.
select sum(lo_extendedprice*lo_discount) as revenue
from lineorder, dwdate
where lo_orderdate = d_datekey
and d_year = 1997
and lo_discount between 1 and 3
and lo_quantity < 24;


Time: 853.559 ms

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

Time: 4958.231 ms

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

Time: 897.132 ms
```

You can see we've dropped in query time execution by about 30%!
