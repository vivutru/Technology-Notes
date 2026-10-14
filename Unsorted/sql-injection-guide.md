# SQL Injection: What It Is, How It Works, and How to Defend Against It

A defensive guide for developers, students, and anyone who builds or maintains apps that talk to a database.

> **Purpose:** Understanding how attacks work is the best way to prevent them. Only test these ideas on systems you own or have written permission to test (like a local practice app). Attacking other people's systems is illegal.

---

## 1. What Is SQL Injection?

**SQL injection (SQLi)** happens when an application builds a SQL query by pasting **untrusted user input** directly into the query text. An attacker can then supply input that **changes the meaning of the query**, so the database runs commands the developer never intended.

The root cause is simple: **the database can't tell the difference between the developer's SQL code and the attacker's data**, because they were mixed into one string.

### What can go wrong

| Impact | Description |
|--------|-------------|
| **Authentication bypass** | Logging in as any user (including admin) without a password |
| **Data theft** | Reading tables the user should never see (emails, password hashes, payment info) |
| **Data tampering** | Changing or inserting records (prices, balances, roles) |
| **Data destruction** | Deleting rows or dropping tables |
| **Privilege escalation** | Gaining admin rights within the app or database |
| **Server compromise** | On badly configured servers, running OS commands or reading files |

SQL injection has been on the **OWASP Top 10** list of web risks for many years (it falls under the "Injection" category), and it is still found in real applications today.

---

## 2. What a Vulnerable App Looks Like

Imagine a simple login form with two fields: **username** and **password**.

### Vulnerable code (Python, for illustration)

```python
# ❌ VULNERABLE: never do this
username = request.form["username"]
password = request.form["password"]

query = "SELECT * FROM users WHERE username = '" + username + "' AND password = '" + password + "'"
result = db.execute(query)

if result.fetchone():
    print("Login successful")
```

The same mistake looks similar in every language:

```php
// ❌ PHP
$query = "SELECT * FROM users WHERE username = '$user' AND password = '$pass'";
```
```javascript
// ❌ Node.js
const query = `SELECT * FROM users WHERE username = '${user}' AND password = '${pass}'`;
```
```java
// ❌ Java
String query = "SELECT * FROM users WHERE username = '" + user + "'";
```

**The telltale sign:** user input is joined into a query using string concatenation (`+`), interpolation (`f"..."`, `${...}`, `"$var"`), or `format()`.

---

## 3. A Dummy Example, Step by Step

### The setup

A table called `users`:

| id | username | password | role |
|----|----------|----------|------|
| 1 | admin | S3cret!pass | admin |
| 2 | alice | alicepw | user |
| 3 | bob | bobpw | user |

(In reality, passwords should be hashed. Plain text is used here only to keep the example simple.)

### Normal behavior

Alice types:
- Username: `alice`
- Password: `alicepw`

The code builds this query:

```sql
SELECT * FROM users WHERE username = 'alice' AND password = 'alicepw'
```

One row is returned, so Alice logs in. 

### Attack 1: Authentication bypass

An attacker types:
- Username: `admin' --`
- Password: *(anything)*

The code builds:

```sql
SELECT * FROM users WHERE username = 'admin' --' AND password = 'anything'
```

What happened:
- The `'` after `admin` **closed the string** early.
- `--` starts a **SQL comment**, so everything after it, including the password check, is ignored.
- The query now means: *"find the user named admin"*, with no password required.

The attacker is logged in as **admin**.

### Attack 2: The "always true" condition

An attacker types:
- Username: `anything`
- Password: `' OR '1'='1`

The code builds:

```sql
SELECT * FROM users WHERE username = 'anything' AND password = '' OR '1'='1'
```

Because `'1'='1'` is always true and `OR` only needs one side to be true, the `WHERE` clause matches **every row**. The app often logs in as the **first user in the table**, which is frequently the admin.

### Attack 3: Data extraction in a search box

