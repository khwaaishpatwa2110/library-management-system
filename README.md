# ARCANA — Library Management System

> A fully browser-based, real-time library management platform built as a **Database Management Systems (DBMS) practical project**. No server. No installation. Open the HTML file and your SQLite database starts instantly.

### 🌐 [Live Demo → khwaaishpatwa2110.github.io/library-management-system](https://khwaaishpatwa2110.github.io/library-management-system/)

---

## 📚 Overview

**ARCANA** is a single-file web application that simulates the complete operations of a library — managing books, authors, genres, members, borrowing, returns, overdue tracking, and fines — all backed by a real **SQLite database running entirely in the browser** via [SQL.js](https://sql-wasm.netlify.app/).

Every action you take (adding a book, issuing a loan, processing a return) directly executes SQL against the live database. Your data **persists across sessions** via `localStorage` — close the tab, reopen it, your records are still there.

---

## ✨ Features

| Module | What It Does |
|---|---|
| **Dashboard** | Live stats (total books, active members, active loans, overdue count), recent books panel, recent borrowings panel, genre stock overview via `GROUP BY` |
| **Books** | Add / remove books linked to authors and genres via `FOREIGN KEY`. Tracks `available_copies` separately from `total_copies`. `ISBN` is `UNIQUE` |
| **Authors** | Add authors with nationality and birth year. Table shows book count per author via `LEFT JOIN + COUNT` |
| **Genres** | Add genre classifications. Shows book count per genre via `LEFT JOIN` |
| **Members** | Register members with type (Standard / Premium / Student / Senior). `UNIQUE` constraint on email. Suspend / delete members |
| **Borrow a Book** | Issue books to members — `available_copies` decrements by 1 instantly via `UPDATE`. Checks availability before issuing |
| **Returns** | Process returns with optional fine entry — `available_copies` restored by 1 via `UPDATE`. Return history with days held |
| **Overdue** | Live query using `julianday(date('now'))` to find all unreturned loans past due date, with ₹2/day estimated fine |
| **SQL Terminal** | Full raw SQL shell — run any `DDL`, `DML`, `DQL`, `JOIN`, or aggregate query with formatted table output |
| **DB Schema** | ASCII entity-relationship overview of all 5 tables and their FK connections |

---

## 🗄️ Database Design

### Tables (5 total)

```
Authors          (author_id PK, name UNIQUE, nationality, birth_year)
    │
    └──(1:N)──▶ Books        (book_id PK, title, author_id FK, genre_id FK,
                               isbn UNIQUE, published_year, total_copies,
                               available_copies CHECK(>=0), shelf_location)
                    │
                    └──(1:N)──▶ Loans   (loan_id PK, book_id FK, member_id FK,
                                          borrow_date, due_date, return_date,
                                          fine_amount CHECK(>=0), status)

Genres           (genre_id PK, name UNIQUE, description)
    │
    └──(1:N)──▶ Books (via genre_id FK)

Members          (member_id PK, full_name, email UNIQUE, phone,
                  membership_type, join_date, status)
    │
    └──(1:N)──▶ Loans (via member_id FK)
```

### Constraints Used

| Type | Example |
|---|---|
| `PRIMARY KEY AUTOINCREMENT` | Every table |
| `FOREIGN KEY` | `Books.author_id → Authors`, `Loans.book_id → Books`, `Loans.member_id → Members` |
| `UNIQUE` | `Authors.name`, `Books.isbn`, `Members.email`, `Genres.name` |
| `NOT NULL` | `Books.title`, `Members.full_name`, `Loans.borrow_date` |
| `CHECK` | `Books.available_copies >= 0`, `Loans.fine_amount >= 0`, membership_type enum, status enum |
| `DEFAULT` | `Loans.status = 'Active'`, `Members.status = 'Active'`, `date('now')` on join/borrow dates |

### Normalisation

The schema satisfies **Third Normal Form (3NF)**:
- **1NF** — All attributes are atomic; no repeating groups or multi-valued fields
- **2NF** — All non-key attributes fully depend on the whole primary key (all PKs are single-column)
- **3NF** — No transitive dependencies; author details are never repeated in Books — only `author_id FK` is stored

---

## 🧑‍💻 SQL Operations Demonstrated

### DDL — Data Definition Language
```sql
CREATE TABLE Books (
  book_id          INTEGER PRIMARY KEY AUTOINCREMENT,
  title            TEXT    NOT NULL,
  author_id        INTEGER REFERENCES Authors(author_id),
  genre_id         INTEGER REFERENCES Genres(genre_id),
  isbn             TEXT    UNIQUE,
  published_year   INTEGER,
  total_copies     INTEGER DEFAULT 1 CHECK(total_copies >= 1),
  available_copies INTEGER DEFAULT 1 CHECK(available_copies >= 0),
  shelf_location   TEXT
);

CREATE TABLE Loans (
  loan_id     INTEGER PRIMARY KEY AUTOINCREMENT,
  book_id     INTEGER NOT NULL REFERENCES Books(book_id),
  member_id   INTEGER NOT NULL REFERENCES Members(member_id),
  borrow_date DATE    NOT NULL DEFAULT (date('now')),
  due_date    DATE    NOT NULL,
  return_date DATE,
  fine_amount REAL    DEFAULT 0 CHECK(fine_amount >= 0),
  status      TEXT    DEFAULT 'Active'
              CHECK(status IN ('Active','Returned','Overdue'))
);
```

### DML — Data Manipulation Language
```sql
-- INSERT a new book
INSERT INTO Books (title, author_id, genre_id, isbn, total_copies, available_copies)
VALUES ('Beloved', 3, 2, '978-1-4000-3341-6', 2, 2);

-- UPDATE — decrement copies when a book is borrowed (real-time)
UPDATE Books
SET available_copies = available_copies - 1
WHERE book_id = 1;

-- UPDATE — process a return with fine
UPDATE Loans
SET return_date = '2025-04-10',
    fine_amount = 20,
    status      = 'Returned'
WHERE loan_id = 3;

-- UPDATE — restore copies on return
UPDATE Books
SET available_copies = available_copies + 1
WHERE book_id = 1;

-- DELETE a member
DELETE FROM Members WHERE member_id = 4;
```

### DQL — Data Query Language
```sql
-- SELECT available books ordered by title
SELECT title, available_copies, shelf_location
FROM Books
WHERE available_copies > 0
ORDER BY title ASC;

-- Overdue detection using SQLite date functions
SELECT loan_id, due_date,
       CAST(julianday(date('now')) - julianday(due_date) AS INT) AS days_overdue
FROM Loans
WHERE status IN ('Active', 'Overdue')
  AND due_date < date('now')
ORDER BY days_overdue DESC;

-- GROUP BY + HAVING — genres with more than one book
SELECT genre_id, COUNT(*) AS total_books
FROM Books
GROUP BY genre_id
HAVING total_books > 1;
```

### JOIN Queries
```sql
-- Most borrowed books — 3-table LEFT JOIN + aggregate
SELECT b.title,
       a.name                AS Author,
       COUNT(l.loan_id)      AS Times_Borrowed,
       b.available_copies    AS Available
FROM Books b
LEFT JOIN Authors a ON b.author_id = a.author_id
LEFT JOIN Loans  l ON b.book_id   = l.book_id
GROUP BY b.book_id
ORDER BY Times_Borrowed DESC;

-- Member borrowing summary
SELECT m.full_name,
       m.membership_type,
       COUNT(l.loan_id)                              AS Total_Loans,
       COUNT(CASE WHEN l.status='Active' THEN 1 END) AS Active,
       COALESCE(SUM(l.fine_amount), 0)               AS Total_Fines
FROM Members m
LEFT JOIN Loans l ON m.member_id = l.member_id
GROUP BY m.member_id
ORDER BY Total_Loans DESC;
```

### Subquery
```sql
-- Members who have at least one overdue book
SELECT full_name, membership_type
FROM Members
WHERE member_id IN (
  SELECT DISTINCT member_id
  FROM Loans
  WHERE due_date < date('now')
    AND status = 'Active'
);
```

---

## 💾 Data Persistence

ARCANA saves your database to `localStorage` automatically after every change — add a book, close the tab, reopen it, the book is still there.

| Event | What Happens |
|---|---|
| First open | Fresh database with seed data, saved to `localStorage` |
| Every INSERT / UPDATE / DELETE | `db.export()` → Base64 → `localStorage` |
| Reopen the file | Reads from `localStorage`, restores full database |
| Status bar shows | **"arcana_library.db · Restored ✓"** when loaded from storage |
| **⚠ Reset DB** button | Clears `localStorage` and reloads with default seed data |

> **Note:** Data is tied to the browser + file path. Opening the file in a different browser or from a different folder will start fresh.

---

## 🚀 Getting Started

**Option 1 — Use the live demo (no download needed):**

🔗 **[https://khwaaishpatwa2110.github.io/library-management-system/](https://khwaaishpatwa2110.github.io/library-management-system/)**

**Option 2 — Run locally:**

```bash
# 1. Clone or download the repository
git clone https://github.com/khwaaishpatwa2110/library-management-system.git

# 2. Open the file in any modern browser
open library_management.html
# or just double-click the file
```

The SQLite engine loads via CDN and the database initialises automatically.

### Browser Compatibility

| Browser | Status |
|---|---|
| Chrome / Edge | ✅ Fully supported |
| Firefox | ✅ Fully supported |
| Safari | ✅ Fully supported |
| Mobile browsers | ✅ Responsive layout |

---

## 🧪 SQL Terminal

The SQL Terminal (sidebar → **⌥ SQL Terminal**) lets you run arbitrary queries live. Three built-in sample queries are included:

| Sample | Query Type |
|---|---|
| **Popular Books** | `LEFT JOIN` across Books → Authors → Loans with borrow count |
| **Member Stats** | `LEFT JOIN + CASE WHEN` aggregate for loans and fines per member |
| **Overdue Query** | `JOIN + date()` functions to list overdue loans with fine estimation |

Use **Ctrl + Enter** to run a query quickly.

---

## 📋 Test Cases

| # | Test Case | Expected | Result |
|---|---|---|---|
| 01 | Insert valid book | Row added | ✅ PASS |
| 02 | Insert duplicate ISBN | UNIQUE violation | ✅ PASS |
| 03 | Issue book — copies decrement | `available_copies - 1` | ✅ PASS |
| 04 | Return book — copies restore | `available_copies + 1` | ✅ PASS |
| 05 | Issue with no copies available | Error shown | ✅ PASS |
| 06 | Overdue query using `date('now')` | Loans listed correctly | ✅ PASS |
| 07 | Fine calculation ₹2/day | Correct amount | ✅ PASS |
| 08 | Delete member with active loans | DB handles FK | ✅ PASS |
| 09 | CHECK violation: `fine < 0` | Constraint error | ✅ PASS |
| 10 | 3-Table JOIN: Books + Authors + Loans | Correct results | ✅ PASS |

---

## 🗂️ Project Structure

```
arcana-library-system/
└── library_management.html    # Entire application — HTML + CSS + JS + SQL schema
```

Everything is intentionally in a single self-contained file for maximum portability.

---

## 🛠️ Tech Stack

| Technology | Purpose |
|---|---|
| **HTML5 / CSS3 / Vanilla JS** | Frontend — no frameworks |
| **SQL.js v1.10.2** | SQLite compiled to WebAssembly, runs in browser |
| **localStorage** | Cross-session data persistence |
| **Playfair Display** | Display / heading typeface |
| **Crimson Pro** | Body typeface |
| **JetBrains Mono** | Code / data display |

---

## 📋 Seed Data

The application comes pre-loaded with sample data for demonstration:

- **6 Authors** — García Márquez, Dostoevsky, Toni Morrison, Murakami, Adichie, Tolstoy
- **5 Genres** — Magical Realism, Literary Fiction, Psychological Thriller, Contemporary Fiction, Classic Literature
- **5 Members** — with Standard, Premium, Student, and Senior membership types
- **10 Books** — spanning all 6 authors and all 5 genres, with shelf locations
- **7 Loan Records** — mix of Active, Returned, and Overdue statuses for a realistic demo

---

## 📚 Academic Context

This project was built as part of a **2nd Semester DBMS Practical** submission, demonstrating:

- ✅ Real-time application directly connected to a database
- ✅ Proper database schema with tables, relationships, and constraints
- ✅ SQL operations: DDL, DML, DQL, JOINs, Aggregates, Subqueries
- ✅ Data integrity, consistency, and 3NF normalisation
- ✅ Features: insertion, updating, deletion, retrieval
- ✅ Real-time effects: `available_copies` updates live on every borrow and return
- ✅ Date arithmetic: `julianday()` for overdue detection and fine calculation

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

---

<div align="center">

**ARCANA Library Management System**  
Built with SQL.js · Runs entirely in the browser · No server required

</div>
