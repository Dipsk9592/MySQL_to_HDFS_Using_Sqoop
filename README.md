# Loudacre Inc. — MySQL → Apache Sqoop → Hadoop HDFS

**A beginner-friendly, step-by-step practical lab with real terminal results**  
**Questions 1–7 | Cloudera training virtual machine | Community learning resource**

> This guide documents one learner's successful classroom exercise. It is intended to help classmates understand *why* each command is used, not just copy commands. Screenshots are included in the `screenshots/` folder. The original dataset is a classroom training dataset, not a universal database.

## Start here: what are we doing?

Loudacre Inc. is a fictional electronics retailer in the exercise. The `loudacre` database was already installed in the training MySQL server. We did **not** create it.

Think of the systems as a warehouse operation:

- **MySQL** = an organized office containing databases and tables (original records).
- **SQL** = the language used to inspect and query those records.
- **Sqoop** = a delivery service that **copies** rows from MySQL into Hadoop.
- **HDFS** = a distributed warehouse storing the copied records as **files**, not live MySQL tables.
- **Mapper** = one parallel worker handling a portion of the import.
- **Terminal** = where we enter commands. We used **two terminals**: one inside MySQL (`mysql>`) and one normal Linux shell (`[training@localhost ~]$`).

```text
MySQL server
└── loudacre (database)
    ├── accounts (table)
    ├── accountdevice (table)
    ├── webpage (table)
    └── customerservicerep (table)
             │
             │ Sqoop imports selected rows/columns
             ▼
HDFS (distributed file storage)
├── /Loudacre/accounts/
├── /Loudacre/accountdevice/
├── /Loudacre/webpage/
└── /user/training/ ... (other exercise outputs)
```

**Important:** Sqoop imports make copies. Original MySQL rows remain in MySQL, and future changes in MySQL do **not** automatically update earlier HDFS copies.

## Before you begin

This exercise was performed on an older **Cloudera training Linux virtual machine**, with MySQL, Hadoop/HDFS, YARN and Sqoop already installed and running. The commands below are **environment-specific**, not a universal setup recipe.

- You need access to a working Linux/Hadoop environment and a MySQL server containing the `loudacre` tables.
- The classroom username/password were `training` / `training`; replace these with your own credentials if different.
- This environment used `localhost` for MySQL. A remote database would have a different host.
- **Mac users:** Hadoop is not inherently incompatible with macOS. This particular old VM image or virtualization setup may be difficult to run on a Mac, especially Apple Silicon. You can follow the concepts and commands in a compatible Linux VM, remote lab server, or course-provided environment. **Do not assume these commands will work without an installed and configured cluster.**
- **Security:** The inline password is copied from the classroom demonstration only. For real work, avoid exposing passwords in shell history or process listings; use `-P`/`--password-prompt` or an approved secret-management approach.
- Each Sqoop `--target-dir` generally needs to be **absent before a fresh import**. If you have already run the exercise, choose a new output directory or carefully manage the existing one; do not delete important files blindly.

### Which terminal?

**Terminal A — MySQL:** Start from a normal Linux terminal and connect:

```bash
mysql -utraining -ptraining
```

When you see `mysql>`, type **SQL commands** ending in `;`. For example:

```sql
SHOW DATABASES;
```

**Terminal B — Linux:** Open a **second** normal terminal, with a prompt such as `[training@localhost ~]$`. Run **Sqoop and HDFS commands** here, without SQL-style semicolons.

A terminal command that begins with `sqoop` or `hdfs` does **not** belong at the `mysql>` prompt.

---

## Question 1 — List all databases in the RDBMS

**Goal:** Find out which databases exist on the MySQL server.

**Terminal A (`mysql>`):**

```sql
SHOW DATABASES;
```

**Observed result:** six databases: `information_schema`, `hue`, `loudacre`, `metastore`, `mysql`, and `test`.

![Question 1: MySQL databases](screenshots/01-databases.png)

**Meaning:** `loudacre` is the training database name, **not** another database product or an RDBMS standard. MySQL is the database management system; `loudacre` is one database inside it.

## Question 2 — Create `/Loudacre` in HDFS

**Goal:** Prepare a top-level HDFS folder for the imported tables.

**Terminal B (Linux):**

```bash
hdfs dfs -mkdir /Loudacre
```

Verify:

```bash
hdfs dfs -ls /Loudacre
```

