# Databases Basics: Concepts, Building, and Tools (Linux & Windows)

A beginner-friendly guide to what databases are, how to design and build one, and which tools to use on Linux and Windows.

---

## 1. What Is a Database?

A **database** is an organized collection of data stored so it can be efficiently searched, updated, and managed. A **DBMS** (Database Management System) is the software that does this work, such as PostgreSQL, MySQL, or SQLite. People often say "database" for both.

**Why not just use spreadsheets or text files?**

| Need | Spreadsheet / files | Database |
|------|--------------------|----------|
| Many users at once | Conflicts, overwrites | Built-in concurrency control |
| Large data (millions of rows) | Slow, fragile | Indexed and fast |
| Data integrity (no duplicates, valid values) | Manual | Enforced by rules |
| Relationships between data | Awkward | Native (joins, keys) |
| Crash safety | Easy to corrupt | Transactions and recovery |

---

## 2. Types of Databases

### Relational (SQL)
Data lives in **tables** (rows and columns) linked by keys. Queried with **SQL**. Best for structured data like customers, orders, inventory, and finance.
- Examples: **PostgreSQL, MySQL, MariaDB, SQLite, Microsoft SQL Server, Oracle**

### NoSQL
Flexible models for data that doesn't fit neatly into tables.

| Type | Idea | Examples | Good for |
|------|------|----------|----------|
| Document | JSON-like documents | MongoDB, CouchDB | Content, catalogs, varied structures |
| Key-value | Key maps to a value | Redis, Valkey | Caching, sessions, queues |
| Column-family | Wide columns, huge scale | Cassandra, HBase | Massive write-heavy workloads |
| Graph | Nodes and relationships | Neo4j | Social networks, recommendations |
| Time-series | Timestamped measurements | InfluxDB, TimescaleDB | Metrics, IoT, monitoring |

> **Rule of thumb:** If you're unsure, start with a relational database. PostgreSQL or SQLite covers most projects.

---

## 3. Core Concepts

- **Table:** a collection of records about one kind of thing (e.g., `customers`).
- **Row (record):** a single entry in a table.
- **Column (field):** one attribute, with a data type (e.g., `email TEXT`).
- **Primary key:** a column (or set of columns) that uniquely identifies each row.
- **Foreign key:** a column that references another table's primary key, creating a relationship.
- **Index:** a lookup structure that speeds up searches, at the cost of extra storage and slower writes.
- **Constraint:** a rule such as `NOT NULL`, `UNIQUE`, `CHECK`, or `FOREIGN KEY`.
- **Query:** a request for data, usually written in SQL.
- **View:** a saved query that behaves like a table.
- **Transaction:** a group of operations that succeed or fail together.
- **ACID:** the guarantees of reliable relational databases:
  - **Atomicity:** all or nothing
  - **Consistency:** rules are always respected
  - **Isolation:** concurrent transactions don't interfere
  - **Durability:** committed data survives crashes

### Relationship types
- **One-to-many:** one customer has many orders (most common).
- **Many-to-many:** students and courses. Requires a **junction table** (e.g., `enrollments`).
- **One-to-one:** a user and their profile. Less common.

---

## 4. Designing a Database (Before You Build)

Good design saves enormous pain later.

1. **Define the purpose.** What questions must the database answer?
2. **List the entities** (things you track): customers, products, orders.
3. **List attributes** for each entity.
4. **Choose primary keys** (often an auto-incrementing ID).
5. **Define relationships** and add foreign keys.
6. **Normalize** to reduce duplication (see below).
7. **Choose data types and constraints.**
8. **Sketch an ERD** (Entity-Relationship Diagram) on paper or in a tool like dbdiagram.io, draw.io, or pgModeler.
9. **Add indexes** for columns you search or join on frequently.

### Normalization in brief

| Form | Rule | Example fix |
|------|------|-------------|
| 1NF | One value per cell, no repeating groups | Don't store `"red, blue, green"` in one column |
| 2NF | Every non-key column depends on the whole key | Move product details out of an order-items table |
| 3NF | No column depends on another non-key column | Store `city` in a locations table, not repeated with zip |

