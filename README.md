# uuid-db-performance

Testing performance when uuid used as primary with foreignkey reference

## Configuration
Below is the data setup to test the performance of having UUID as primary key column with one child table having this column as foreign key.
Two primary tables
* `trans_uuid` has three columns with `id` being the primary key of type uuid - auto-generated by the postgresql.
* `trans_numeric` has three columns with `id` being the primary key of type numeric (serial).

Two child tables
* `child_table1` has foreign key `trans_id` of `trans_uuid` table.
* `child_table2` has foreign key `trans_id` of `trans_numeric` table.

## Setup
A large set of random data are populated into both the tables.

```sql
# setup the indexes
create index idx_trans_uuid on child_table1(trans_id);
create index idx_trans_numeric on child_table2(trans_id);
```

Populate into the trans_uuid table.
```sql
# with id being auto-generated by the postgresql
insert into trans_uuid(name, amount) select left(md5(random()::text), 5), random()::numeric from generate_series(1,1000000) s(i);
```
And, into the trans_numeric table.
```sql
# rerun this query as many times as needed
insert into trans_numeric(name, value) select left(md5(random()::text), 5), random()::numeric from generate_series(1,10000000) s(i);
```

Child tables are populated similarly by additionally selecting a random row from the trans table.

```sql
insert into child_table1 (name, trans_id) select left(md5(random()::text), 5), id from trans offset floor(random()*1000000) limit 500000;
```
```sql
insert into child_table2 (msg, trans_id) select left(md5(random()::text), 5), id from trans_numeric offset floor(random()*1000000) limit 500000;
``` 

## Storage
The below is the data count populated

```sql
uuid_load_test=# select count(*) from trans_uuid;
 2000010

Time: 2037.248 ms (00:02.037)
```
```sql
uuid_load_test=# select count(*) from trans_numeric;
 2000010

Time: 2491.988 ms (00:02.492)
```

And, child tables

```sql
>  select count(*) from child_table1;
 1000104

Time: 528.310 ms
```

```sql
> select count(*) from child_table2;
 1000104

Time: 201.089 ms
```

## Report

### Select
Select a random uuid

``` sql
explain analyze select * from child_table1 where trans_id='e30ab279-3145-4782-b323-bc70ac74f23d';
 Index Scan using idx_trans_uuid on child_table1  (cost=0.42..8.44 rows=1 width=26) (actual time=3.643..3.643 rows=0 loops=1)
   Index Cond: (trans_id = 'e30ab279-3145-4782-b323-bc70ac74f23d'::uuid)
 Planning time: 0.499 ms
 Execution time: 3.716 ms

Time: 50.045 ms
```

Select a random number

```sql
explain analyze select * from trans_numeric where id=16087330;
 Index Scan using trans_numeric_pkey on trans_numeric  (cost=0.43..8.45 rows=1 width=14) (actual time=0.021..0.022 rows=1 loops=1)
   Index Cond: (id = 16087330)
 Planning time: 0.093 ms
 Execution time: 0.046 ms
 ```
 
### Inserts

Running straight inserts - the UUID are generated before hand - to verify insert without UUID generating
```sql
select gen_random_uuid();
 ee85cd8a-ea51-4f0f-aad6-eba718ba6e59
Time: 0.310 ms
insert into trans_uuid(id, name, value) values('ee85cd8a-ea51-4f0f-aad6-eba718ba6e59', 'new-record-3', '11.11');
INSERT 0 1
Time: 2.499 ms

# another one
uuid_load_test=# select gen_random_uuid();
 c3456240-b69a-483b-8c5f-a54fc4fdcf32

Time: 0.369 ms
uuid_load_test=# insert into trans_uuid(id, name, value) values('c3456240-b69a-483b-8c5f-a54fc4fdcf32', 'new2', '11.11');
INSERT 0 1
Time: 3.192 ms
```


Running multiple time averaged to min 2.5+ ms to max 5.1+ms
```sql
explain analyze insert into trans_uuid(name, value) select left(md5(random()::text), 5), random()::numeric limit 1;
 Insert on trans_uuid  (cost=0.00..0.04 rows=1 width=60) (actual time=2.490..2.490 rows=0 loops=1)
   ->  Subquery Scan on "*SELECT*"  (cost=0.00..0.04 rows=1 width=60) (actual time=0.033..0.036 rows=1 loops=1)
         ->  Limit  (cost=0.00..0.03 rows=1 width=64) (actual time=0.019..0.019 rows=1 loops=1)
               ->  Result  (cost=0.00..0.03 rows=1 width=64) (actual time=0.018..0.018 rows=1 loops=1)
 Planning time: 0.072 ms
 Execution time: 2.538 ms
```