An empty listing without an error is normal: the directory exists but has no files yet. You can also inspect the parent directory:

```bash
hdfs dfs -ls /
```

**Meaning:** `-mkdir` makes a directory; `-ls` lists directory contents. **HDFS `/Loudacre` is not the same as a local Linux directory**. Creating a folder directly under `/` requires HDFS permissions; use an authorized location if your environment denies access.

**Screenshot note:** No separate screenshot of the empty directory was provided; the later imports into `/Loudacre/...` demonstrate it was accessible.

## Question 3 — Import three tables separately into `/Loudacre`

**Goal:** Copy three complete MySQL tables to **three separate HDFS directories**. Run the commands **one at a time** in Terminal B. `--num-mappers 1` makes the first exercise easier to inspect.

### 3A. Import `accounts`

```bash
sqoop import --connect jdbc:mysql://localhost/loudacre --username training --password training --table accounts --target-dir /Loudacre/accounts --num-mappers 1
```

Verify:

```bash
hdfs dfs -ls /Loudacre/accounts
```

**Observed:** **129,761 records**; `_SUCCESS` and `part-m-00000` were present.

![Accounts import completed](screenshots/02-accounts-import.png)
![Accounts files in HDFS](screenshots/03-accounts-verified.png)

### 3B. Import `accountdevice`

```bash
sqoop import --connect jdbc:mysql://localhost/loudacre --username training --password training --table accountdevice --target-dir /Loudacre/accountdevice --num-mappers 1
```

Verify:

```bash
hdfs dfs -ls /Loudacre/accountdevice
```

**Observed:** **194,764 records**, plus `_SUCCESS` and `part-m-00000`.

![Accountdevice HDFS verification](screenshots/04-accountdevice-verified.png)

> **Common confusion:** `device` and `accountdevice` are **different tables**. In an earlier exercise, `device` had only **50 records**; the `accountdevice` table imported here had **194,764 records**. Nothing went wrong.

### 3C. Import `webpage`

```bash
sqoop import --connect jdbc:mysql://localhost/loudacre --username training --password training --table webpage --target-dir /Loudacre/webpage --num-mappers 1
```

Verify:

```bash
hdfs dfs -ls /Loudacre/webpage
```

**Observed:** **53 records**, plus `_SUCCESS` and `part-m-00000`.

![Webpage import and verification](screenshots/05-webpage-verified.png)

**What are the two files?** `part-m-00000` contains the actual imported rows; `_SUCCESS` is a zero-byte marker indicating the Hadoop output job completed. With one mapper, there is typically one `part-m-...` data file.

## Question 4 — Import `accounts` with 10 mappers, `:` delimiter, split by `state`

**Goal:** Learn **parallel importing**, **choosing a split column**, and **formatting output**.

- **10 mappers** = ask Sqoop for ten parallel map tasks instead of one.
- **`state` split column** = use the `state` values to partition which rows each mapper reads; **not** a separate folder for every state.
- **`:` field delimiter** = separate output fields using a colon rather than the usual comma.

**First inspect the table** in Terminal A:

```sql
USE loudacre;
DESCRIBE accounts;
```

The screenshot confirms that `accounts` has a `state` column of type `varchar(255)`.

![Accounts table schema showing state](screenshots/06-accounts-schema.png)

**Terminal B — example command for this older Sqoop version:**

```bash
sqoop import -Dorg.apache.sqoop.splitter.allow_text_splitter=true --connect jdbc:mysql://localhost/loudacre --username training --password training --table accounts --target-dir /user/training/accounts_q4 --num-mappers 10 --split-by state --fields-terminated-by ':'
```

**Important compatibility detail:** Splitting on text (`state` is VARCHAR) is not supported or recommended by all Sqoop versions. The `-Dorg.apache.sqoop.splitter.allow_text_splitter=true` setting is a **version-dependent workaround**, not a general best practice. In production, a numeric, evenly distributed split key is usually better. The exact originally typed command is **not visible in the screenshots**; the command above reconstructs the observed behavior for this older lab environment.

Verify:

```bash
hdfs dfs -ls /user/training/accounts_q4
```

**Observed:** 11 entries: one `_SUCCESS` marker and **ten** `part-m-00000` through `part-m-00009` files. **129,761 records** were retrieved. Several part files were **0 bytes** because the chosen text-based split was uneven; ten mappers do not guarantee ten nonempty outputs or faster execution.