Suppose a product search does this:

```python
# ❌ VULNERABLE
query = "SELECT name, price FROM products WHERE category = '" + category + "'"
```

Normal input: `books` returns books.

An attacker types this into the category field:

```
' UNION SELECT username, password FROM users --
```

The query becomes:

```sql
SELECT name, price FROM products WHERE category = '' UNION SELECT username, password FROM users --'
```

`UNION` glues a **second query's results** onto the first. The page that normally shows product names and prices now displays **usernames and passwords**.

### Attack 4: Data destruction (the classic "Bobby Tables")

If the database driver allows multiple statements in one call, input like:

```
'; DROP TABLE users; --
```

can turn one query into two:

```sql
SELECT * FROM products WHERE category = ''; DROP TABLE users; --'
```

Many modern drivers block multiple statements by default, but you should never rely on that.

*(This is the joke behind the well-known xkcd comic "Exploits of a Mom".)*

---

## 4. Types of SQL Injection

| Type | How it works | What the attacker sees |
|------|--------------|------------------------|
| **In-band (classic)** | Results come back in the normal page response | Data directly on the page |
| **Error-based** | Attacker triggers database errors that leak information | Detailed error messages |
| **UNION-based** | Uses `UNION` to append extra query results | Extra data mixed into normal output |
| **Blind (boolean-based)** | No data shown; attacker infers answers from true/false page differences | Page changes subtly |
| **Blind (time-based)** | Attacker makes the database pause if a condition is true and measures the delay | Response time |
| **Out-of-band** | Database is made to send data through another channel (e.g., DNS/HTTP request) | Data arrives elsewhere |
| **Second-order** | Malicious input is stored safely first, then used unsafely in a later query | Effect appears later |

Even if your app shows **no errors and no data**, it can still be vulnerable to blind SQL injection.

---

## 5. Where Injection Can Hide

It's not just login forms. Any place where external input reaches a query:

- Search boxes and filters
- URL parameters (`/product?id=5`)
- Form fields (contact, registration, checkout)
- HTTP headers (`User-Agent`, `Referer`, cookies)
- JSON or XML API request bodies
- Sorting parameters (`?sort=price`) and `ORDER BY` clauses
- Data pulled from files, other systems, or **your own database** (second-order)
- Hidden form fields and mobile app API calls

**Rule:** treat *all* input as untrusted, even hidden fields, dropdown values, and data from "internal" sources.

---

## 6. How to Protect Yourself

### Defense 1: Parameterized queries (prepared statements) ✅ The #1 fix

Send the SQL **structure** and the **data** separately. The database compiles the query first, then plugs in values strictly as data, so input can never become code.

**Python (sqlite3)**
```python
# ✅ SAFE
cur.execute(
    "SELECT * FROM users WHERE username = ? AND password_hash = ?",
    (username, password_hash)
)
```

**Python (PostgreSQL, psycopg)**
```python
# ✅ SAFE
cur.execute("SELECT * FROM users WHERE username = %s", (username,))
```

**Node.js (pg)**
```javascript
// ✅ SAFE
const result = await pool.query(
  "SELECT * FROM users WHERE username = $1",
  [username]
);
```

**PHP (PDO)**
```php
// ✅ SAFE
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = :user");
$stmt->execute(['user' => $username]);
```

**Java (JDBC)**
```java
// ✅ SAFE
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE username = ?");
ps.setString(1, username);
ResultSet rs = ps.executeQuery();
```

**C# (ADO.NET)**
```csharp
// ✅ SAFE
var cmd = new SqlCommand("SELECT * FROM users WHERE username = @user", conn);
cmd.Parameters.AddWithValue("@user", username);
```

**Replaying the attack against the safe version:** if an attacker enters `admin' --`, the database searches for a user literally named `admin' --`. No such user exists, so the login simply fails. The quote and dashes are **just characters**.

### Defense 2: Use an ORM or query builder (correctly)

