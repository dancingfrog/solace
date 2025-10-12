Discussion
==========

I recently worked through setting up this fullstack Next.js application with PostgreSQL (running in Docker), and what follows is a reconstruction of that diagnostic journey.

## Database setup

The project came with `docker-compoose.yml` file that provisions a single container to host a PostgreSQL server, along with a persistent volume. I appended an additional command to `docker compose up` that allowed me `psql` directly into this databsae from the host terminal: 
```bash
docker exec -it $(docker ps --filter "ancestor=postgres" --format "{{.Names}}") psql -U postgres
```

I will walk through what happened next, exploring the state of the database at each step.

### Query Zero: Fresh Start

```sql
You are now connected to database "solaceassignment" as user "postgres".
solaceassignment=# \dt
Did not find any tables.
solaceassignment=# \dn
      List of schemas
  Name  |       Owner       
--------+-------------------
 public | pg_database_owner
(1 row)
```

After running `docker compose up -d`, the PostgreSQL container started and the `solaceassignment` database was automatically created thanks to the `POSTGRES_DB` environment variable in the docker-compose configuration.

This is the ground zero state. The database exists as a named container, but it doesn't know anything about advocates or migrations or application structure yet. Everything that makes this database useful for the application still needs to be built.

#### Schema Inspection

```sql
solaceassignment=# \dn
       List of schemas
  Name   |       Owner       
---------+-------------------
 drizzle | postgres
 public  | pg_database_owner
(2 rows)
```

This query lists two schemas: `public` (where the advocates table lives) and `drizzle` (which tracks migrations). The presence of the `drizzle` schema confirms that migrations were applied through the proper workflow rather than manual table creation.

### Query One: The Empty Table

```sql
SELECT COUNT(*) FROM advocates;
-- Result: 0
```

This query returns zero instead of an error, which means the `advocates` table now exists. Between Query Zero and this query, two things happened:

First, `npx drizzle-kit generate` read the TypeScript schema and created migration file `0000_elite_thor.sql`. Then `npm run drizzle:migrate:up` applied those migrations, creating the `advocates` table and a `drizzle` schema for tracking which migrations have run.

The database now has structure but no data.

### The Gap: When Plans Don't Survive Contact

When I tried running `npm run seed`, Node threw an error about not being able to find a module. Specifically, it was looking for `src/db/seed/index.ts`, which doesn't exist in the project. The package.json script is configured to run this file using esbuild-register to transpile TypeScript on the fly, but you can't transpile a file that isn't there.

This is one of those interesting artifacts you find in software projects. The script definition suggests someone planned to implement a direct database seeding approach where a standalone Node script would connect to PostgreSQL and insert records independently of the Next.js application. It's a reasonable pattern. Direct seeding scripts don't require the whole application to be running, they execute faster, and they're easier to integrate into CI/CD pipelines.

But somewhere along the line, the project pivoted to a different approach. Maybe the direct script seemed like overkill for a small dataset. Maybe the API-based approach was simpler to implement. Maybe someone just forgot to finish it. Whatever the reason, the script definition stayed in package.json even though the actual implementation never materialized.

The file structure in `src/db/seed/` contains only `advocates.ts`, which exports the seed data itself but doesn't include any logic to actually insert it into the database. The package.json script is trying to run an `index.ts` that would have contained that insertion logic, but it's not there.

This is actually a fairly common situation in full-stack development: multiple ways to accomplish the same task, each with different tradeoffs. Direct database scripts don't require the application to be running and they bypass application-layer logic, which can be good for speed but bad for consistency. API endpoints integrate with the application's middleware and validation logic, which is good for maintainability but requires the server to be running.

The incomplete implementation suggests the project started with one approach and switched to another without cleaning up the old script definition. It happens. Code evolves.

### The Working Alternative

Since the direct seeding script didn't work, I used the API endpoint at `/api/seed`. After starting the dev server with `npm run dev`, I ran `curl -X POST http://localhost:3000/api/seed` to trigger the seeding.