![Question 4: 10 mapper output files](screenshots/07-ten-mappers.png)

Preview a few lines:

```bash
hdfs dfs -cat /user/training/accounts_q4/part-m-00000 | head -n 3
```

![Question 4: colon-separated output](screenshots/08-colon-separated-output.png)

The visible output has `:` between fields, demonstrating the requested formatting. You may see `cat: Unable to write to output stream` after piping into `head`: `head` stops after three lines, closing the pipe while `hdfs dfs -cat` is still writing. **It does not mean the Sqoop import failed.**

## Question 5 — Import full names (first name + space + last name)

**Goal:** Produce a **single field per person**, such as `Donald Becton`, rather than importing every account column.

**Preview in Terminal A:**

```sql
SELECT CONCAT(first_name, ' ', last_name) AS full_name
FROM accounts
LIMIT 5;
```

**Observed first five:** Donald Becton, Donna Jones, Dorthy Chalmers, Leila Spencer, Anita Laughlin.

![Question 5: full names preview in MySQL](screenshots/09-full-names-preview.png)

**Terminal B — Sqoop query import:**

```bash
sqoop import --connect jdbc:mysql://localhost/loudacre --username training --password training --query "SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM accounts WHERE \$CONDITIONS" --target-dir /user/training/full_names_q5 --num-mappers 1
```

Why the unusual `\$CONDITIONS`? Sqoop's free-form query mode requires its `$CONDITIONS` placeholder, which Sqoop substitutes when it builds the SQL. The backslash stops the Linux shell from expanding `$CONDITIONS` before Sqoop sees it. This command is a **reconstructed equivalent** consistent with the observed result; the exact import command was not captured in the screenshot.

Verify and preview:

```bash
hdfs dfs -ls /user/training/full_names_q5
hdfs dfs -cat /user/training/full_names_q5/part-m-00000 | head -n 5
```

**Observed:** **129,761 records**; the HDFS output contains one full name per line, matching the MySQL preview.

![Question 5: imported full names in HDFS](screenshots/10-full-names-hdfs.png)

## Question 6 — Import only `accountdevice` records with `device_id` between 10 and 50

**Goal:** Filter rows **during import**, rather than copying the entire table.

**Inspect in Terminal A:**

```sql
DESCRIBE accountdevice;
```

The screenshot confirms that `device_id` is an integer column.

![Question 6: accountdevice schema](screenshots/11-accountdevice-schema.png)

**Terminal B:**

```bash
sqoop import --connect jdbc:mysql://localhost/loudacre --username training --password training --table accountdevice --where "device_id BETWEEN 10 AND 50" --target-dir /user/training/accountdevice_q6 --num-mappers 1
```

`BETWEEN 10 AND 50` **includes** 10 and 50.

Verify:

```bash
hdfs dfs -ls /user/training/accountdevice_q6
```

**Observed:** **156,825 matching records**, `_SUCCESS`, and `part-m-00000`.

![Question 6: filtered import verified](screenshots/12-filtered-device-import.png)

This is different from Question 3, which imported **all 194,764** `accountdevice` rows. We did not delete or change the source rows in MySQL.

## Question 7 — Find Roger Miller's account number and copy it to HDFS

**Goal:** Select one specific customer's account number. The classroom question says “Copy the acct no.” We first looked it up in MySQL, then also imported that single result into HDFS for a practical demonstration.

**Inspect in Terminal A:**

```sql
DESCRIBE customerservicerep;
```

The two columns are `acct_num` and `cs_name`.

![Question 7: customer service representative schema](screenshots/13-service-rep-schema.png)

**Look up the answer in Terminal A:**

```sql
SELECT acct_num
FROM customerservicerep
WHERE cs_name = 'Roger Miller';
```

**Observed result: `198`.**

![Question 7: MySQL lookup returns 198](screenshots/14-roger-miller-mysql.png)

**Optional HDFS import in Terminal B (also performed in this lab):**

```bash
sqoop import --connect jdbc:mysql://localhost/loudacre --username training --password training --query "SELECT acct_num FROM customerservicerep WHERE cs_name = 'Roger Miller' AND \$CONDITIONS" --target-dir /user/training/roger_miller_q7 --num-mappers 1
```

Verify the actual value:

```bash
hdfs dfs -cat /user/training/roger_miller_q7/part-m-00000
```

**Observed:** one record, containing `198`.

