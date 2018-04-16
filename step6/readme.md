## Recreating the Test Data Set

Now that we fully understand sort keys, distribution keys and column encodings we can update our schema to reflect the design choices we've made.

### Drop the Test Data Set

```sql
drop table part cascade;
drop table supplier cascade;
drop table customer cascade;
drop table dwdate cascade;
drop table lineorder cascade;
```

### Recreate The Tables

Now we will re-create the tables with sort keys and dist keys applied

```
CREATE TABLE part (
  p_partkey     	integer     	not null sortkey distkey,
  p_name        	varchar(22) 	not null,
  p_mfgr        	varchar(6)      not null,
  p_category    	varchar(7)      not null,
  p_brand1      	varchar(9)      not null,
  p_color       	varchar(11) 	not null,
  p_type        	varchar(25) 	not null,
  p_size        	integer     	not null,
  p_container   	varchar(10)     not null
);

CREATE TABLE supplier (
  s_suppkey     	integer        not null sortkey,
  s_name        	varchar(25)    not null,
  s_address     	varchar(25)    not null,
  s_city        	varchar(10)    not null,
  s_nation      	varchar(15)    not null,
  s_region      	varchar(12)    not null,
  s_phone       	varchar(15)    not null)
diststyle all;

CREATE TABLE customer (
  c_custkey     	integer        not null sortkey,
  c_name        	varchar(25)    not null,
  c_address     	varchar(25)    not null,
  c_city        	varchar(10)    not null,
  c_nation      	varchar(15)    not null,
  c_region      	varchar(12)    not null,
  c_phone       	varchar(15)    not null,
  c_mktsegment      varchar(10)    not null)
diststyle all;

CREATE TABLE dwdate (
  d_datekey            integer       not null sortkey,
  d_date               varchar(19)   not null,
  d_dayofweek	      varchar(10)   not null,
  d_month      	    varchar(10)   not null,
  d_year               integer       not null,
  d_yearmonthnum       integer  	 not null,
  d_yearmonth          varchar(8)	not null,
  d_daynuminweek       integer       not null,
  d_daynuminmonth      integer       not null,
  d_daynuminyear       integer       not null,
  d_monthnuminyear     integer       not null,
  d_weeknuminyear      integer       not null,
  d_sellingseason      varchar(13)    not null,
  d_lastdayinweekfl    varchar(1)    not null,
  d_lastdayinmonthfl   varchar(1)    not null,
  d_holidayfl          varchar(1)    not null,
  d_weekdayfl          varchar(1)    not null)
diststyle all;

CREATE TABLE lineorder (
  lo_orderkey      	    integer     	not null,
  lo_linenumber        	integer     	not null,
  lo_custkey           	integer     	not null,
  lo_partkey           	integer     	not null distkey,
  lo_suppkey           	integer     	not null,
  lo_orderdate         	integer     	not null sortkey,
  lo_orderpriority     	varchar(15)     not null,
  lo_shippriority      	varchar(1)      not null,
  lo_quantity          	integer     	not null,
  lo_extendedprice     	integer     	not null,
  lo_ordertotalprice   	integer     	not null,
  lo_discount          	integer     	not null,
  lo_revenue           	integer     	not null,
  lo_supplycost        	integer     	not null,
  lo_tax               	integer     	not null,
  lo_commitdate         integer         not null,
  lo_shipmode          	varchar(10)     not null
);
```

Now we'll re-use the script for populating the tables with one key difference, we'll allow the COPY statement to apply compression encodings!

```
INFO:  Load into table 'customer' completed, 3000000 record(s) loaded successfully.
COPY
Time: 77655.380 ms

INFO:  Load into table 'dwdate' completed, 2556 record(s) loaded successfully.
COPY
Time: 48028.143 ms

INFO:  Load into table 'lineorder' completed, 600037902 record(s) loaded successfully.
COPY
Time: 815671.227 ms

INFO:  Load into table 'part' completed, 1400000 record(s) loaded successfully.
COPY
Time: 45533.422 ms

INFO:  Load into table 'supplier' completed, 1000000 record(s) loaded successfully.
COPY
Time: 66378.212 ms
```

These should take marginally longer to load than the first time we did it. But we'll see that time back once we start taking a look at storage space and execution time.