Denormalization (deliberate duplication) is sometimes used later for speed, but start normalized.

### Naming tips
- Use consistent, lowercase names with underscores: `order_items`, `created_at`.
- Singular or plural table names are both fine; just pick one and stay consistent.
- Avoid reserved words like `order` or `user` as table names (use `orders`, `users`, or quote them).

---

## 5. SQL Fundamentals

SQL is the standard language for relational databases.

### Create tables
```sql
CREATE TABLE customers (
    id          INTEGER PRIMARY KEY,
    name        TEXT NOT NULL,
    email       TEXT UNIQUE NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id           INTEGER PRIMARY KEY,
    customer_id  INTEGER NOT NULL REFERENCES customers(id),
    total        NUMERIC(10,2) NOT NULL CHECK (total >= 0),
    ordered_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
> Auto-increment syntax varies: `AUTOINCREMENT` (SQLite), `SERIAL` / `GENERATED ... AS IDENTITY` (PostgreSQL), `AUTO_INCREMENT` (MySQL), `IDENTITY(1,1)` (SQL Server).

### Insert, read, update, delete (CRUD)
```sql
INSERT INTO customers (name, email) VALUES ('Ada', 'ada@example.com');

SELECT * FROM customers;
SELECT name, email FROM customers WHERE name LIKE 'A%' ORDER BY name;

UPDATE customers SET name = 'Ada Lovelace' WHERE id = 1;

DELETE FROM customers WHERE id = 1;
```
> Always include a `WHERE` clause on `UPDATE` and `DELETE`, or every row is affected.

### Joins
```sql
SELECT c.name, o.total, o.ordered_at
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.total > 100;
```
- `INNER JOIN`: only matching rows
- `LEFT JOIN`: all rows from the left table, plus matches
- `RIGHT JOIN` / `FULL JOIN`: the mirror image / everything

### Aggregation
```sql
SELECT customer_id, COUNT(*) AS num_orders, SUM(total) AS spent
FROM orders
GROUP BY customer_id
HAVING SUM(total) > 500
ORDER BY spent DESC;
```

### Indexes
```sql
CREATE INDEX idx_orders_customer ON orders(customer_id);
```

### Transactions
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;   -- or ROLLBACK; to undo
```

---

## 6. Choosing a Tool

| Database | Type | Best for | Runs as server? | Linux | Windows |
|----------|------|----------|-----------------|:-----:|:-------:|
| **SQLite** | Relational | Learning, apps, small sites, prototypes | No (single file) | ✅ | ✅ |
| **PostgreSQL** | Relational | General purpose, complex queries, production | Yes | ✅ | ✅ |
| **MySQL / MariaDB** | Relational | Web apps (WordPress, PHP) | Yes | ✅ | ✅ |
| **SQL Server** (Express/Developer) | Relational | Microsoft ecosystem, .NET | Yes | ✅ | ✅ |
| **MongoDB** | Document | Flexible JSON-style data | Yes | ✅ | ✅ |
| **Redis / Valkey** | Key-value | Caching, sessions | Yes | ✅ | Via WSL or Docker |

**Quick recommendations**
- **Just learning SQL?** SQLite.
- **Building a real app?** PostgreSQL.
- **Working with WordPress or an existing PHP stack?** MySQL or MariaDB.
- **Microsoft/.NET shop?** SQL Server.
- **Caching or fast temporary data?** Redis.

### GUI (graphical) tools

| Tool | Works with | Platforms |
|------|-----------|-----------|
| **DBeaver Community** | Almost everything | Linux, Windows, macOS |
| **DB Browser for SQLite** | SQLite | Linux, Windows, macOS |
| **pgAdmin** | PostgreSQL | Linux, Windows, macOS |
| **HeidiSQL** | MySQL, MariaDB, PostgreSQL, SQL Server | Windows (runs on Linux via Wine) |
| **MySQL Workbench** | MySQL | Linux, Windows, macOS |
| **SQL Server Management Studio (SSMS)** | SQL Server | Windows only |
| **MongoDB Compass** | MongoDB | Linux, Windows, macOS |
| **DataGrip** (paid) | Many | Linux, Windows, macOS |