### Query Two: Data Appears

```sql
SELECT COUNT(*) FROM advocates;
-- Result: 15
```

The seeding worked. Fifteen advocate records are now in the database.

### Query Three: No Idempotency Protection

```sql
SELECT COUNT(*) FROM advocates;
-- Result: 30
```

I made a second request to `http://localhost:3000/api/seed`, and the count doubled to 30. **The seeding endpoint has no duplicate protection, so running it multiple times creates duplicate records**. This _might be_ fine for development, but may also confuse testing and verification of certain feature's (i.e., search and user selection of a specific advocate). We would need additional safeguards for production in order to constrain the data to storing unique records (not just unique identifiers).

### Summary

The setup sequence: Docker startup → migrations → seeding via API endpoint. Query Zero established the baseline (no tables). Query One proved migrations ran (table exists, returns 0 instead of error). Query Two proved seeding worked (15 rows). Query Three revealed that the seeding endpoint lacks idempotency protection (30 rows after second request).

## Frontend Issues

After completing the database setup above, I was able to navigate to `http://localhost:3003` and could see that the application loads and displays data correctly, but there was a React hydration error in the console. 

### Hydration Error

> "Hydration failed because the initial UI does not match what was rendered on the server." 

The specific issue was in `src/app/page.tsx` where table headers are incorrectly structured. The `<th>` elements are direct children of `<thead>`, which violates HTML standards. Table headers must be wrapped in a `<tr>` element.

Current problematic code:
```jsx
<thead>
  <th>First Name</th>
  <th>Last Name</th>
  <!-- more headers -->
</thead>
```

The fix is simple - add a `<tr>` wrapper:
```jsx
<thead>
  <tr>
    <th>First Name</th>
    <th>Last Name</th>
    <!-- more headers -->
  </tr>
</thead>
```

This error occurred because the server renders valid HTML (automatically inserting the missing `<tr>`), but the client-side React code tries to match with the incorrect structure. Despite the error, the table displays correctly because browsers are forgiving with HTML structure.

In the process of reading and correcting this code in `src/app/page.tsx`, I noticed there are a couple of other issues that could be improved in the code:

1) The `<tr>` and `<div>` elements inside the mapping functions don't have key props, which React requires for efficient list rendering. Solution:
    - For table rows, add `key={advocate.id}` to each `<tr>`
    - For specialty divs, add `key={index}` to each specialty `<div>`
2) The filter function is directly checking if strings include a search term without any case-insensitivity consideration.

Additionally, testing revealed these existing issues with the current search functionality:

1) There's an error when trying to search for terms in the `yearsOfExperience` field because it's trying to use `.includes()` on a number, but `.includes()` is a string method, not a number method.
2) The search on the `specialties` field is also incorrect - it's treating `specialties` as a string when it's actually an array based on the UI.



### Solutions

All of the issues identified above were fixed:

1. **Table structure:** Added the missing `<tr>` wrapper around table headers in the `<thead>` section

2. **React key props:** 
   - Added `key={advocate.id}` to each `<tr>` element in the table rows
   - Added `key={index}` to each specialty `<div>` within the specialties cell

3. **Case-insensitive search:**
   - Modified the search function to convert both the search term and the data to lowercase before comparison
   - Search now works for any case combination (e.g., "PHD", "phd", or "PhD" will all match)

4. **Type handling for numeric fields:**
   - Fixed the error with `yearsOfExperience` by converting it to a string before using the `includes()` method
   - Now searching by numbers (e.g., "10") works correctly

5. **Array handling for specialties:**
   - Fixed the search for the `specialties` array by using `some()` to check if any specialty includes the search term
   - This allows searching for specialties like "PTSD" to work properly

The application now functions correctly with proper search capabilities, proper HTML structure, and no React warnings or errors.

I also added an OpenAPI specification and documented it with `redocly`.