![Question 7: HDFS contains 198](screenshots/15-roger-miller-hdfs.png)

---

## Final results at a glance

| Question | What we did | Verified result |
|---|---|---|
| Q1 | Listed MySQL databases | 6 databases |
| Q2 | Created HDFS directory | `/Loudacre` accessible |
| Q3a | Imported `accounts` | 129,761 rows |
| Q3b | Imported `accountdevice` | 194,764 rows |
| Q3c | Imported `webpage` | 53 rows |
| Q4 | Imported accounts with 10 mappers, colon delimiter, `state` split | 129,761 rows; 10 part files (some empty) |
| Q5 | Imported combined full names | 129,761 rows; one name per line |
| Q6 | Imported only device IDs 10–50 | 156,825 rows |
| Q7 | Found and imported Roger Miller's account number | `198` |

## Command cheat sheet

| Command/option | Plain-English meaning |
|---|---|
| `SHOW DATABASES;` | Show MySQL databases (inside MySQL) |
| `USE loudacre;` | Choose the database |
| `DESCRIBE accounts;` | Inspect a table's columns |
| `SELECT ... WHERE ...;` | Retrieve matching database rows |
| `hdfs dfs -mkdir PATH` | Make an HDFS directory |
| `hdfs dfs -ls PATH` | List HDFS files/directories |
| `hdfs dfs -cat FILE` | Print an HDFS file's contents |
| `sqoop import` | Copy rows from an RDBMS into Hadoop |
| `--connect` | Database connection address |
| `--table` | Source MySQL table |
| `--target-dir` | HDFS destination |
| `--num-mappers` | Number of parallel mapper tasks |
| `--split-by` | Column used to divide mapper work |
| `--fields-terminated-by ':'` | Colon as output field separator |
| `--where` | Filter source rows |
| `--query` | Use a custom SQL query as the source |
| `_SUCCESS` | Marker that output job completed |
| `part-m-00000` | A mapper's output data file |

## Common beginner questions

**Is HDFS another SQL database?** No. HDFS stores files. Tools such as Hive or Spark can process/query data stored there; Sqoop does not magically turn HDFS files into live MySQL tables.

**Does importing remove rows from MySQL?** No. These Sqoop imports copy the selected data.

**Why do mapper files have different sizes?** Some partitions contain more matching records than others. Splitting by text state values can be especially uneven.

**Why did the first exercise have only 50 `device` rows?** That was the `device` table, not `accountdevice`. They are separate tables.

**Why is the import slower than expected?** Hadoop job startup overhead can dominate small training imports. More mappers can add overhead; more workers are not always faster.

**What if `--target-dir` already exists?** Choose a fresh target name or follow your instructor's safe re-run procedure. Don't blindly remove existing HDFS data.

**What if I see a password warning?** This classroom command exposes a password on the command line. For real projects, use secure credential handling.

**What if my Mac can't run this old VM?** Ask for a course-supported remote environment or compatible VM. The learning concepts are platform-independent, but setup differs.

## Screenshot index

All images are real screenshots shared during the exercise, ordered to match the walkthrough:

1. `01-databases.png` — Q1 list databases
2. `02-accounts-import.png` — Q3 accounts job result
3. `03-accounts-verified.png` — Q3 accounts HDFS files
4. `04-accountdevice-verified.png` — Q3 accountdevice HDFS files
5. `05-webpage-verified.png` — Q3 webpage HDFS files
6. `06-accounts-schema.png` — Q4 accounts `state` column
7. `07-ten-mappers.png` — Q4 ten mapper output files
8. `08-colon-separated-output.png` — Q4 output delimiter demonstration
9. `09-full-names-preview.png` — Q5 SQL preview
10. `10-full-names-hdfs.png` — Q5 imported full names
11. `11-accountdevice-schema.png` — Q6 filter column
12. `12-filtered-device-import.png` — Q6 filtered results
13. `13-service-rep-schema.png` — Q7 column names
14. `14-roger-miller-mysql.png` — Q7 SQL answer
15. `15-roger-miller-hdfs.png` — Q7 HDFS answer

### How to share on GitHub

Upload **`README.md` and the entire `screenshots/` folder together** into the same GitHub repository. GitHub automatically renders the Markdown and displays its relative screenshot links. The ZIP bundle contains this exact structure.

*Shared as a voluntary learning reference, not an official course solution. Commands and results are from an older Cloudera classroom environment; adapt carefully for other systems.*
