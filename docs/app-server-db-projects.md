# app-server/src/db/projects.rs

## Overview

This file is responsible for all database operations related to "projects" within the application. It provides functions for creating new projects, retrieving existing projects by their ID, and deleting projects. A key feature of project creation is an integrated authorization check to ensure that the user attempting to create a project is a member of the specified workspace.

## Key Components

- `struct Project`: Represents a project entity in the database.
    - `id: Uuid`: The unique identifier for the project.
    - `name: String`: The name of the project.
    - `workspace_id: Uuid`: The unique identifier of the workspace to which this project belongs.
    It derives `Deserialize`, `Serialize` (for JSON conversion), `FromRow` (for `sqlx` row mapping), and `Clone`.

- `async fn get_project(pool: &PgPool, project_id: &Uuid) -> Result<Project>`:
    Retrieves a single project from the database based on its unique `project_id`. Returns a `Result` containing the `Project` if found, or an error.

- `async fn create_project(pool: &PgPool, user_id: &Uuid, name: &str, workspace_id: Uuid) -> Result<Project>`:
    Creates a new project in the database.
    - Parameters:
        - `pool`: A reference to the `sqlx::PgPool` for database access.
        - `user_id`: The ID of the user attempting to create the project. This is used for an authorization check.
        - `name`: The desired name for the new project.
        - `workspace_id`: The ID of the workspace where the project will be created.
    - Logic: It attempts to insert a new project only if the provided `user_id` is found as a member of the given `workspace_id` in the `members_of_workspaces` table. If successful, it returns the newly created `Project`.

- `async fn delete_project(pool: &PgPool, project_id: &Uuid) -> Result<()>`:
    Deletes a project from the database based on its `project_id`. Returns a `Result` indicating success or failure.

## Important Variables/Constants

Not applicable for this file, as configurations are primarily passed as function arguments or are implicit in SQL queries. The fields of the `Project` struct are the main data points.

## Usage Examples

Conceptual examples of how these functions might be used by other parts of the application (e.g., API handlers):

```rust
// Assuming 'db_pool' is an available &PgPool and 'user_id', 'project_id', 'workspace_id' are Uuids.

// Example: Creating a new project
// let new_project_name = "My Awesome Project";
// let target_workspace_id = Uuid::new_v4();
// let current_user_id = Uuid::new_v4();
//
// match db::projects::create_project(&db_pool, &current_user_id, new_project_name, target_workspace_id).await {
//     Ok(project) => {
//         // Use the created project
//         println!("Created project: {}", project.name);
//     }
//     Err(e) => {
//         // Handle error, e.g., user not member of workspace or other DB error
//         eprintln!("Failed to create project: {}", e);
//     }
// }

// Example: Retrieving a project
// let project_to_find = Uuid::new_v4();
//
// match db::projects::get_project(&db_pool, &project_to_find).await {
//     Ok(project) => {
//         println!("Found project: {}", project.name);
//     }
//     Err(e) => {
//         eprintln!("Failed to get project: {}", e);
//     }
// }

// Example: Deleting a project
// let project_to_delete = Uuid::new_v4();
//
// match db::projects::delete_project(&db_pool, &project_to_delete).await {
//     Ok(()) => {
//         println!("Project deleted successfully.");
//     }
//     Err(e) => {
//         eprintln!("Failed to delete project: {}", e);
//     }
// }
```

## Dependencies and Interactions

- **`sqlx`**: Used extensively for all asynchronous PostgreSQL database interactions.
    - `PgPool`: For connection pooling.
    - `FromRow`: Trait used to map query results directly to the `Project` struct.
    - `query_as!`, `query!`: Macros/functions for executing typed SQL queries.
- **`serde`**: Used for `Serialize` and `Deserialize` traits on the `Project` struct, allowing it to be easily converted to/from formats like JSON, typically for API responses/requests.
- **`uuid::Uuid`**: Used for project and workspace identifiers.
- **`anyhow::Result`**: Standardized error handling mechanism for function return types.
- **Database Schema**: Interacts with at least two tables:
    - `projects`: The primary table for storing project data (id, name, workspace_id).
    - `members_of_workspaces`: Used by `create_project` to verify that the user creating the project is a member of the target workspace.
- **API Handlers**: These database functions are intended to be called by higher-level logic, most likely API route handlers within the `app-server` (e.g., in `app-server/src/routes/` or `app-server/src/api/`) that manage project-related HTTP requests.
```
