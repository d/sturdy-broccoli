## Two-Stage Aggregation
In Postgres, an aggregate function can be declared as:

```sql
CREATE TYPE int8_pair AS (sum int8, count int8);
CREATE FUNCTION int4_avg_accum(transvalue int8_pair, newval int4) RETURNS int8_pair AS $$
SELECT (transvalue.sum + newval, transvalue.count +1);
$$ LANGUAGE SQL STRICT IMMUTABLE;

CREATE FUNCTION int8_avg(transvalue int8_pair) RETURNS numeric AS $$
SELECT transvalue.sum / transvalue.count;
$$ LANGUAGE SQL STRICT IMMUTABLE;

CREATE AGGREGATE avg(int4)
(
    sfunc = int4_avg_accum,
    stype = int8_pair,
    finalfunc = int8_avg,
    initcond = '{0,0}'
)
```

To better facilitate parallel execution of aggregation, Greenplum introduced "preliminary function", which can combine two trans values and return a new trans value (aggregate state):

```sql
CREATE FUNCTION int4_avg_combine(state1 int8_pair, state2 int8_pair) RETURNS int8_pair AS $$
SELECT (state1.sum + state2.sum, state1.count + state2.count);
$$ LANGUAGE SQL STRICT IMMUTABLE;

CREATE AGGREGATE avg(int4)
(
    sfunc = int4_avg_accum,
    stype = int8_pair,
    finalfunc = int8_avg,
    prefunc = int4_avg_combine
    initcond = '{0,0}'
)
```