Running multiple times averaged to min 0.105ms to max of 0.363ms
 
```sql
explain analyze insert into trans_numeric(name, value) select left(md5(random()::text), 5), random()::numeric limit 1;
 Insert on trans_numeric  (cost=0.00..0.05 rows=1 width=48) (actual time=0.103..0.103 rows=0 loops=1)
   ->  Subquery Scan on "*SELECT*"  (cost=0.00..0.05 rows=1 width=48) (actual time=0.069..0.071 rows=1 loops=1)
         ->  Limit  (cost=0.00..0.03 rows=1 width=64) (actual time=0.039..0.039 rows=1 loops=1)
               ->  Result  (cost=0.00..0.03 rows=1 width=64) (actual time=0.038..0.038 rows=1 loops=1)
 Planning time: 0.071 ms
 Execution time: 0.177 ms
 ```

#### Multiple Inserts

Test inserting 10 records in row

```sql
uuid_load_test=# explain analyze insert into trans_uuid(name, value) select left(md5(random()::text), 5), random()::numeric limit 10;
 Insert on trans_uuid  (cost=0.00..0.04 rows=1 width=60) (actual time=3.109..3.109 rows=0 loops=1)
   ->  Subquery Scan on "*SELECT*"  (cost=0.00..0.04 rows=1 width=60) (actual time=0.048..0.052 rows=1 loops=1)
         ->  Limit  (cost=0.00..0.03 rows=1 width=64) (actual time=0.028..0.030 rows=1 loops=1)
               ->  Result  (cost=0.00..0.03 rows=1 width=64) (actual time=0.027..0.028 rows=1 loops=1)
 Planning time: 0.101 ms
 Execution time: 3.175 ms

Time: 4.005 ms
uuid_load_test=# explain analyze insert into trans_numeric(name, value) select left(md5(random()::text), 5), random()::numeric limit 10;
 Insert on trans_numeric  (cost=0.00..0.05 rows=1 width=48) (actual time=0.084..0.084 rows=0 loops=1)
   ->  Subquery Scan on "*SELECT*"  (cost=0.00..0.05 rows=1 width=48) (actual time=0.040..0.043 rows=1 loops=1)
         ->  Limit  (cost=0.00..0.03 rows=1 width=64) (actual time=0.024..0.025 rows=1 loops=1)
               ->  Result  (cost=0.00..0.03 rows=1 width=64) (actual time=0.023..0.023 rows=1 loops=1)
 Planning time: 0.104 ms
 Execution time: 0.151 ms

Time: 5.406 ms
```

## Delete
Delete some random data

Delete first in the child tables
```sql
uuid_load_test=# delete from child_table1 where trans_id ='83a30f80-b011-47e9-9cf9-afbc7e287a16';
DELETE 0
Time: 2.741 ms
uuid_load_test=# delete from child_table1 where trans_id ='f50acdee-8540-4c90-b9e6-8b491d5618ca';
DELETE 0
Time: 3.022 ms
uuid_load_test=# delete from child_table1 where trans_id ='ff1c2e36-e9ef-417f-ac99-34f891925d4f';
DELETE 0
Time: 2.557 ms
```
Now, delete in trans_uuid table
```sql
uuid_load_test=# delete from trans_uuid where id='83a30f80-b011-47e9-9cf9-afbc7e287a16';
DELETE 1
Time: 7.328 ms
uuid_load_test=# delete from trans_uuid where id='f50acdee-8540-4c90-b9e6-8b491d5618ca';
DELETE 1
Time: 6.526 ms
uuid_load_test=# delete from trans_uuid where id='ff1c2e36-e9ef-417f-ac99-34f891925d4f';
DELETE 1
Time: 7.238 ms
```

Now in the trans_numeric table
```sql
uuid_load_test=# delete from child_table2 where trans_id =15819239;
DELETE 1
Time: 4.372 ms
uuid_load_test=# delete from child_table2 where trans_id =15819240;
DELETE 1
Time: 2.562 ms
uuid_load_test=# delete from child_table2 where trans_id =15819241;
DELETE 1
Time: 2.768 ms
```

```sql
uuid_load_test=# delete from trans_numeric where id=15819239;
DELETE 1
Time: 4.141 ms
uuid_load_test=# delete from trans_numeric where id=15819240;
DELETE 1
Time: 2.789 ms
uuid_load_test=# delete from trans_numeric where id=15819241;
DELETE 1
Time: 2.742 ms
```
## Conclusion
There seems to be some inconsistent performance different during inserts in case UUID is used as primary key, possible reasons could be the non sequential ordering in the primary key because of random UUID.

## TODO
- [ ] Verify the performance of join queries
- [ ] Analyze the internals to understand what is happening down there.
