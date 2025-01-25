---
layout: post
title: "Solutions for Handling or Importing Data Without Using the 'COPY' Command"
date: 2025-01-25 02:00:01 +0100
categories: sql parameter limit error postgressql postgres jsonb_to_recordset insert batch
---

# Potential causes/examples for the error

Here are some of the most common scenarios that can result in the following errors:

```sql
bind message has 65536 parameter formats but 0 parameters
```

or

```sql
bind message supplies 65536 parameters, but the prepared statement "" requires 0
```

## Example 1: Bulk data imports / inserts

```sql
INSERT INTO "public"."product" (unique_id, name, quantity, price)
VALUES
    ($1, $2, $3, $4),   -- Item 1
    ($5, $6, $7, $8),   -- Item 2
    ($9, $10, $11, $12), -- Item 3
    -- ... (thousands of rows)
    ($100000, $100001, $100002, $100003) -- Item N
ON CONFLICT (unique_id) DO NOTHING;
```

In this scenario, attempting to insert too many items at once may exceed the parameter limit, resulting in an error. This becomes particularly problematic when handling conflicts (e.g., using ON CONFLICT DO NOTHING), as it requires specifying each parameter individually.

## Example 2: Queries with complex conditions

```sql
SELECT * FROM public.product
WHERE id IN ($1, $2, $3, ..., $100000)
AND price BETWEEN $100001 AND $100002
AND category = $100003;
```

In this case, specifying a large number of IDs in the IN clause may cause the query to exceed the parameter limit, resulting in errors. This situation might arise when you need to validate a large set of IDs before further processing.

# Why PostgreSQL's 'COPY' is not the solution here

The COPY command in PostgreSQL is efficient for bulk data operations but comes with significant limitations. Specifically, it restricts the ability to include additional conflict checks or PostgreSQL clauses directly within the insert operation.

Similarly, in the case of complex queries, such as those involving large IN clauses or multiple conditions, the COPY command alone is insufficient.

# Solution by using 'jsonb_to_recordset'

By using this function, you can pass the entire data batch as a single parameter, avoiding SQL injection risks associated with pasting raw, unparameterized data directly into the query.

Postgresql.org reference [PostgreSQL: Documentation: 9.5: JSON Functions and Operators](https://www.postgresql.org/docs/9.5/functions-json.html)

Official Description of 'jsonb_to_recordset': `Builds an arbitrary set of records from a JSON array of objects. As with all functions returning record, the caller must explicitly define the structure of the record with an AS clause.`

## How to use 'jsonb_to_recordset'

JSON input

```json
[
  { "a": 1, "b": "foo" },
  { "a": "2", "c": "bar" }
]
```

Constructed query

```sql
select * from
jsonb_to_recordset('[
  { "a": 1, "b": "foo" },
  { "a": "2", "c": "bar" }
]')
as x(a int, b text);
```

Result

```
 a |  b
---+-----
 1 | foo
 2 |
```

# Implementing the solution on examples above

## Example 1

JSON input

```json
[
  { "unique_id": "uuid1", "name": "item1", "quantity": 10, "price": 100.5 },
  { "unique_id": "uuid2", "name": "item2", "quantity": 5, "price": 50.0 }
]
```

Constructed query

```sql
WITH data as (
  SELECT * FROM jsonb_to_recordset($1::jsonb)
  AS x(unique_id uuid, name text, quantity int, price numeric)
)
INSERT INTO "public"."product" (unique_id, name, quantity, price)
SELECT unique_id, name, quantity, price
FROM data
ON CONFLICT (unique_id) DO NOTHING;
```

## Example 2

JSON input

```json
[
  {
    "id": "uuid1",
    "min_price": 50,
    "max_price": 100,
    "category": "electronics"
  },
  { "id": "uuid2", "min_price": 10, "max_price": 20, "category": "books" }
]
```

Constructed query

```sql
WITH filter_conditions as (
  SELECT * FROM jsonb_to_recordset($1::jsonb)
  AS x(id uuid, min_price numeric, max_price numeric, category text)
)
SELECT *
FROM public.product p
JOIN filter_conditions fc
  ON p.id = fc.id
WHERE p.price BETWEEN fc.min_price AND fc.max_price
AND p.category = fc.category;

```

# Key Takeaways

Although I have not tested it, I am confident that PostgreSQL's 'COPY' command will outperform this solution if it meets your use case requirements..

Furthermore, if you supply a very large amount of data using the `jsonb_to_recordset` function, you may encounter connection timeouts. This is because the query may take too long to compare, process, or insert the data, depending on the implementation.

If you start encountering connection timeouts, it may be time to reconsider whether this is the best approach or if there is significant room for improvement in the database structure itself (e.g., tables, indexes).

If time is limited or optimizing the specified aspects seems impossible, you can always resort to the final option: batching.

# Resources:

- [PostgreSQL: Documentation: 9.5: JSON Functions and Operators](https://www.postgresql.org/docs/9.5/functions-json.html)
- [How a few lines of code reduced database load by a few million queries
  ](https://gajus.medium.com/how-a-few-lines-of-code-reduced-database-load-by-a-few-million-queries-964d43ec668a)
- [Postgres jsonb_to_record() function](https://neon.tech/docs/functions/jsonb_to_record)
- [jsonb_to_recordset() and json_to_recordset()](https://docs.yugabyte.com/preview/api/ysql/datatypes/type_json/functions-operators/jsonb-to-recordset/)
- [PostgreSQL json_to_recordset() Function](https://www.commandprompt.com/education/postgresql-json_to_recordset-function/)
