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

## Seed Test Fixture Data with Explicit `id`s to Generated Identity Fields

When adding data to a database for testing purposes, it's often useful to have explicit `id` values to reference records in other tables via foreign keys. However, these explicit `id` values are incompatible with identity fields such as a field specified with `id PRIMARY KEY GENERATED ALWAYS AS IDENTITY`.

To use explicit `id` values in test fixture data while using generated identity fields, drop the identity, insert the records and add the identity back.

The following example of this approach uses:

- [Postgres.js](https://github.com/porsager/postgres)

`seedFixtures.ts`

```ts
import { readdir } from 'node:fs/promises';
import postgres from 'postgres';

if (!process.env.FEATURE_TEST_SEEDING) {
  throw new Error('Set the environment variable FEATURE_TEST_SEEDING to seed database with test data');
}

const sql = postgres({
  transform: postgres.camel,
});

const testFixtures = (await readdir('./tables', { withFileTypes: true }))
  .filter((entry) => {
    return entry.isFile() && /^\d+-[^.]+\.fixture\.ts$/.test(entry.name);
  })
  .sort((a, b) => {
    return parseInt(a.name.split('-')[0]!) - parseInt(b.name.split('-')[0]!);
  });

for (const testFixture of testFixtures) {
  const testFixtureModule = (await import(`../tables/${testFixture.name}`)) as {
    [key: string]: { [key: string]: { id: number } };
  };

  for (const [exportName, fixturesObj] of Object.entries(testFixtureModule)) {
    const tableName = camelToSnake(
      exportName
        .replace(/^test/, '')
        .replace(/^[A-Z]/, (letter) => letter.toLowerCase()),
    ) as string;
    const fixtures = Object.values(fixturesObj);

    if (fixtures.length > 0) {
      const idFieldIsIdentity =
        (
          await sql<{ isIdentity: 'YES' | 'NO' }[]>`
            SELECT
              is_identity
            FROM
              information_schema.columns
            WHERE
              table_name = ${tableName}
              AND column_name = 'id'
          `
        )[0]!.isIdentity === 'YES';

      if (idFieldIsIdentity) {
        await sql`
          ALTER TABLE ${sql(tableName)}
          ALTER COLUMN id
          DROP IDENTITY
        `;
      }

      await sql`
        INSERT INTO
          ${sql(tableName)} ${sql(fixtures)}
      `;

      if (idFieldIsIdentity) {
        // Only configure GENERATED ALWAYS, to avoid inconsistencies with GENERATED BY DEFAULT
        await sql`
          ALTER TABLE ${sql(tableName)}
          ALTER COLUMN id
          ADD GENERATED ALWAYS AS IDENTITY
        `;

        // Reset sequence to the next record id, to allow for new
        // record inserts with the sequentially generated index
        // (PRIMARY KEY GENERATED ALWAYS AS IDENTITY)
        await sql`
          SELECT
            setval(
              pg_get_serial_sequence(
                ${tableName},
                'id'
              ),
              (
                SELECT
                  max(id)
                FROM
                  ${sql(tableName)}
              )
            )
        `;
      }

      console.log(`✔️ Inserted ${fixtures.length} records into ${tableName}`);
    }
  }
}

await sql.end();

console.log('Done syncing test seeding fixtures to database');

type CamelToSnake<
  S extends string,
  Result extends string = '',
> = S extends `${infer First}${infer Rest}`
  ? First extends Capitalize<First>
    ? CamelToSnake<Rest, `${Result}_${Lowercase<First>}`>
    : CamelToSnake<Rest, `${Result}${First}`>
  : Result;

function camelToSnake<CamelCaseString extends string>(
  camelCaseString: CamelCaseString,
): CamelToSnake<CamelCaseString> {
  return camelCaseString.replace(
    /[A-Z]/g,
    (letter) => `_${letter.toLowerCase()}`,
  ) as CamelToSnake<CamelCaseString>;
}
```

Fixture files look like this:

`tables/001-regions.fixture.ts`

```ts
type Region = {
  id: number;
  slug: string;
  title: string;
};

export const testRegions = {
  europe: {
    id: 1,
    slug: 'eu',
    title: 'Europe',
  },
  australia: {
    id: 2,
    slug: 'au',
    title: 'Australia',
  },
} as const satisfies { [key: string]: Region };
```

This allows the `id` values to be used in other tables as foreign keys, eg. `testRegions.europe.id` could be imported in `tables/002-campuses.fixture.ts` to use as a value for a foreign key field `campuses.region_id`.
