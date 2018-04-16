# Compression Encodings

Compression is a column-level operation that reduces the size of data when it is stored. It conserves space and reduces the size of data when read from storage, leading to less DISK I/O and better query performance.

## Reviewing Compression Encodings

First lets get an idea for how much space a particular table uses per column. We'll query the STV_BLOCKLIST system table to do so


```sql
select col, max(blocknum)
from stv_blocklist b, stv_tbl_perm p
where (b.tbl=p.id) and name ='lineorder'
and col < 17
group by name, col
order by col;

 col | max
-----+-----
   0 | 295
   1 | 128
   2 | 295
   3 | 295
   4 | 295
   5 | 249
   6 | 190
   7 |  10
   8 | 145
   9 | 263
  10 | 268
  11 | 139
  12 | 263
  13 | 294
  14 | 137
  15 | 248
  16 | 184
```

We can also examine what current encodings are applied to the table as follows:


```sql
select encoding
from pg_table_def
where tablename = 'lineorder';

 encoding
----------
 lzo
 lzo
 lzo
 lzo
 lzo
 lzo
 lzo
 lzo
 lzo
 lzo
 lzo
 lzo
 lzo
 lzo
 lzo
 lzo
 lzo
```

## Experimenting with Encodings
We'll create an identical table except with column encodings.

First lets create our test table for `lineorder.ship_mode`

```sql
create table encodingshipmode (
moderaw varchar(22) encode raw,
modebytedict varchar(22) encode bytedict,
modelzo varchar(22) encode lzo,
moderunlength varchar(22) encode runlength,
modetext255 varchar(22) encode text255,
modetext32k varchar(22) encode text32k);
```

Now we'll insert the data from the source table

```sql
insert into encodingshipmode
select lo_shipmode as moderaw, lo_shipmode as modebytedict, lo_shipmode as modelzo,
lo_shipmode as moderunlength, lo_shipmode as modetext255,
lo_shipmode as modetext32k
from lineorder where lo_orderkey < 200000000;
```

Finally we'll see how the compression affects space usage

```sql
select col, max(blocknum)
from stv_blocklist b, stv_tbl_perm p
where (b.tbl=p.id) and name = 'encodingshipmode'
and col < 6
group by name, col
order by col;

 col | max
-----+-----
   0 | 221
   1 |  26
   2 |  61
   3 | 192
   4 |  54
   5 | 105
```

## Suggested Encodings

We can ask redshift to analyze the schema for us and propose encodings

```
analyze compression lineorder;

   Table   |       Column       | Encoding | Est_reduction_pct
-----------+--------------------+----------+-------------------
 lineorder | lo_orderkey        | zstd     | 10.52
 lineorder | lo_linenumber      | zstd     | 60.66
 lineorder | lo_custkey         | zstd     | 18.00
 lineorder | lo_partkey         | zstd     | 21.53
 lineorder | lo_suppkey         | zstd     | 21.53
 lineorder | lo_orderdate       | zstd     | 35.57
 lineorder | lo_orderpriority   | bytedict | 60.66
 lineorder | lo_shippriority    | zstd     | 91.04
 lineorder | lo_quantity        | delta    | 47.54
 lineorder | lo_extendedprice   | zstd     | 18.02
 lineorder | lo_ordertotalprice | zstd     | 14.58
 lineorder | lo_discount        | zstd     | 58.45
 lineorder | lo_revenue         | zstd     | 17.71
 lineorder | lo_supplycost      | zstd     | 36.82
 lineorder | lo_tax             | zstd     | 59.71
 lineorder | lo_commitdate      | zstd     | 36.20
 lineorder | lo_shipmode        | bytedict | 59.14
```

You can see it chose bytedict as the encoding for the `lo_shipmode` column
