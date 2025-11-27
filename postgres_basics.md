# PostgreSQL Cheatsheet for Linux System Admins

### Connecting to PostgreSQL

```bash
# Connect as default user
psql

# Connect to a specific database as a user
psql -U username -d dbname

# Connect specifying host and port
psql -h host -p port -U username -d dbname
```

---

### Navigating Databases

```sql
-- List all databases
\l

-- Connect to a database
\c dbname

-- Show current database
\conninfo
```

---

### Navigating Tables and Schemas

```sql
-- List all tables in current database
\dt

-- List tables in a specific schema
\dt schema_name.*

-- List all schemas
\dn

-- Show table structure (columns, types)
\d table_name

-- List indexes for a table
\di table_name

-- List all users/roles
\du
```

---

### Managing Tables

```sql
-- Create a table
CREATE TABLE table_name (
    id SERIAL PRIMARY KEY,
    column1 TEXT,
    column2 INTEGER
);

-- Drop a table
DROP TABLE table_name;

-- Rename a table
ALTER TABLE old_table RENAME TO new_table;
```

---

### Managing Data

```sql
-- Insert data
INSERT INTO table_name (column1, column2) VALUES ('value1', 123);

-- Update data
UPDATE table_name SET column1 = 'new_value' WHERE id = 1;

-- Delete data
DELETE FROM table_name WHERE id = 1;

-- Select data
SELECT * FROM table_name;
SELECT column1, column2 FROM table_name WHERE column2 > 100;
```

---

### Database Admin Tasks

```sql
-- Create a database
CREATE DATABASE dbname;

-- Drop a database
DROP DATABASE dbname;

-- Backup database
pg_dump dbname > dbname_backup.sql

-- Restore database
psql dbname < dbname_backup.sql
```

---

### User Management

```sql
-- Create a user
CREATE USER username WITH PASSWORD 'password';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE dbname TO username;

-- Change user's password
ALTER USER username WITH PASSWORD 'newpassword';
```

---

### Checking Locks and Activity

```sql
-- View active connections
SELECT * FROM pg_stat_activity;

-- See who is connected and what queries are running
SELECT pid, usename, datname, client_addr, application_name, state, query
FROM pg_stat_activity;
```

---

### Maintenance

```sql
-- Vacuum (clean up and optimize)
VACUUM;

-- Analyze (update stats for planner)
ANALYZE;

-- Reindex a table
REINDEX TABLE table_name;
```

---

### Miscellaneous

```sql
-- List available commands in psql
\?

-- Describe a SQL function
\df function_name

-- Get server/version info
SELECT version();
```

---

### Useful psql Shortcuts

| Shortcut | Description          |
|----------|----------------------|
| \l       | List databases       |
| \c       | Connect to database  |
| \dt      | List tables          |
| \dn      | List schemas         |
| \du      | List roles/users     |
| \d table | Describe table       |
| \df      | Describe functions   |
| \q       | Quit psql            |

---

#### Notes

- Prefix SQL commands with `\` for internal psql commands.
- Use semicolon `;` to execute SQL statements in psql.
- You can run shell commands from within psql using `\!`.
- For more detailed help: [Postgres Official Docs](https://www.postgresql.org/docs/current/)