Command-line clients are also worth knowing: `sqlite3`, `psql`, `mysql`, `sqlcmd`, `mongosh`, `redis-cli`.

---

## 7. Installing and Building on Linux

Commands below are for **Ubuntu/Debian** (`apt`). On Fedora/RHEL use `dnf`, on Arch use `pacman`.

### 7.1 SQLite (easiest start)

```bash
sudo apt update
sudo apt install sqlite3
sqlite3 shop.db
```
Inside the prompt:
```sql
CREATE TABLE customers (id INTEGER PRIMARY KEY, name TEXT NOT NULL, email TEXT UNIQUE);
INSERT INTO customers (name, email) VALUES ('Ada', 'ada@example.com');
SELECT * FROM customers;
.tables
.schema customers
.quit
```
Your whole database is now the single file `shop.db`.

### 7.2 PostgreSQL

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl enable --now postgresql
sudo systemctl status postgresql
```

Create a user and database:
```bash
sudo -u postgres psql
```
```sql
CREATE USER appuser WITH PASSWORD 'choose-a-strong-password';
CREATE DATABASE shop OWNER appuser;
\q
```

Connect and work:
```bash
psql -h localhost -U appuser -d shop
```

Handy `psql` commands:

| Command | Meaning |
|---------|---------|
| `\l` | List databases |
| `\c dbname` | Connect to a database |
| `\dt` | List tables |
| `\d tablename` | Describe a table |
| `\du` | List users/roles |
| `\i file.sql` | Run a SQL file |
| `\q` | Quit |

### 7.3 MariaDB / MySQL

```bash
sudo apt update
sudo apt install mariadb-server
sudo systemctl enable --now mariadb
sudo mysql_secure_installation
sudo mysql
```
```sql
CREATE DATABASE shop;
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'choose-a-strong-password';
GRANT ALL PRIVILEGES ON shop.* TO 'appuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
```bash
mysql -u appuser -p shop
```
(For MySQL itself, use `sudo apt install mysql-server`.)

### 7.4 MongoDB and Redis

