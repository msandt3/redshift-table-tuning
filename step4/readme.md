# Selecting Distribution Styles

Redshift distributes the rows ot a table to each of its underlying node slices based on the tables distribution style. In our example each of our four nodes has two slices for a total of eight. This is very similar to how citus collocates tenant data.

When executing a query the optimizer redistributes the rows to the compute nodes as needed to perform joins and aggregations.

Distribution styles should be assigned to achieve these goals

* Collocate rows from joining tables. When rows for joining columns are on the same slices, less data needs to be moved during execution.

* Distribute data evenly among slices in a cluser. This means workload can be allocated evenly among the slices.

# Distribution Styles


### KEY distribution
Rows are distributes according to the values in one column. The leader node will attempt to place matching values on the same node slice. Distributing a pair of tables on the joining keys will cause the leader node to collacte these rows.

### ALL distribution
A copy of the entire table is distributed to every node. Ensures every row is collocated for every join. Has potential space impact.

### EVEN Distribution
Rows are distributed across the slices in a round robin fashion. Candidate when a table does not participate in joins or there is not a clear choice between KEY or ALL distribution.

## In Practice

Let's examine something in practice by examining a query plan for a query we previously executed.

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

It produces

```
 XN Merge  (cost=1037037856269.59..1037037856270.29 rows=280 width=20)
   Merge Key: dwdate.d_year, part.p_brand1
   ->  XN Network  (cost=1037037856269.59..1037037856270.29 rows=280 width=20)
         Send to leader
         ->  XN Sort  (cost=1037037856269.59..1037037856270.29 rows=280 width=20)
               Sort Key: dwdate.d_year, part.p_brand1
               ->  XN HashAggregate  (cost=37037856257.50..37037856258.20 rows=280 width=20)
                     ->  XN Hash Join DS_BCAST_INNER  (cost=30654.76..37037821623.22 rows=4617904 width=20)
                           Hash Cond: ("outer".lo_orderdate = "inner".d_datekey)
                           ->  XN Hash Join DS_BCAST_INNER  (cost=30622.81..36628757688.43 rows=4617904 width=20)
                                 Hash Cond: ("outer".lo_suppkey = "inner".s_suppkey)
                                 ->  XN Hash Join DS_BCAST_INNER  (cost=17640.00..13453758507.64 rows=24001516 width=24)
                                       Hash Cond: ("outer".lo_partkey = "inner".p_partkey)
                                       ->  XN Seq Scan on lineorder  (cost=0.00..6000378.88 rows=600037888 width=16)
                                       ->  XN Hash  (cost=17500.00..17500.00 rows=56000 width=16)
                                             ->  XN Seq Scan on part  (cost=0.00..17500.00 rows=56000 width=16)
                                                   Filter: ((p_category)::text = 'MFGR#12'::text)
                                 ->  XN Hash  (cost=12500.00..12500.00 rows=193122 width=4)
                                       ->  XN Seq Scan on supplier  (cost=0.00..12500.00 rows=193122 width=4)
                                             Filter: ((s_region)::text = 'AMERICA'::text)
                           ->  XN Hash  (cost=25.56..25.56 rows=2556 width=8)
                                 ->  XN Seq Scan on dwdate  (cost=0.00..25.56 rows=2556 width=8)
(22 rows)
```

We're keeping an eye out for DS_BCAST or DS_DIST labels. DS_BCAST_INNER in this context indicates the inner join was broadcast to every slice. While DS_DIST_BOTH would indicate that both the outer and inner join were distributed across all slices. We want to select distribution strategies that reduce or eliminate broadcast and distribution steps.

## Distributing the Data

Every table can have only one distribution key. Meaning only one pair of tables in the schema can be collocated on their common columns. In this case `lineorder` is our central fact table. Our second collocation candidate will be the largest of the dimension tables `part`

## But What About the Other Tables

If we can't collocate our other dimension tables with the fact table or other important joining tables, we can often improve performance by distributing across all nodes. Because the tables `customer`, `supplier` and `dwdate` are small and update infrequently we can choose to use the ALL distribution style.

## In Summary

We've selected distribution keys for our commonly joined tables so that we can achieve some level of performance gain.

| Tbl Name  | Sort Key     | Dist Style |
| --------- | ------------ | ---------- |
| LINEORDER	| lo_orderdate | lo_partkey |
| PART | p_partkey | p_partkey |
| CUSTOMER | c_custkey | ALL |
| SUPPLIER | s_suppkey | ALL |
| DWDATE | d_datekey | ALL |
