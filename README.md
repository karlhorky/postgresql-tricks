# PostgreSQL Tricks

A collection of useful tricks for PostgreSQL

## Create Records in Result Set

### [DB Fiddle Demo](https://www.db-fiddle.com/f/jXbjFvW2iLg5XhFXibx1e6/0)

It can be useful to generate new hardcoded records which are not returned from data in your tables.

For example, it may be useful to have an arbitrary empty record at the end of this result set, filled with `NULL`s:

```sql
SELECT
  id,
  title
FROM
  competence_levels
ORDER BY id DESC NULLS LAST;

 id |        title
----+----------------------
  6 | Proficient
  5 | Skilled
  4 | Experienced
  3 | Familiar
  2 | Practical Experience
  1 | Basic Understanding
(6 rows)
```

To create this new empty row at the bottom of the result set, the `UNION ALL` operator can be used to add an additional row:

```sql
SELECT
  id,
  title
FROM
  (
    SELECT id, title FROM competence_levels
    UNION ALL
    SELECT NULL AS id, NULL AS title
  ) AS competence_levels_with_nulls
ORDER BY id DESC NULLS LAST;

 id |        title
----+----------------------
  6 | Proficient
  5 | Skilled
  4 | Experienced
  3 | Familiar
  2 | Practical Experience
  1 | Basic Understanding
    |
(7 rows)
```

To add multiple records, another option is to use the `VALUES` keyword:

```sql
SELECT
  id,
  title
FROM
  (
    SELECT id, title FROM competence_levels
    UNION ALL
    SELECT id, title FROM (VALUES
      (NULL, NULL),
      (1, NULL),
      (NULL, 'Basic Understanding')
     ) AS hardcoded_competence_levels(id, title)
  ) AS competence_levels_with_nulls
ORDER BY id DESC NULLS LAST;

 id |        title
----+----------------------
  6 | Proficient
  5 | Skilled
  4 | Experienced
  3 | Familiar
  2 | Practical Experience
  1 | Basic Understanding
  1 |
    | Basic Understanding
    |
(9 rows)
```