- **MongoDB:** not in default Ubuntu repositories. Follow the official install guide at [mongodb.com/docs/manual/administration/install-on-linux](https://www.mongodb.com/docs/manual/administration/install-on-linux/), then use `mongosh`.
- **Redis:** `sudo apt install redis-server`, then `redis-cli ping` (should reply `PONG`).

### 7.5 GUI on Linux

```bash
sudo apt install sqlitebrowser          # DB Browser for SQLite
sudo snap install dbeaver-ce            # DBeaver Community
```
pgAdmin: see [pgadmin.org/download](https://www.pgadmin.org/download/pgadmin-4-apt/) for the apt repository.

---

## 8. Installing and Building on Windows

### 8.1 SQLite

**Option A: winget (PowerShell or Command Prompt)**
```powershell
winget install SQLite.SQLite
sqlite3 shop.db
```
**Option B:** download the "sqlite-tools" zip from [sqlite.org/download](https://www.sqlite.org/download.html), extract it, and add the folder to your `PATH`.

The SQL commands are the same as in the Linux section. For a GUI, install **DB Browser for SQLite**:
```powershell
winget install DBBrowserForSQLite.DBBrowserForSQLite
```

### 8.2 PostgreSQL

1. Download the installer from [postgresql.org/download/windows](https://www.postgresql.org/download/windows/) (EnterpriseDB installer).
2. Run it. Keep the default components (PostgreSQL Server, pgAdmin 4, Command Line Tools).
3. Set a password for the `postgres` superuser. **Remember it.**
4. Keep the default port `5432`.

Or with winget (search for the current version ID first):
```powershell
winget search postgresql
```

Connect:
```powershell
psql -U postgres
```
```sql
CREATE USER appuser WITH PASSWORD 'choose-a-strong-password';
CREATE DATABASE shop OWNER appuser;
```
> If `psql` isn't recognized, add `C:\Program Files\PostgreSQL\<version>\bin` to your `PATH`, or use the **SQL Shell (psql)** from the Start menu. You can also use **pgAdmin 4**, installed alongside.

### 8.3 MySQL / MariaDB

- **MySQL:** download the **MySQL Installer** from [dev.mysql.com/downloads/installer](https://dev.mysql.com/downloads/installer/). It can also install MySQL Workbench.
- **MariaDB:** download from [mariadb.org/download](https://mariadb.org/download/), which bundles **HeidiSQL**.

Then:
```powershell
mysql -u root -p
```
Use the same SQL as in the Linux section.

### 8.4 SQL Server (Express or Developer edition)

1. Download **SQL Server Express** (free, size-limited) or **Developer** (free, non-production) from [microsoft.com/sql-server/sql-server-downloads](https://www.microsoft.com/en-us/sql-server/sql-server-downloads).
2. Run the installer and choose **Basic**.
3. Install **SQL Server Management Studio (SSMS)** from Microsoft as your GUI.
4. Or use the command line:
```powershell
sqlcmd -S localhost\SQLEXPRESS -E
```
```sql
CREATE DATABASE shop;
GO
USE shop;
GO
```

### 8.5 MongoDB and Redis

- **MongoDB:** download the MSI from [mongodb.com/try/download/community](https://www.mongodb.com/try/download/community). Also install **MongoDB Compass** (GUI).
- **Redis:** not officially supported natively on Windows. Use **WSL2** (Ubuntu) or **Docker**.

### 8.6 GUI on Windows
```powershell
winget install DBeaver.DBeaver.Community
winget install HeidiSQL.HeidiSQL
```

---

## 9. Optional: Docker (Same Steps on Linux and Windows)

Docker lets you run any database without installing it directly, and delete it cleanly afterward.

**PostgreSQL:**
```bash
docker run --name pg-dev -e POSTGRES_PASSWORD=devpass -e POSTGRES_DB=shop \
  -p 5432:5432 -v pgdata:/var/lib/postgresql/data -d postgres
```

**MariaDB:**
```bash
docker run --name maria-dev -e MARIADB_ROOT_PASSWORD=devpass -e MARIADB_DATABASE=shop \
  -p 3306:3306 -v mariadata:/var/lib/mysql -d mariadb
```

**MongoDB:**
```bash
docker run --name mongo-dev -p 27017:27017 -v mongodata:/data/db -d mongo
```

**Redis:**
```bash
docker run --name redis-dev -p 6379:6379 -d redis
```

Stop/start/remove:
```bash
docker stop pg-dev
docker start pg-dev
docker rm -f pg-dev
```
> The `-v` flag stores data in a Docker volume so it survives container restarts. On Windows, install **Docker Desktop** (uses WSL2).

---

## 10. Connecting From Code

### Python + SQLite (no install needed; it's built in)
```python
import sqlite3

conn = sqlite3.connect("shop.db")
cur = conn.cursor()
cur.execute("CREATE TABLE IF NOT EXISTS customers (id INTEGER PRIMARY KEY, name TEXT, email TEXT UNIQUE)")
cur.execute("INSERT INTO customers (name, email) VALUES (?, ?)", ("Ada", "ada@example.com"))
conn.commit()

for row in cur.execute("SELECT * FROM customers"):
    print(row)
conn.close()
```

### Python + PostgreSQL
```bash
pip install "psycopg[binary]"
```
```python
import psycopg

with psycopg.connect("host=localhost dbname=shop user=appuser password=yourpassword") as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT name, email FROM customers WHERE email = %s", ("ada@example.com",))
        print(cur.fetchall())
```

> **Always use parameterized queries** (`?` or `%s` placeholders) as shown above. Never build SQL by pasting user input into strings; that leads to **SQL injection**.

Other languages have equivalents: `node-postgres` / `better-sqlite3` (Node.js), JDBC (Java), `Npgsql` / `Microsoft.Data.SqlClient` (C#), PDO (PHP).

---

## 11. Backups and Restore

| Database | Backup | Restore |
|----------|--------|---------|
| SQLite | Copy the `.db` file (or `.backup backup.db` in `sqlite3`) | Copy it back |
| PostgreSQL | `pg_dump -U appuser shop > shop.sql` | `psql -U appuser -d shop -f shop.sql` |
| MySQL/MariaDB | `mysqldump -u appuser -p shop > shop.sql` | `mysql -u appuser -p shop < shop.sql` |
| MongoDB | `mongodump --db shop` | `mongorestore` |
| SQL Server | `BACKUP DATABASE shop TO DISK = 'C:\backups\shop.bak'` | `RESTORE DATABASE ...` |

**Backup rules of thumb:** automate them (cron on Linux, Task Scheduler on Windows), store copies off the machine, and **test restoring** at least once.

---

## 12. Security Basics

- Never use the admin account (`root`, `postgres`, `sa`) for your application. Create a limited user.
- Use **strong, unique passwords** and keep them out of source code (use environment variables).
- **Don't expose database ports** (5432, 3306, 1433, 27017, 6379) to the public internet. Bind to `localhost` or use a firewall/VPN.
- Grant **least privilege**: only the permissions each user needs.
- Use parameterized queries to prevent SQL injection.
- Keep the database software **updated**.
- Encrypt connections (TLS) when accessing over a network.
- Back up regularly.

---

## 13. Performance Tips

- Add indexes to columns used in `WHERE`, `JOIN`, and `ORDER BY`, but don't index everything.
- Use `EXPLAIN` (or `EXPLAIN ANALYZE` in PostgreSQL) to see how a query runs.
- Select only the columns you need instead of `SELECT *`.
- Use `LIMIT` when you only need a few rows.
- Avoid running queries in a loop; use a single joined query instead.
- Use connection pooling in applications.

---

## 14. Common Beginner Mistakes

| Mistake | Fix |
|---------|-----|
| `UPDATE`/`DELETE` without `WHERE` | Test with `SELECT` first; use transactions |
| Storing lists in one column | Create a related table |
| No primary key | Every table should have one |
| Storing money as floating point | Use `NUMERIC`/`DECIMAL` or integer cents |
| Storing passwords in plain text | Hash them with bcrypt or argon2 in your app |
| Storing dates as text | Use proper date/time types |
| No backups | Automate them |
| Using the admin account in apps | Create a limited user |

---

## 15. Suggested Learning Path

1. Install **SQLite** and **DB Browser for SQLite**.
2. Practice `CREATE`, `INSERT`, `SELECT`, `UPDATE`, `DELETE`.
3. Learn `JOIN`s, `GROUP BY`, and subqueries.
4. Design a small project (library, shop, or school) with 3 to 5 related tables.
5. Move to **PostgreSQL** and learn users, permissions, and indexes.
6. Connect it from a programming language.
7. Learn backups, transactions, and query optimization.
8. Explore NoSQL (MongoDB or Redis) once relational concepts are solid.

### Free resources
- [SQLite Tutorial](https://www.sqlitetutorial.net/)
- [PostgreSQL Tutorial](https://www.postgresqltutorial.com/) and the [official docs](https://www.postgresql.org/docs/)
- [SQLBolt](https://sqlbolt.com/) (interactive lessons)
- [SQL Murder Mystery](https://mystery.knightlab.com/) (a game for practicing SQL)
- [MongoDB University](https://learn.mongodb.com/)

---

## Quick Reference

| Task | SQLite | PostgreSQL | MySQL/MariaDB |
|------|--------|------------|---------------|
| Open CLI | `sqlite3 file.db` | `psql -U user -d db` | `mysql -u user -p db` |
| List tables | `.tables` | `\dt` | `SHOW TABLES;` |
| Describe table | `.schema table` | `\d table` | `DESCRIBE table;` |
| List databases | (n/a, one file) | `\l` | `SHOW DATABASES;` |
| Run a file | `.read file.sql` | `\i file.sql` | `source file.sql` |
| Quit | `.quit` | `\q` | `EXIT;` |