ORMs like SQLAlchemy, Django ORM, Hibernate, Entity Framework, Prisma, and Sequelize use parameters by default.

```python
# ✅ Django ORM
User.objects.filter(username=username)
```

**Caution:** ORMs offer "raw SQL" escape hatches. If you concatenate strings into `raw()`, `execute()`, or `.extra()`, you're vulnerable again.

```python
# ❌ Still vulnerable, even inside an ORM
User.objects.raw("SELECT * FROM users WHERE username = '" + username + "'")

# ✅ Safe raw query with parameters
User.objects.raw("SELECT * FROM users WHERE username = %s", [username])
```

### Defense 3: Allow-list what can't be parameterized

Parameters work for **values**, not for **table names, column names, or sort direction**. For those, validate against a fixed list:

```python
# ✅ Allow-list approach
ALLOWED_SORT = {"price", "name", "created_at"}
ALLOWED_DIR = {"ASC", "DESC"}

if sort not in ALLOWED_SORT or direction not in ALLOWED_DIR:
    raise ValueError("Invalid sort option")

query = f"SELECT * FROM products ORDER BY {sort} {direction}"
```

Here the user never supplies the SQL, only a choice from a list the developer controls.

### Defense 4: Validate and sanitize input (as a second layer)

- Check **type** (is `id` really an integer?), **length**, **format**, and **range**.
- Reject anything unexpected instead of trying to "clean" it.
- Cast numbers: `int(user_id)`.

> ⚠️ Validation alone is **not** enough. Escaping and blacklists (blocking words like `DROP` or `--`) are notoriously easy to bypass. Use them only *in addition to* parameterized queries.

### Defense 5: Least privilege for database accounts

The app's database user should have only the permissions it truly needs.

```sql
-- PostgreSQL example: a limited app account
CREATE USER webapp WITH PASSWORD 'strong-unique-password';
GRANT SELECT, INSERT, UPDATE ON orders, customers TO webapp;
-- No DROP, no ALTER, no access to unrelated tables
```

- Never connect as `root`, `postgres`, `sa`, or `sysadmin`.
- Use **separate accounts** for read-only and write operations where practical.
- Deny access to system tables, file functions, and OS command features.

This doesn't prevent injection, but it **limits the damage** if one slips through.

### Defense 6: Stored procedures (used safely)

Stored procedures can help, but only if they use parameters internally. A procedure that concatenates strings inside itself is just as vulnerable.

### Defense 7: Hide detailed errors

- Show users a **generic message** ("Something went wrong").
- Log the detailed error **server-side only**.
- Turn off debug mode in production.

Detailed database errors give attackers a roadmap.

### Defense 8: Hash passwords properly

Even if data is stolen, hashed passwords are much harder to use.

- Use **bcrypt**, **argon2**, or **scrypt**, never plain text, MD5, or plain SHA-1.
- Compare hashes in your app code (verify function), and query only by username.

```python
# ✅ Look up by username only, then verify the hash in code
row = cur.execute("SELECT password_hash FROM users WHERE username = ?", (username,)).fetchone()
if row and bcrypt.checkpw(password.encode(), row[0]):
    ...
```

### Defense 9: Web Application Firewall (WAF)

A WAF (e.g., Cloudflare, AWS WAF, ModSecurity) can block common injection patterns. It's a **helpful extra layer**, not a substitute for secure code.

### Defense 10: Keep everything updated

Update your database server, frameworks, drivers, and libraries. Old versions can contain their own injection-related vulnerabilities.

---

## 7. Defense Summary

| Priority | Defense | Stops injection? |
|:--------:|---------|:----------------:|
| 1 | **Parameterized queries / prepared statements** | ✅ Yes |
| 2 | ORM used correctly (no string-built raw SQL) | ✅ Yes |
| 3 | Allow-lists for identifiers (table/column/sort) | ✅ Yes |
| 4 | Input validation | ⚠️ Helps, not sufficient alone |
| 5 | Least-privilege DB account | 🛡️ Limits damage |
| 6 | Generic error messages | 🛡️ Reduces information leaks |
| 7 | Password hashing | 🛡️ Limits damage |
| 8 | WAF | 🛡️ Extra layer |
| 9 | Updates and patching | 🛡️ Extra layer |

