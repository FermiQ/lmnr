# frontend/lib/sql/transpile.ts

## Overview

This TypeScript module provides a robust system for validating and transpiling user-submitted SQL queries. The primary goal is to ensure that these queries are safe and operate within predefined constraints before execution. Key functionalities include:

1.  Restricting queries to `SELECT` statements only.
2.  Enforcing access to a whitelist of allowed tables (`ALLOWED_TABLES`).
3.  Automatically injecting a `project_id` filter into `WHERE` clauses to enforce data isolation and security.
4.  Providing syntactic sugar for querying JSONB fields, abstracting the underlying SQL complexities.
5.  Applying a default query limit (e.g., `LIMIT 100`) if not specified by the user, to prevent performance issues.
6.  Adding predefined Common Table Expressions (CTEs) to enhance query capabilities.

The module also includes a function `executeSafeQuery` to run these validated and transpiled queries against a PostgreSQL database using Drizzle ORM.

## Key Components

-   `class SQLValidator`:
    The core class orchestrating the parsing, validation, and transpilation process.
    -   `constructor(allowedTables: Set<string> = ALLOWED_TABLES)`: Initializes the validator. It uses `node-sql-parser` for AST manipulation and takes an optional set of `allowedTables`.
    -   `validateAndTranspile(sqlQuery: string, projectId: string): TranspiledQuery`: The main public method. It takes a raw SQL query string and the current user's `projectId`. It parses the query, validates it against security and structural rules, modifies the Abstract Syntax Tree (AST) to enforce constraints and add features, and then converts the AST back to a SQL string. The result includes the SQL, arguments, validity status, and any errors or warnings.
    -   `private isSelectQuery(ast: AST | AST[]): boolean`: Checks if the parsed AST represents only `SELECT` queries.
    -   `private transpileQuery(ast: AST | AST[], projectId: string): { sql: string; args: Arg[]; warnings?: string[] }`: Modifies the input AST. It incorporates additional CTEs, applies a default limit, and calls `processSubqueries` for in-depth AST manipulation.
    -   `private processSubqueries(node: AST, processedNodes: Set<AST>): AST`: A recursive function that traverses the query AST. For each `SELECT` node (including subqueries and CTEs), it performs several transformations:
        -   Applies automatic join rules (via `applyAutoJoinRules`).
        -   Qualifies column names with table aliases or names (via `qualifyColumnReferences`).
        -   Rewrites JSONB field access to standard SQL (via `replaceJsonbFields`).
        -   Recursively processes nested structures like CTEs, `FROM` clause subqueries, and expressions in `WHERE` clauses.
        -   Injects the `project_id` filter into the `WHERE` clause of the current `SELECT` statement (via `applyProjectIdToStatement`), targeting its main table.

-   `async function executeSafeQuery(sqlQuery: string, projectId: string, logger: Logger = new NoopLogger()): Promise<{ result: Record<string, any>[]; warnings?: string[] }>`:
    A utility function that creates an `SQLValidator` instance, calls `validateAndTranspile` on the input query and `projectId`. If the transpilation is successful, it executes the generated SQL using a Drizzle ORM `PostgresJsPreparedQuery` with the provided `projectId` as a parameter. It returns the query results and any warnings.

-   `interface TranspiledQuery` (imported from `./types`):
    Defines the structure of the return value from `validateAndTranspile`. It typically includes:
    -   `valid: boolean`: Whether the transpilation was successful.
    -   `sql: string | null`: The transpiled SQL string, or null if invalid.
    -   `args: Arg[]`: An array of arguments for the prepared statement (e.g., `project_id`).
    -   `error: string | null`: An error message if validation or transpilation failed.
    -   `warnings?: string[]`: Any warnings generated during transpilation (e.g., default limit applied).

-   `ALLOWED_TABLES: Set<string>` (imported from `./types`):
    A predefined set of table names that users are permitted to query.

-   `ADDITIONAL_WITH_CTES: any[]` (imported from `./with`):
    An array of AST structures representing CTEs that are automatically prepended to user queries.

## Important Variables/Constants

-   `ALLOWED_TABLES`: As described above, critical for security by restricting table access.
-   `ADDITIONAL_WITH_CTES`: Extends query capabilities by injecting common table expressions.
-   Default query limit: A hardcoded limit (e.g., 100) is applied if the user doesn't specify one, primarily for performance control.

## Usage Examples

```typescript
// Conceptual example of using SQLValidator and executeSafeQuery

import { SQLValidator, executeSafeQuery } from './transpile'; // Assuming correct path
import { db } from '@/lib/db/drizzle'; // For executeSafeQuery
import { Logger } from 'drizzle-orm';

const userQuery = "SELECT id, name, details->>'fieldName' AS field FROM events WHERE status = 'active' ORDER BY created_at DESC";
const currentProjectId = "project_abc123";

// Using SQLValidator directly:
const validator = new SQLValidator();
const transpiledOutput = validator.validateAndTranspile(userQuery, currentProjectId);

if (transpiledOutput.valid && transpiledOutput.sql) {
  console.log("Transpiled SQL:", transpiledOutput.sql);
  console.log("Arguments:", transpiledOutput.args);
  if (transpiledOutput.warnings) {
    console.warn("Warnings:", transpiledOutput.warnings.join('\n'));
  }
  // transpiledOutput.sql could then be passed to a database execution function
  // that handles prepared statements with named arguments or positional parameters.
} else {
  console.error("SQL Error:", transpiledOutput.error);
}

// Using executeSafeQuery:
async function runUserQuery() {
  try {
    // Assuming 'db' is your Drizzle client instance and 'logger' is a Drizzle logger
    const { result, warnings } = await executeSafeQuery(userQuery, currentProjectId, /* logger */);
    console.log("Query Results:", result);
    if (warnings) {
      console.warn("Execution Warnings:", warnings.join('\n'));
    }
  } catch (error) {
    console.error("Failed to execute query:", error.message);
  }
}

// runUserQuery();
```

## Dependencies and Interactions

-   **`node-sql-parser`**: The core external library used for parsing SQL strings into an Abstract Syntax Tree (AST) and for converting the modified AST back into a SQL string.
-   **`drizzle-orm` / `drizzle-orm/postgres-js`**: Used by `executeSafeQuery` to create and execute prepared SQL statements against a PostgreSQL database.
-   **`@/lib/db/drizzle`**: Provides the application's configured Drizzle database client (`db`) used by `executeSafeQuery`.
-   **Internal Helper Modules (within `frontend/lib/sql/`)**:
    -   `./types`: Provides shared type definitions like `TranspiledQuery`, `Arg`, and the `ALLOWED_TABLES` constant.
    -   `./expression` (`getExpressionASTs`): Utilities for working with expression nodes in the AST.
    -   `./join` (`applyAutoJoinRules`, `qualifyColumnReferences`): Functions to modify the AST by adding necessary joins and ensuring column references are unambiguous.
    -   `./project-id-filter` (`applyProjectIdToStatement`, `findMainTable`): Logic to find the appropriate place in the AST to inject the `project_id` filter and perform the injection.
    -   `./replace` (`replaceJsonbFields`): Handles the transformation of custom JSONB field access syntax into standard SQL.
    -   `./utils` (`getFromTableNames`): General utility functions for inspecting or manipulating the AST.
    -   `./with` (`ADDITIONAL_WITH_CTES`): Supplies predefined CTE definitions to be merged into the user's query AST.
-   **Consumers**: This module is likely consumed by backend API endpoints or services within the frontend's server-side logic that receive user-defined queries (e.g., from a custom query builder UI or a direct SQL input field). It acts as a security and transformation layer before database execution.
```
