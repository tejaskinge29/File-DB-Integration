# File-DB-Integration

MuleSoft 4 application that automatically reads employee CSV files from a local directory, validates and transforms the data, and upserts (INSERT or UPDATE) the records into a MySQL database using batch processing.

## Project Overview

| Property        | Value                        |
|-----------------|------------------------------|
| **Project Name**| File DB-integration               |
| **Flow Name**   | file-to-db-flow              |
| **Batch Job**   | db-integrationBatch_Job      |
| **Runtime**     | Mule 4.x (Enterprise Edition)|
| **Database**    | MySQL                        |
| **Trigger**     | File Listener (Daily)        |

---

## Architecture & Flow

```
input/                      processing/              processed/  OR  error/
[CSV File] ──► File Listener ──► Transform ──► Batch Job ──► file:move
                                  (CSV→JSON)     │
                                                 ├── Salary Validation
                                                 ├── DB SELECT (check exists)
                                                 ├── UPDATE (if exists)
                                                 └── INSERT  (if new)
```

---

## Prerequisites

Before running this project, ensure you have the following installed and configured:

- **Anypoint Studio** 7.x or later
- **Mule Runtime** 4.x (EE)
- **Java** JDK 8 or 11
- **MySQL** 5.7+ running on `localhost:3306`
- **MySQL JDBC Driver** (`mysql-connector-java`) added to the project dependencies
- The database `testdb` created in MySQL
- The `employees` table created (see schema below)

---

## Project Structure

```
db-integration/
│
├── src/
│   └── main/
│       ├── mule/
│       │   └── db-integration.xml        ← Main flow definition
│       └── resources/
│           └── (config files if any)
│
├── C:\mule-demo\                          ← Working directory (local filesystem)
│   ├── input\                             ← Drop CSV files here
│   ├── processing\                        ← Files move here while being processed
│   ├── processed\                         ← Successfully processed files land here
│   └── error\                             ← Files that failed processing land here
│
├── employees1.csv                         ← Sample input file
└── README.md
```

> **Important:** Make sure all four subdirectories (`input`, `processing`, `processed`, `error`) exist under `C:\mule-demo\` before running the application.

---

## Key Implementation Notes

1. **Store `attributes.fileName` early** — After the Transform step, `attributes` is overwritten. Always capture it in a variable right after the file listener.

2. **Store the record before `db:select`** — `db:select` overwrites `payload` with the query result. Use `vars.currentRecord` to preserve the original employee data for use in UPDATE/INSERT.

3. **Date format for MySQL** — DataWeave's `Date` type cannot be passed directly to MySQL via JDBC named parameters. Always convert to a `String` with format `yyyy-MM-dd`.

4. **`file:move` path evaluation** — The `sourcePath` attribute does not evaluate inline DataWeave expressions like `"processing/#[vars.fileName]"`. Always pre-evaluate the path using `ee:transform` into a variable, then reference `#[vars.sourceFilePath]`.

5. **Batch is asynchronous** — Never place post-processing logic (like file moves) after `batch:job` in the main flow. Always use `batch:on-complete` for actions that should run after all records are processed.