---

## 8. How to Check Whether Your Code Is Vulnerable

### Code review checklist

Search your codebase for SQL built by joining strings:

- [ ] Any `"SELECT ... " + variable`
- [ ] Any `f"SELECT ... {variable}"` or `.format()` in SQL
- [ ] Any `${variable}` inside SQL template strings
- [ ] Any `"...$variable..."` in PHP queries
- [ ] Any `execute()`, `raw()`, `query()` call that receives a built string
- [ ] Dynamic `ORDER BY`, `LIMIT`, or table names taken from user input

Every one of these should either use **parameters** or an **allow-list**.

### Automated tools (for your own systems)

| Tool | Purpose |
|------|---------|
| **Semgrep** | Static analysis: finds string-built SQL in source code |
| **Bandit** (Python), **SpotBugs** / **Find Security Bugs** (Java), **ESLint security plugins** (JS) | Language-specific static checks |
| **OWASP ZAP** | Free scanner for testing your own running web apps |
| **sqlmap** | Open-source SQLi testing tool, for **authorized** testing only |
| **Dependency scanners** (Dependabot, `npm audit`, `pip-audit`) | Flag vulnerable libraries |

### Safe practice environments

Learn hands-on legally using intentionally vulnerable apps run locally:

- **OWASP Juice Shop**
- **DVWA** (Damn Vulnerable Web Application)
- **WebGoat**
- **PortSwigger Web Security Academy** (free labs online)
- **HackTheBox / TryHackMe** (structured training)

---

## 9. If You Think You've Been Hit

1. **Contain:** take the affected app offline or block the vulnerable endpoint.
2. **Preserve logs** (web server, database, application) for investigation.
3. **Patch** the vulnerable code using parameterized queries.
4. **Rotate secrets:** database passwords, API keys, session secrets.
5. **Force password resets** if user credentials may have been exposed.
6. **Assess what data was accessed** and check legal or notification obligations (privacy and breach-notification laws vary by country).
7. **Review** other endpoints for the same pattern.

---

## 10. Common Myths

| Myth | Reality |
|------|---------|
| "I escape quotes, so I'm safe." | Escaping is error-prone and can be bypassed (encoding tricks, numeric contexts). Use parameters. |
| "I block words like DROP and UNION." | Blacklists are easily evaded (case changes, comments, encodings). |
| "My app has no error messages, so it's safe." | Blind SQL injection works without visible errors. |
| "It's a small site; nobody would target it." | Attacks are automated and scan the whole internet. |
| "I use an ORM, so I'm immune." | Raw queries and string-built filters inside ORMs are still vulnerable. |
| "JavaScript validation on the form protects me." | Client-side checks are trivial to bypass. Always validate on the server. |
| "Hidden fields and dropdowns can't be changed." | Anyone can edit requests directly. |

---

## Quick Reference Card

**Golden rule:** *Never build SQL by joining strings with user input.*

```text
❌  "SELECT * FROM users WHERE name = '" + name + "'"
✅  "SELECT * FROM users WHERE name = ?"   +   parameter: name
```

1. Use **parameterized queries** everywhere.
2. **Allow-list** column names, table names, and sort order.
3. Run the app with a **least-privilege** database account.
4. Show **generic errors**; log details privately.
5. **Hash** passwords with bcrypt/argon2.
6. **Test your own apps** with tools like OWASP ZAP and Semgrep.
7. Keep software **patched**.

---

### Further Reading

- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [PortSwigger Web Security Academy: SQL injection](https://portswigger.net/web-security/sql-injection)
- [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)
