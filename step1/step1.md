First we need to create a dataset as seen here

https://docs.aws.amazon.com/redshift/latest/dg/tutorial-tuning-tables-create-test-data.html


Let's open up a psql connection to our warehouse instance to do so.

```bash
psql -h <your-redshift-db-host> -U <your-master-user> -d <your-database-name> -p 5439
```

Mine looks like this

```bash
psql -h test-redshift.cpsxc2fgncuj.us-east-1.redshift.amazonaws.com -U mikesandt -d warehouse -p 5439
```

Next lets create our schema

```sql
CREATE TABLE part
(
  p_partkey     INTEGER NOT NULL,
  p_name        VARCHAR(22) NOT NULL,
  p_mfgr        VARCHAR(6) NOT NULL,
  p_category    VARCHAR(7) NOT NULL,
  p_brand1      VARCHAR(9) NOT NULL,
  p_color       VARCHAR(11) NOT NULL,
  p_type        VARCHAR(25) NOT NULL,
  p_size        INTEGER NOT NULL,
  p_container   VARCHAR(10) NOT NULL
);

CREATE TABLE supplier
(
  s_suppkey   INTEGER NOT NULL,
  s_name      VARCHAR(25) NOT NULL,
  s_address   VARCHAR(25) NOT NULL,
  s_city      VARCHAR(10) NOT NULL,
  s_nation    VARCHAR(15) NOT NULL,
  s_region    VARCHAR(12) NOT NULL,
  s_phone     VARCHAR(15) NOT NULL
);

CREATE TABLE customer
(
  c_custkey      INTEGER NOT NULL,
  c_name         VARCHAR(25) NOT NULL,
  c_address      VARCHAR(25) NOT NULL,
  c_city         VARCHAR(10) NOT NULL,
  c_nation       VARCHAR(15) NOT NULL,
  c_region       VARCHAR(12) NOT NULL,
  c_phone        VARCHAR(15) NOT NULL,
  c_mktsegment   VARCHAR(10) NOT NULL
);

CREATE TABLE dwdate
(
  d_datekey            INTEGER NOT NULL,
  d_date               VARCHAR(19) NOT NULL,
  d_dayofweek          VARCHAR(10) NOT NULL,
  d_month              VARCHAR(10) NOT NULL,
  d_year               INTEGER NOT NULL,
  d_yearmonthnum       INTEGER NOT NULL,
  d_yearmonth          VARCHAR(8) NOT NULL,
  d_daynuminweek       INTEGER NOT NULL,
  d_daynuminmonth      INTEGER NOT NULL,
  d_daynuminyear       INTEGER NOT NULL,
  d_monthnuminyear     INTEGER NOT NULL,
  d_weeknuminyear      INTEGER NOT NULL,
  d_sellingseason      VARCHAR(13) NOT NULL,
  d_lastdayinweekfl    VARCHAR(1) NOT NULL,
  d_lastdayinmonthfl   VARCHAR(1) NOT NULL,
  d_holidayfl          VARCHAR(1) NOT NULL,
  d_weekdayfl          VARCHAR(1) NOT NULL
);
CREATE TABLE lineorder
(
  lo_orderkey          INTEGER NOT NULL,
  lo_linenumber        INTEGER NOT NULL,
  lo_custkey           INTEGER NOT NULL,
  lo_partkey           INTEGER NOT NULL,
  lo_suppkey           INTEGER NOT NULL,
  lo_orderdate         INTEGER NOT NULL,
  lo_orderpriority     VARCHAR(15) NOT NULL,
  lo_shippriority      VARCHAR(1) NOT NULL,
  lo_quantity          INTEGER NOT NULL,
  lo_extendedprice     INTEGER NOT NULL,
  lo_ordertotalprice   INTEGER NOT NULL,
  lo_discount          INTEGER NOT NULL,
  lo_revenue           INTEGER NOT NULL,
  lo_supplycost        INTEGER NOT NULL,
  lo_tax               INTEGER NOT NULL,
  lo_commitdate        INTEGER NOT NULL,
  lo_shipmode          VARCHAR(10) NOT NULL
);
```

Once we've created our schema we need to load some data into it. Simply replace the access key id and secret placeholders with your personal credentials and run the COPY commands one at a time. This should take 10-15 minutes total to complete.

Note the time it takes to load the items, we'll use these values later. Mine were:

```sql
INFO:  Load into table 'customer' completed, 3000000 record(s) loaded successfully.
COPY
Time: 20069.243 ms

INFO:  Load into table 'dwdate' completed, 2556 record(s) loaded successfully.
COPY
Time: 7786.561 ms

INFO:  Load into table 'lineorder' completed, 600037902 record(s) loaded successfully.
COPY
Time: 664644.743 ms

INFO:  Load into table 'part' completed, 1400000 record(s) loaded successfully.
COPY
Time: 19120.994 ms

INFO:  Load into table 'supplier' completed, 1000000 record(s) loaded successfully.
COPY
Time: 12411.616 ms
```

Finally lets double check our row sizes to make sure they match what we expectselect count(*) from LINEORDER;

```sql
warehouse=# select count(*) from LINEORDER;
   count
-----------
 600037902
(1 row)

warehouse=# select count(*) from PART;
  count
---------
 1400000
(1 row)

warehouse=# select count(*) from  CUSTOMER;
  count
---------
 3000000
(1 row)

warehouse=# select count(*) from  SUPPLIER;
  count
---------
 1000000
(1 row)

warehouse=# select count(*) from  DWDATE;
 count
-------
  2556
(1 row)
```
