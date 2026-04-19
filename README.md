# Library Management System — EDA & Data Analysis

**Electronics & Telecommunication Engineering | Batch A-2**

| Member | PRN |
|---|---|
| Ayush Dumbre | 25070123029 |
| Ayush Phale | 25070123028 |
| Bidisha Chaudhuri | 25070123034 |

---

## Table of Contents

- [Project Overview](#project-overview)
- [Tech Stack](#tech-stack)
- [Setup & Installation](#setup--installation)
- [Configuration](#configuration)
- [Core System — Library Management](#core-system--library-management)
  - [Importing Libraries](#importing-libraries)
  - [Utility Functions](#utility-functions)
  - [Receipt Formatting & Email](#receipt-formatting--email)
  - [Issue Book](#issue-book)
  - [Return Book](#return-book)
  - [View Expiry Dates](#view-expiry-dates)
  - [Search](#search)
  - [Late Records & Receipts](#late-records--receipts)
  - [Analytics](#analytics)
  - [Main Menu Loop](#main-menu-loop)
- [Experiment 10 — Load & Explore Dataset with Pandas](#experiment-10--load--explore-dataset-with-pandas)
- [Experiment 11 — Categorical Data Analysis](#experiment-11--categorical-data-analysis)
- [Experiment 12 — Preprocessing & Handling Missing Values](#experiment-12--preprocessing--handling-missing-values)
- [Experiment 13 — Data Binning, Formatting & Visualization](#experiment-13--data-binning-formatting--visualization)
- [Experiment 14 — Data Normalization & Data Type Conversion](#experiment-14--data-normalization--data-type-conversion)
- [Experiment 16 — Visualization Techniques](#experiment-16--visualization-techniques)
- [Experiment 17 — Statistical Measures & Advanced Plots](#experiment-17--statistical-measures--advanced-plots)
- [Experiment 18 — Real-World & Interactive Visualization](#experiment-18--real-world--interactive-visualization)
- [Dataset Description](#dataset-description)

---

## Project Overview

This project is a fully functional **Library Management System** built and run on Google Colab. It manages student book records, tracks due dates, calculates late fees, sends email receipts, and also doubles as a data analysis sandbox.

The second half of the notebook is a series of **EDA (Exploratory Data Analysis) experiments** — from basic dataset loading all the way to interactive 3D plots and Sankey diagrams — all performed on the library's own CSV dataset.

---

## Tech Stack

| Library | Purpose |
|---|---|
| `pandas` | Data loading, manipulation, CSV I/O |
| `numpy` | Numeric operations |
| `matplotlib` | Static charts and plots |
| `seaborn` | Statistical visualizations (heatmap) |
| `plotly` | Interactive charts (treemap, 3D scatter, Sankey, radar) |
| `sklearn` | Normalization and encoding |
| `scipy` | Hierarchical clustering (dendrogram) |
| `prettytable` | CLI-friendly tabular output |
| `yagmail` | Gmail-based email automation |

---

## Setup & Installation

Run this at the top of your notebook to install all required packages:

```python
!pip install prettytable yagmail pandas matplotlib
!pip install yagmail
```

> **Why does Colab need `!pip install` at runtime?**
> Google Colab provides a fresh Python environment every session. Unlike a local machine where you install packages once and they persist, Colab resets when the runtime disconnects — so installations need to happen at the start of each session. The `!` prefix tells Colab to run the command in the system shell rather than treating it as Python code. `yagmail` appears twice in the original notebook; the second install is redundant but harmless.

For interactive visualizations (Experiment 18), you'll also need:

```python
!pip install plotly scipy scikit-learn seaborn
```

> `plotly` is not always bundled in Colab's default environment. `scipy` is needed specifically for hierarchical clustering in the dendrogram experiment. `scikit-learn` (imported as `sklearn`) provides the scalers and encoders used in preprocessing. `seaborn` is a statistical visualization library built on top of matplotlib — it handles heatmaps elegantly with very little code, and brings a cleaner visual style than raw matplotlib defaults.

---

## Configuration

Before anything runs, a few globals are set:

```python
from google.colab import userdata

LIBRARIAN_EMAIL = 'librarian232@gmail.com'
EMAIL_PASSWORD = userdata.get('YAGMAIL_PASSWORD')

LIBRARY_RECORDS_FILE = "/content/Library_records_500.csv"
LATE_SUBMISSION_FILE = "/content/LATE_submission_Records.csv"

yag = None
```

> **Why store config at the top?**
> Centralizing configuration as constants at the top of a program is standard software practice. If the librarian's email ever changes, you update one line instead of hunting through every function that uses it. The same logic applies to file paths.
>
> **Why Colab Secrets instead of hardcoding the password?**
> Hardcoding credentials in code is a serious security risk — especially on a shared platform. If you accidentally share your notebook or push it to GitHub, your Gmail password is exposed. `userdata.get('YAGMAIL_PASSWORD')` pulls the value from Colab's encrypted secrets store, which is tied to your Google account and never visible in the notebook output. It's essentially environment variable management for Colab. To set it up: go to the  **Secrets** tab on the left panel, add a key named `YAGMAIL_PASSWORD`, and paste your Gmail app password.
>
> **Why `yag = None`?**
> This declares the SMTP connection variable in the global scope before any function references it. Python requires a name to exist before it can be referenced. Setting it to `None` is a clean way to say "this will be filled in later." Functions that need it can check `if yag is None` before proceeding, preventing a `NameError`.

---

## Core System — Library Management

### Importing Libraries

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from prettytable import PrettyTable
from datetime import datetime, timedelta
import yagmail
```

> Every library here has a specific job. `pandas` is the backbone — it reads and writes CSVs and acts as the in-memory "database" for student records. `numpy` supports the numeric operations pandas relies on internally, and is directly used for things like `np.number` type filtering. `matplotlib.pyplot` generates the analytics pie chart in the menu system. `PrettyTable` formats data into readable ASCII tables in the Colab output — much cleaner than printing raw DataFrames. `datetime` and `timedelta` are the standard library's tools for date math — without them, calculating how many days overdue a book is would require painful manual string parsing. `yagmail` wraps Gmail's SMTP protocol into simple one-line Python calls.

---

### Utility Functions

These functions are the foundation the entire system is built on. Think of them as the data access layer — all file reading, writing, and email sending flows through here. Every feature calls these instead of directly touching the CSV files.

```python
def initialize_email():
    global yag
    try:
        yag = yagmail.SMTP(LIBRARIAN_EMAIL, EMAIL_PASSWORD)
        print("Email system initialized!")
        return True
    except Exception as e:
        print(f"Email setup failed: {e}")
        print("Receipts will be displayed but not emailed.")
        return False
```

> **How SMTP works:** SMTP (Simple Mail Transfer Protocol) is the protocol computers use to send email over the internet. When you call `yagmail.SMTP(email, password)`, it opens an authenticated connection to Gmail's outgoing mail servers. This connection object (`yag`) is stored globally and reused every time a receipt needs to be sent — much more efficient than reconnecting to Gmail's servers each time.
>
> **`global yag`:** Without this keyword, Python would treat `yag` inside the function as a local variable, and the assignment would disappear when the function returns. `global yag` explicitly tells Python to modify the module-level `yag` variable instead.
>
> **Graceful failure with `try/except`:** Email initialization can fail for many reasons — wrong password, no internet connection, Gmail blocking the sign-in, 2FA issues, etc. Wrapping it in `try/except` means the entire library system doesn't crash if email isn't configured. The system falls back to printing receipts to the screen and continues working normally.

```python
def load_library_records():
    try:
        df = pd.read_csv(LIBRARY_RECORDS_FILE)
        df.columns = df.columns.str.strip()
        for col in ['unique id', 'phone no.', 'book serial no.']:
            if col in df.columns:
                df[col] = df[col].astype(str)
        return df
    except:
        columns = ['unique id', 'name', 'year of study', 'department', 'book serial no.',
                   'book name', 'date of issue', 'submit date', 'phone no.', 'email id']
        return pd.DataFrame(columns=columns)

def load_late_records():
    try:
        df = pd.read_csv(LATE_SUBMISSION_FILE)
        df.columns = df.columns.str.strip()
        return df
    except:
        columns = ['unique id', 'name', 'year of study', 'department', 'book serieal no.',
                   'book name', 'submit date', 'email id', 'Net Charges']
        return pd.DataFrame(columns=columns)
```

> **The self-bootstrapping pattern:** The `try/except` does something smart here. On first run, the CSV files don't exist yet — `pd.read_csv()` throws a `FileNotFoundError`. The `except` block catches it and returns an empty DataFrame with the correct column structure. The system creates the files on the first save. Zero setup required.
>
> **`df.columns.str.strip()`:** CSV files created by humans or exported from spreadsheets often have invisible whitespace in column headers — like `" book name"` (with a leading space) instead of `"book name"`. If you try `df['book name']` later, pandas won't find it and raises a `KeyError`. Stripping whitespace from all headers defensively prevents this entire class of silent bugs.
>
> **Why cast IDs to `str`?** Pandas is clever about types — when it reads a column of numbers, it stores them as integers. That's usually helpful, but for IDs like `007` or phone numbers like `09876543210`, the leading zero gets silently dropped. Casting to string preserves the exact value as entered by the user.

```python
def save_library_records(df):
    try:
        df.to_csv(LIBRARY_RECORDS_FILE, index=False)
        return True
    except Exception as e:
        print(f" Save failed: {e}")
        return False

def save_late_records(df):
    try:
        df.to_csv(LATE_SUBMISSION_FILE, index=False)
        return True
    except Exception as e:
        print(f" Save failed: {e}")
        return False
```

> **`index=False` — why it matters:** By default, `df.to_csv()` writes the DataFrame's internal row index (0, 1, 2, 3...) as the first column of the CSV. When you reload that file, you get a strange unnamed column full of numbers that wasn't in your original data. `index=False` skips writing the index, keeping the CSV clean and "round-trip safe" — what you save is exactly what you load.
>
> **Persistence model:** This system uses CSVs as a lightweight flat-file database. Every operation loads the full file into memory as a DataFrame, modifies it, and writes it back to disk. This pattern is sometimes called "read-modify-write." It's not as performant as a real database for large datasets, but for a college project managing hundreds of records, it's simple, portable, and requires no database server.

---

### Receipt Formatting & Email

```python
def send_email_receipt(recipient_email, subject, receipt_text):
    global yag
    if yag is None:
        print("Email not initialized")
        return False
    try:
        yag.send(to=recipient_email, subject=subject, contents=receipt_text)
        print(f"Receipt emailed to {recipient_email}")
        return True
    except Exception as e:
        print(f"Email failed: {e}")
        return False
```

> This function is a thin, safe wrapper around `yag.send()`. The `if yag is None` guard prevents a cryptic `AttributeError: 'NoneType' object has no attribute 'send'` if the email system failed to initialize — instead you get a clear, human-readable message. Returning `True` or `False` lets the calling code know whether the email succeeded. This "status return" pattern is useful when you want to log outcomes or show the user a conditional success message.

```python
def format_receipt(data, receipt_type="ISSUE"):
    receipt = f"""
Name:                {data['name']}
Unique ID:           {data['unique_id']}
Book Serial No.:     {data['book_serial']}
Book Name:           {data['book_name']}
Date of Issue:       {data['date_of_issue']}
Date of Submission:  {data['submit_date']}
"""
    if receipt_type == "LATE FEE":
        receipt += f"Late Fee:            {data['late_fee']} INR\n"

    receipt += """
1) Late fees which are progressive in nature will be charged on failure of returning book on time.
2) No excuses are entertained.
"""
    return receipt
```

> **f-strings and triple-quoted strings:** Python's f-strings (formatted string literals, introduced in Python 3.6) let you embed variable values directly inside a string using `{variable_name}`. The triple-quote syntax `"""..."""` lets the string span multiple lines without explicit newline characters or string concatenation — ideal for formatted text like receipts.
>
> **Default parameter `receipt_type="ISSUE"`:** If you call `format_receipt(data)` without specifying a type, it defaults to `"ISSUE"` — a normal borrowing receipt. The late fee line is only appended when the caller explicitly passes `receipt_type="LATE FEE"`. This is more flexible than writing two separate functions, and means any future receipt types can be added with minimal new code.

---

### Issue Book

```python
def issue_book():
    df = load_library_records()

    unique_id = input("Unique ID: ").strip()
    name = input("Name: ").strip()
    year = input("Year of Study: ").strip()
    dept = input("Department: ").strip()
    phone = input("Phone: ").strip()
    email = input("Email: ").strip()
    book_serial = input("Book Serial No.: ").strip()
    book_name = input("Book Name: ").strip()

    issue_date = datetime.now().strftime('%Y-%m-%d')
    days_allowed = int(input("Days allowed (default 14): ") or 14)
    submit_date = (datetime.now() + timedelta(days=days_allowed)).strftime('%Y-%m-%d')
```

> **`datetime.now()` and `timedelta` for due dates:** `datetime.now()` returns the current date and time as a Python object. `.strftime('%Y-%m-%d')` formats it as a string: year-month-day. This format is intentional — strings in `YYYY-MM-DD` format sort correctly in both alphabetical and chronological order, which matters later when comparing dates.
>
> `timedelta(days=days_allowed)` creates a duration object. Adding it to `datetime.now()` produces a future date — automatically handling month and year rollovers that you'd have to code manually otherwise (e.g., adding 14 days to January 25 correctly gives February 8).
>
> **`input(...) or 14`:** If the user presses Enter without typing anything, `input()` returns an empty string `""`. In Python, empty strings are "falsy," so `"" or 14` evaluates to `14`. This is a concise single-line default value — without it, you'd need a multi-line if-else.

```python
    record = {
        'unique id': unique_id, 'name': name, 'year of study': year,
        'department': dept, 'book serial no.': book_serial,
        'book name': book_name, 'date of issue': issue_date,
        'submit date': submit_date, 'phone no.': phone, 'email id': email
    }

    student_exists = df[df['unique id'] == unique_id]

    if not student_exists.empty:
        idx = student_exists.index[0]
        for key, value in record.items():
            df.at[idx, key] = value
        print("Student record updated")
    else:
        df = pd.concat([df, pd.DataFrame([record])], ignore_index=True)
        print("New student record added")

    if save_library_records(df):
        receipt_data = { ... }
        receipt = format_receipt(receipt_data)
        print("\nRECEIPT:\n" + receipt)
        confirm = input("\nEmail receipt? (yes/no): ").strip().lower()
        if confirm == 'yes':
            send_email_receipt(email, f"Library Receipt - {book_name}", receipt)
```

> **Upsert logic (Update + Insert):** The check `df[df['unique id'] == unique_id]` is a boolean filter that returns rows where the ID matches. If `.empty` is True, it's a new student and we append with `pd.concat()`. If the student already exists, `df.at[idx, key] = value` updates the specific cell — `.at[]` is the most efficient way to modify a single cell by row index and column label.
>
> This "update if exists, insert if not" behavior is called an **upsert** (a portmanteau of "update" and "insert"). It's a fundamental database operation pattern that prevents duplicate records when the same student borrows multiple books over time.
>
> **Why `pd.concat()` instead of `df.append()`?** `DataFrame.append()` was deprecated in pandas 1.4 and removed in 2.0. `pd.concat([df, pd.DataFrame([record])], ignore_index=True)` is the modern replacement — it concatenates the existing DataFrame with a new single-row DataFrame created from the record dictionary, and resets the row index to stay sequential.

---

### Return Book

```python
def submit_book():
    df = load_library_records()
    late_df = load_late_records()

    search_by = input("Search by: 1=Unique ID / 2=Book Serial: ").strip()
    if search_by == '1':
        records = df[df['unique id'] == input("Unique ID: ").strip()]
    else:
        records = df[df['book serial no.'] == input("Book Serial No.: ").strip()]

    table = PrettyTable(['Row#', 'ID', 'Name', 'Book Serial', 'Book', 'Due Date'])
    for row_num, (idx, row) in enumerate(records.iterrows()):
        table.add_row([row_num, row['unique id'], row['name'][:12],
                       row['book serial no.'][:8], row['book name'][:15], row['submit date']])
    print(table)
```

> **Why use display Row# instead of the DataFrame index?** After rows have been added, deleted, and re-added over time, DataFrame indices become non-sequential (rows labeled 0, 3, 7, 12, etc.). Asking the user to type `"7"` when they see the third row on screen is confusing and error-prone. `enumerate()` assigns clean sequential display numbers starting from 0. The actual DataFrame index is tracked separately for internal operations. This is a UX decision — keep the display layer and the data layer separate.
>
> **`row['name'][:12]`:** Truncating names to 12 characters prevents long names from breaking the PrettyTable column widths. Terminal output has fixed width; overflow pushes everything out of alignment.

```python
    record_idx = records.index[row_num]
    record = records.loc[record_idx]

    actual_date = datetime.now().strftime('%Y-%m-%d')
    submit_dt = pd.to_datetime(record['submit date'])
    actual_dt = pd.to_datetime(actual_date)
    days_late = (actual_dt - submit_dt).days
    late_fee = max(0, days_late * 10)

    df = df.drop(record_idx)
    save_library_records(df)
```

> **Date arithmetic — why `pd.to_datetime()` is needed:** Date strings like `"2024-03-15"` are just text — you can't subtract two text strings to get a number of days. `pd.to_datetime()` converts the string into a proper datetime object that supports arithmetic. Subtracting two datetime objects returns a `timedelta`, and `.days` extracts the integer count of days.
>
> **`max(0, days_late * 10)`:** If the book is returned early, `days_late` is negative, which would produce a negative fine — absurd. `max(0, ...)` clamps the result at zero, so early returns generate no charge. Late returns incur ₹10 per day.
>
> **The two-file system design:** When a book is returned, the active record is deleted from `Library_records_500.csv`. If there's a late fee, a new entry is simultaneously created in `LATE_submission_Records.csv`. These are separate records — the active record is closed, and a penalty record is opened. This mirrors how real accounting systems work: an "open transaction" is settled and a "penalty invoice" is raised independently.

---

### View Expiry Dates

```python
def view_expiry():
    df = load_library_records()
    today = datetime.now()
    table_data = []

    for idx, row in df.iterrows():
        submit_dt = pd.to_datetime(row['submit date'])
        days_left = (submit_dt - today).days
        status = 'OVERDUE' if days_left < 0 else 'OK'
        table_data.append([row['name'], row['unique id'], row['book name'],
                           row['submit date'], days_left, status])

    table = PrettyTable(['Name', 'ID', 'Book', 'Due Date', 'Days Left', 'Status'])
    for row in table_data:
        table.add_row(row)
    print(table)
    print(f"\nOverdue: {len([r for r in table_data if r[4] < 0])}")
```

> **The value of proactive monitoring:** This view exists so the librarian can see upcoming and current overdue books *before* a return happens. A book returned late already has a fine. Catching it before — sending a reminder, calling the student — prevents the fine in the first place, which is better for everyone. The `days_left` column makes this actionable: a book with 2 days left warrants a reminder; one with -10 days is already a problem.
>
> **Ternary expression `'OVERDUE' if days_left < 0 else 'OK'`:** This is Python's condensed if-else for single-expression assignments. Negative `days_left` means the due date is in the past.
>
> **List comprehension for counting:** `len([r for r in table_data if r[4] < 0])` filters to rows where index 4 (days_left) is negative, then counts them. This is the Pythonic alternative to an explicit counter loop.

---

### Search

```python
def search():
    df = load_library_records()
    choice = input("1=ID 2=Name 3=Book Serial 4=Book Name: ").strip()
    term = input("Search: ").strip()

    if choice == '1':
        results = df[df['unique id'].astype(str) == term]
    elif choice == '2':
        results = df[df['name'].str.contains(term, case=False, na=False)]
    elif choice == '3':
        results = df[df['book serial no.'] == term]
    elif choice == '4':
        results = df[df['book name'].str.contains(term, case=False, na=False)]
```

> **Exact match vs. partial/fuzzy match — when to use which:**
>
> IDs and serial numbers use exact match (`==`) because they're unique identifiers — a partial match on `"10"` would incorrectly return `"100"`, `"210"`, `"510"`. You want one specific record.
>
> Names and book titles use `str.contains(term, case=False)` because users rarely remember exact spellings or capitalization. `case=False` makes it case-insensitive so "ayush", "Ayush", and "AYUSH" all return the same results. `na=False` is a defensive parameter — without it, `str.contains` raises a `ValueError` if any cell contains `NaN` (a missing value). The `na=False` argument tells it to treat NaN cells as "not a match" instead of crashing.

---

### Late Records & Receipts

```python
def view_late_records():
    df = load_late_records()
    table = PrettyTable(['ID', 'Name', 'Book', 'Date', 'Fine(INR)'])
    total_fine = 0
    for idx, row in df.iterrows():
        fine = row['Net Charges']
        total_fine += fine
        table.add_row([row['unique id'], row['name'], row['book name'],
                       row['submit date'], f"{fine}"])
    print(table)
    print(f"Total Fine Collected: {total_fine} INR")
```

> **Accumulator pattern:** Starting `total_fine = 0` and doing `total_fine += fine` on each iteration is the classic accumulator pattern — the simplest way to compute a running total. The final value after the loop is the total fine collected across all late records, useful for the librarian's financial reporting.

```python
def late_fee_receipt():
    df = load_late_records()
    uid = input("Unique ID: ").strip()
    records = df[df['unique id'] == uid]
    total_fine = records['Net Charges'].sum()
    name = records.iloc[0]['name']
    email = records.iloc[0]['email id']
```

> **`records['Net Charges'].sum()`:** Pandas column-wise aggregation — no loop needed. If a student returned three books late, all three appear in `records`, and `.sum()` adds their fines in one call. This is much faster than manual accumulation for large datasets.
>
> **`records.iloc[0]`:** `.iloc[]` selects rows by integer position (0 = first row). Used here to grab the student's name and email from the first matching record — these fields are the same across all their records, so any row works.

---

### Analytics

```python
def analytics():
    df = load_library_records()
    book_counts = df['book name'].value_counts().head(8)

    plt.figure(figsize=(10,6))
    plt.pie(book_counts.values, labels=book_counts.index, autopct='%1.1f%%')
    plt.title('Top 8 Most Issued Books')
    plt.show()

    table = PrettyTable(['Rank', 'Book Name', 'Count'])
    for i, (book, count) in enumerate(book_counts.items(), 1):
        table.add_row([i, book[:30], count])
    print(table)
```

> **`value_counts()`** returns a sorted Series where each unique value in the column is the index and its frequency is the value. It's one of the most useful single-function summaries in pandas. `.head(8)` limits to the 8 most borrowed titles — more than 8 slices on a pie chart become too thin and unreadable.
>
> **`autopct='%1.1f%%'`:** Controls the label inside each pie slice. `%1.1f` means at least 1 digit with 1 decimal place, and `%%` is an escaped literal `%` sign. Result: each slice shows something like `"23.4%"`. Without this, the pie only shows category labels with no values.
>
> **`enumerate(book_counts.items(), 1)`:** `.items()` yields `(book_name, count)` pairs. Wrapping in `enumerate(..., 1)` injects a counter starting at 1 — so you get rank numbers without a separate counter variable. Elegant and Pythonic.

---

### Main Menu Loop

```python
initialize_email()
print("System Ready!\n")

while True:
    choice = main_menu()
    if choice == '1':
        issue_book()
    elif choice == '2':
        submit_book()
    elif choice == '3':
        view_expiry()
    elif choice == '4':
        search()
    elif choice == '5':
        view_late_records()
    elif choice == '6':
        late_fee_receipt()
    elif choice == '7':
        analytics()
    elif choice == '8':
        print("Thank you!")
        break
    else:
        print(" Invalid choice!")
```

> **The event loop / dispatcher pattern:** `while True` creates an infinite loop that only exits when `break` is reached (option 8). This is the same fundamental architecture used in GUI applications, web servers, and game engines — wait for input, dispatch to the right handler, repeat. Each option calls a self-contained function that handles all its own I/O and returns control to the main loop when done. This separation of responsibilities is called a **dispatcher pattern** and makes the code easy to extend — adding a new feature just means writing a new function and adding an `elif` branch.
>
> **`initialize_email()` before the loop:** Email is set up once at startup and the connection is reused throughout. Calling it inside the loop would reconnect to Gmail's servers on every menu action — slow, wasteful, and likely to trigger rate limiting. One connection, used for the whole session.

---

## Experiment 10 — Load & Explore Dataset with Pandas

**Theory:** EDA (Exploratory Data Analysis) is always the first step in any data science project. Before building models, writing reports, or drawing conclusions, you need to understand the raw material — its shape, content, quirks, and gaps. Skipping EDA is how analysts end up with models trained on misunderstood columns, or visualizations that mislead because of unnoticed missing values. The goal here is to get a complete first impression of the dataset in as few commands as possible.

### Loading the Dataset

```python
import pandas as pd
import numpy as np

df = pd.read_csv("/content/Library_records_500.csv")
df
```

> **What `pd.read_csv()` actually does under the hood:** It reads the file line by line, treats the first row as column headers, splits each subsequent row by commas, and infers each column's data type — integers stay as `int64`, decimals become `float64`, everything else becomes `object` (pandas' term for strings). The result is a 2D labeled data structure called a **DataFrame** — essentially a spreadsheet you can manipulate with code.
>
> Writing just `df` at the end of a Colab/Jupyter cell (without `print()`) triggers the notebook's rich HTML rendering, which shows a scrollable, styled table. This is more readable than `print(df)` for inspection purposes.

### Basic Exploration

```python
print("===== DATASET HEAD =====")
print(df.head())

print("===== DATASET TAIL =====")
print(df.tail())

print("===== DATASET INFO =====")
df.info()

print("===== STATISTICAL SUMMARY =====")
print(df.describe())

print("===== DATASET SHAPE =====")
print(df.shape)

print("===== MISSING VALUES =====")
print(df.isnull().sum())
```

> These six commands together constitute a standard EDA opening:
>
> **`df.head()`** — First 5 rows. A sanity check: do the column names match what you expected? Does the data look right? Are there obvious formatting issues in the first few rows?
>
> **`df.tail()`** — Last 5 rows. This is just as important as `head()`. Some CSV exports append footer rows, summary rows, or garbage at the end. You'd never see this with `head()` alone.
>
> **`df.info()`** — Arguably the most informative single call. It shows: total row count, each column's name, the count of non-null entries, and the dtype. A column showing `500 non-null` out of 500 rows is perfectly clean. A column showing `423 non-null` has 77 missing values you need to address. A column you expected to be `int64` showing up as `object` means there are non-numeric characters hiding in it somewhere.
>
> **`df.describe()`** — Computes 8 statistics for every numeric column: count, mean, standard deviation (spread), minimum, 25th percentile (Q1), 50th percentile (median/Q2), 75th percentile (Q3), and maximum. These 8 numbers tell a lot — if the mean is much larger than the median, the data is right-skewed (a few very large values pulling the average up). If the standard deviation is huge relative to the mean, values are wildly spread.
>
> **`df.shape`** — Returns `(rows, columns)`. Always run this first — it's the ground truth for everything else. If you expected 500 records but shape shows 503, there are duplicate or extra rows.
>
> **`df.isnull().sum()`** — Counts NaN (missing) values per column. A column with many missing values needs a strategy: drop it, fill it, or investigate why it's missing. This single line is often the most consequential output of initial EDA.

### Column Operations

```python
# Add a column
df["Status"] = "Submitted"

# Drop a column
df.drop("Status", axis=1, inplace=True)

# Sort by last column descending
print(df.sort_values(by=df.columns[-1], ascending=False))

# Save
df.to_csv("processed_data.csv", index=False)
```

> **Adding a column** with `df["Status"] = "Submitted"` broadcasts a single value across all rows. This is useful for adding status flags, computed fields, or labels. If you assign a Series instead of a scalar, each row gets the corresponding Series value.
>
> **`df.drop("Status", axis=1, inplace=True)`:** `axis=1` specifies column-wise deletion (as opposed to `axis=0` for row deletion). The `axis` parameter is a common source of confusion in pandas — just remember: axis 0 = rows, axis 1 = columns. `inplace=True` modifies the DataFrame directly; without it, `drop()` returns a modified copy and the original is unchanged.
>
> **`df.columns[-1]`:** Python's negative indexing also works on pandas Index objects — `-1` always refers to the last element. Using this instead of hardcoding the column name makes the code robust to column reordering.
>
> **Saving intermediate results:** `df.to_csv("processed_data.csv", index=False)` is good practice at key points in your analysis. If the notebook crashes or you restart the runtime, you don't have to re-run all preprocessing — just reload the processed file.

### Handling Missing Values

```python
# Fill numeric missing values with mean
df.fillna(df.mean(numeric_only=True), inplace=True)

# Fill categorical missing values with mode
for col in df.select_dtypes(include='object'):
    df[col].fillna(df[col].mode()[0], inplace=True)

print("Missing values handled")
```

> **Why different strategies for numbers vs. text?** The mean is a statistically meaningful replacement for a missing number — if a student's year of study is missing, filling it with the average year (e.g., 2.4) is a defensible estimate that preserves the overall distribution. But the mean is mathematically meaningless for text — you can't average "Electronics" and "Mechanical." For text/categorical columns, the **mode** (most frequent value) is used as the best available proxy.
>
> **`df[col].mode()[0]`:** `.mode()` returns a Series of the most frequent values (there can be ties), so `[0]` selects the first one. Without `[0]`, you'd get a Series object, which `fillna()` doesn't accept as a scalar.
>
> **The broader context — imputation theory:** In machine learning, filling missing values is called **imputation**. Mean and mode are "simple imputation" strategies. More advanced methods exist: median imputation (more robust to outliers than mean), forward fill (copy the previous row's value — useful for time series), or model-based imputation (use other columns to predict the missing value). For EDA and demonstration purposes, mean/mode is standard and sufficient.

### Top Books

```python
top_books = df['book name'].value_counts().head(10)
print("===== TOP 10 BOOKS READ =====")
print(top_books)
```

> **`value_counts()` is a frequency distribution.** It answers "how many times does each unique value appear?" and returns results sorted from most to least common. In library management terms, this directly answers "which books are most in demand?" — information the librarian could use to decide which books to stock more copies of. `.head(10)` limits to the top 10 most frequent titles. This single line of pandas is equivalent to a `SELECT book_name, COUNT(*) FROM records GROUP BY book_name ORDER BY COUNT(*) DESC LIMIT 10` SQL query.

---

## Experiment 11 — Categorical Data Analysis

**Theory:** Data broadly falls into two types — **numerical** (quantities you can measure and calculate with, like age, temperature, or count) and **categorical** (labels or groups, like department names, gender, or yes/no answers). Categorical data requires different analysis techniques. You can't compute the average of "Electronics" and "Civil." Instead, you analyze frequencies, proportions, cross-tabulations, and filters. Before using categorical data in any machine learning model, it must be converted to numbers — a process called **encoding**. The choice of encoding method has significant consequences for model behavior.

### Loading and Full Summary

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

df = pd.read_csv("/content/Library_records_500.csv")

print(df.describe(include='all'))
print(df.nunique())
```

> **`describe(include='all')`:** By default, `describe()` only shows statistics for numeric columns. Adding `include='all'` extends it to object (string) columns too. For categorical columns, instead of mean/std/min/max, you get:
> - `count` — number of non-null values
> - `unique` — how many distinct values the column has
> - `top` — the single most frequent value
> - `freq` — how many times `top` appears
>
> This is incredibly useful for quick profiling. A column with `unique = 1` has only one value across all rows — useless for any analysis or prediction. A column with `unique == count` has no repeats — probably an ID column, also useless as a feature.
>
> **`df.nunique()`:** Returns the number of unique values per column as a compact one-line summary. Cross-referencing with `df.shape[0]` (total rows) quickly identifies ID columns (nearly all unique) vs. categorical grouping columns (far fewer unique values than total rows).

### Identifying Categorical Columns

```python
categorical_cols = df.select_dtypes(include='object').columns
print("===== CATEGORICAL COLUMNS =====")
print(categorical_cols)
```

> **`select_dtypes(include='object')`:** Pandas stores string and mixed-type columns with the `object` dtype. This filter returns only those columns — a dynamic, dataset-agnostic way to identify categoricals. The result is a pandas Index of column names you can loop over, pass to encoders, or use to filter the DataFrame.
>
> **Why identify them explicitly?** Many pandas and sklearn operations expect numeric input and will raise errors on strings. Explicitly separating numeric and categorical columns before applying operations is defensive programming — it prevents confusing errors and makes code intent clear.

### Label Encoding

```python
from sklearn.preprocessing import LabelEncoder

le = LabelEncoder()

for col in categorical_cols:
    df[col] = le.fit_transform(df[col])

print("===== LABEL ENCODED DATA =====")
print(df.head())
```

> **What Label Encoding does:** It maps each unique string in a column to a unique integer. For example, if `department` has values `['ECE', 'Civil', 'IT', 'Mech']`, they get mapped to integers like `[1, 0, 2, 3]` (sorted alphabetically by default). The strings are replaced with numbers in-place.
>
> **The critical limitation:** Label encoding introduces an **implicit ordinal relationship**. The numbers suggest Civil (0) < ECE (1) < IT (2) < Mech (3), implying some kind of ordering or magnitude relationship that simply doesn't exist. Linear models and neural networks will pick up on this false ordering and potentially learn wrong patterns. **Label encoding is only appropriate for tree-based algorithms** (Decision Trees, Random Forests, XGBoost), which split on threshold values and don't assume numerical relationships between categories.
>
> **`fit_transform()` in one step:** `fit()` learns the mapping from strings to integers. `transform()` applies it. `fit_transform()` does both at once. In a full ML pipeline, you'd always `fit()` on training data and `transform()` test data separately — using training data's mapping ensures consistency. Here it's combined for simplicity since we're doing EDA, not building a model.

### One-Hot Encoding

```python
df_encoded = pd.get_dummies(df, columns=categorical_cols)
print("===== ONE HOT ENCODED DATA =====")
print(df_encoded.head())
```

> **What One-Hot Encoding does:** For each categorical column with N unique values, it creates N new binary (0/1) columns — one per category. The original column is dropped. For `department` with 4 values, you'd get: `department_ECE`, `department_Civil`, `department_IT`, `department_Mech`. Each row has exactly one 1 and the rest 0 across these columns.
>
> **Why this fixes the label encoding problem:** Each category is now represented as an independent binary flag. There's no implied ordering or magnitude. The model sees "is this ECE? yes/no" as four separate unrelated questions.
>
> **The tradeoff — dimensionality explosion:** If a column has 500 unique values, one-hot encoding creates 500 new columns. This is called the **curse of dimensionality** — models struggle with extremely high-dimensional sparse data. For columns with many categories, alternatives like **target encoding** (replace category with average target value), **frequency encoding** (replace with category frequency), or **embedding layers** (in neural networks) are used. For a library dataset with a small number of departments and book categories, one-hot encoding is perfectly suitable.

### Filtering by Category

```python
col = categorical_cols[0]
filtered_df = df[df[col] == df[col].unique()[0]]
print(filtered_df)
```

> **Boolean indexing — the core pandas filtering mechanism:** `df[df[col] == value]` creates a boolean Series (a column of True/False), then uses it to select only the rows where the condition is True. This is equivalent to SQL's `WHERE` clause and is the most fundamental data filtering operation in pandas.
>
> **`df[col].unique()[0]`:** `.unique()` returns an array of all distinct values. `[0]` picks the first one. This programmatic approach avoids hardcoding a specific department name, making the code work on any similar dataset without modification.
>
> **Why filtering matters for categorical EDA:** Splitting your dataset by category and analyzing each group separately is called **stratified analysis**. It reveals group-specific patterns that disappear when you look at the whole dataset averaged together. For example, the most popular book among Electronics students might be completely different from the most popular book among Civil students.

---

## Experiment 12 — Preprocessing & Handling Missing Values

**Theory:** Data preprocessing is the most time-consuming part of real-world data science work — industry surveys consistently show data scientists spend 60–80% of their time cleaning and preparing data, not building models. Missing values are one of the most common and consequential problems. They arise from sensor failures, incomplete form submissions, pipeline errors, or simply data that was never collected. Nearly all machine learning algorithms either crash or silently produce wrong results when given NaN values, making missing value handling non-negotiable before any analysis.

### Assessing the Problem

```python
missing = df.isnull().sum()

print("===== MISSING VALUES COUNT =====")
print(missing)

print("===== MISSING VALUES PERCENTAGE =====")
print((missing / len(df)) * 100)
```

> **Count vs. percentage:** A raw count of 50 missing values means nothing without context. If your dataset has 55 rows, 50 missing is catastrophic. If it has 500,000 rows, it's negligible. Expressing missing values as a percentage of total rows provides a comparable, actionable number.
>
> **Common rules of thumb used in practice:**
> - Under ~5% missing: Safe to drop rows or fill with simple imputation
> - 5–30% missing: Fill with a statistical strategy (mean, median, mode) or model-based imputation
> - Over ~30–40% missing: Consider dropping the column entirely — it may not contain enough signal to be useful
>
> These are guidelines, not rules — the right decision depends on the nature of the data and what's being analyzed.

### Strategy 1 — Dropping Rows

```python
df_dropped = df.dropna()
print("===== DATA AFTER DROPPING MISSING VALUES =====")
print(df_dropped.head())
```

> **When to drop:** `dropna()` is the safest, cleanest option when missing data is sparse and random — you lose a few rows but the remaining data is perfectly clean and unbiased. It avoids injecting artificial values.
>
> **When NOT to drop — selection bias:** If missing values are concentrated in a specific group (e.g., only first-year students skipped a field), dropping rows removes that group disproportionately. Your analysis would then underrepresent first-year students — a form of **selection bias**. Before dropping, always investigate: is the data "missing at random" (MAR), or is missingness systematically related to a group characteristic?
>
> **A new variable, not in-place:** `df_dropped` is stored separately, leaving `df` unchanged. This deliberate pattern lets you compare the results of different strategies side-by-side.

### Strategy 2 — Imputation

```python
# Fill numeric with mean
df.fillna(df.mean(numeric_only=True), inplace=True)

# Fill categorical with mode
for col in df.select_dtypes(include='object'):
    df[col].fillna(df[col].mode()[0], inplace=True)

print("===== MISSING VALUES AFTER FILLING =====")
print(df.isnull().sum())
```

> **Imputation preserves dataset size:** Unlike `dropna()`, filling keeps all rows. This matters when every data point is valuable — medical records, rare events, or small datasets where losing rows significantly impacts statistical power.
>
> **Mean imputation subtly reduces variance:** By filling missing values with the mean, you're adding values that cluster around the center of the distribution. This artificially reduces the column's variance and can make your model slightly overconfident. For analysis purposes it's acceptable; for production ML models, more sophisticated methods are preferred.
>
> **After imputation, verify:** The final `print(df.isnull().sum())` confirms the operation worked — all counts should now be zero. Always verify preprocessing steps rather than assuming they worked correctly.

### Min-Max Normalization

```python
from sklearn.preprocessing import MinMaxScaler

scaler = MinMaxScaler()
numeric_cols = df.select_dtypes(include=np.number).columns
df[numeric_cols] = scaler.fit_transform(df[numeric_cols])

print("===== NORMALIZED DATA =====")
print(df.head())
```

> **Why normalize during preprocessing?** Many algorithms measure distances between data points (KNN, SVM, neural networks). If one column ranges 0–10,000 and another 0–4, the large-scale column dominates all distance calculations — the small-scale column effectively contributes nothing. Normalization levels the playing field so all features contribute proportionally.
>
> **Formula:** `x' = (x - x_min) / (x_max - x_min)`. Output is bounded to [0, 1].
>
> **The scaler object remembers its parameters:** After `fit_transform()`, the scaler has memorized the min and max of each column from the training data. You can call `scaler.transform(new_data)` to apply the exact same scaling to new records — critical in machine learning where you must always scale test and production data using training data's statistics, not its own.

### Removing Duplicates

```python
df.drop_duplicates(inplace=True)

print("===== DATA AFTER REMOVING DUPLICATES =====")
print(df.head())
```

> **Where duplicates come from:** They appear when the same data is imported twice, when multiple data sources overlap, or when a user submits a form twice. In a library system, a student-book combination appearing twice would make that book look twice as popular in the analytics — inflating counts for already-popular books.
>
> **What counts as a duplicate:** By default, `drop_duplicates()` requires all columns to match for a row to be considered duplicate. You can restrict it to key columns: `df.drop_duplicates(subset=['unique id', 'book serial no.'])` would catch cases where the same student borrowed the same book twice, even if the dates differ slightly.

---

## Experiment 13 — Data Binning, Formatting & Visualization

**Theory:** Raw granular data is often too detailed to show patterns clearly. Binning groups continuous values into discrete categories (like converting exact ages into age groups: "under 20", "20–30", "30–40"). A **pivot table** goes further — it cross-tabulates two categorical dimensions and aggregates a third, producing a matrix that reveals relationships invisible in the raw data. It's one of the most powerful summary tools in data analysis, forming the basis for most business intelligence reports.

### Building the Pivot Table

```python
book_col = 'book name'
dept_col = 'department'

top_books = df[book_col].value_counts().head(10).index
filtered_df = df[df[book_col].isin(top_books)]

pivot_df = filtered_df.pivot_table(index=dept_col, columns=book_col, aggfunc='size', fill_value=0)

top_depts = pivot_df.sum(axis=1).sort_values(ascending=False).head(8).index
pivot_df_small = pivot_df.loc[top_depts]

print("===== PIVOT TABLE =====")
print(pivot_df_small)
```

> **What a pivot table does structurally:** It takes a "long" dataset (one row per borrowing event) and reshapes it into a "wide" matrix where rows are departments, columns are books, and cell values are counts. This is called a **contingency table** in statistics — it shows the joint frequency distribution of two categorical variables.
>
> **`aggfunc='size'`:** Counts rows. Other useful aggregation functions: `'sum'` (add up a numeric column), `'mean'` (average), `'max'`/`'min'`. The `size` function is specifically for counting occurrences — like `COUNT(*)` in SQL.
>
> **`fill_value=0`:** Without this, department-book combinations with zero borrows get `NaN`. We replace with 0 because "never borrowed" is meaningful data (zero), not missing data.
>
> **Why filter to top 10 books and top 8 departments?** Visualizations become unreadable noise with too many categories. A bar chart with 100 book categories communicates nothing. Focusing on the most common items surfaces the most actionable insights — which books matter most, which departments are most active.

### Bar Plot

```python
pivot_df_small.plot(kind='bar', figsize=(18,7))
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
```

> **Why grouped bar charts work for this data:** Each cluster of bars represents one department. Within each cluster, different colors represent different books. This lets you simultaneously compare across departments (which is most active overall) and within departments (which books are popular there). This is a two-dimensional comparison in a single chart.
>
> **`rotation=45` and `plt.tight_layout()`:** Department names are long strings. Without rotation, they overlap and become unreadable. `ha='right'` (horizontal alignment right) combined with `rotation=45` angles them diagonally so they're readable without taking up vertical space. `plt.tight_layout()` automatically adjusts figure padding so labels aren't clipped at the edges — a small but essential call.

### Line Plot

```python
pivot_df_small.plot(figsize=(18,7))
plt.xlabel("Department")
plt.ylabel("Book Count")
plt.title("Top 10 Books vs Top Departments (Line Plot)")
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
```

> **The right and wrong use of line plots:** Lines imply continuity — they suggest values between plotted points also exist. This is meaningful for time series (Monday flows into Tuesday) but conceptually awkward for departments (there's no "halfway between Electronics and Civil"). Despite this, line plots on categorical x-axes are used in practice when the goal is visual comparison of multiple series' trends across categories — it's a pragmatic tradeoff between technical correctness and readability. If one book's line sits consistently higher than others across all departments, it's clearly the most popular library-wide.

### Histogram

```python
pivot_df_small.plot(kind='hist', figsize=(15,6))
plt.title("Distribution of Top Books (Top Departments)")
plt.tight_layout()
plt.show()
```

> **Histograms vs. bar charts — a key distinction:** Bar charts have gaps between bars (emphasizing discrete, separate categories). Histograms have no gaps (emphasizing continuous ranges divided into bins). In a histogram, each bin covers a range of values (e.g., "0–5 borrows"), and bar height shows how many observations fall in that range. The shape of the histogram is the story — a "right-skewed" distribution (most values near zero, a long tail to the right) would mean most department-book combinations see very few borrows, with a handful of very popular outliers.

### Scatter Plot

```python
if len(cols) >= 2:
    plt.figure(figsize=(10,6))
    plt.scatter(pivot_df_small[cols[0]], pivot_df_small[cols[1]])
    plt.xlabel(cols[0])
    plt.ylabel(cols[1])
    plt.title("Scatter Plot of Top Books (Top Departments)")
    plt.show()
```

> **Scatter plots reveal relationships:** Each point is a department. Its X coordinate is how many times it borrowed Book A; its Y coordinate is Book B. If departments that borrow a lot of Book A also borrow a lot of Book B, points cluster diagonally from bottom-left to top-right — **positive correlation**. If more of one means less of the other — **negative correlation** (bottom-right to top-left diagonal). Random scatter means no linear relationship. This is the visual foundation for understanding feature correlation before building regression models.

### Pie Chart

```python
dept_totals = pivot_df_small.sum(axis=1)
dept_totals.plot.pie(autopct='%1.1f%%')
plt.title("Top Departments Distribution")
plt.ylabel("")
plt.show()
```

> **`pivot_df_small.sum(axis=1)`:** Sums across columns (books) for each row (department), giving total library activity per department. `axis=1` = "sum horizontally across the columns." The result is one number per department.
>
> **When pie charts work:** Pie charts communicate part-to-whole proportions best when there are fewer than 7–8 slices and one or two slices are clearly dominant. With 8 departments, the chart is at the outer edge of readability — some slices will be thin. For a quick "which department dominates library usage?" question, a pie chart answers instantly. For precise comparison between similar-sized departments, a sorted bar chart is always clearer.

---

## Experiment 14 — Data Normalization & Data Type Conversion

**Theory:** Raw numerical features in a dataset almost always live on different scales. Year of study might range 1–4 while student ID numbers range into the thousands. Algorithms that compute distances or gradients are extremely sensitive to this — a feature with large values dominates all calculations while small-scale features contribute almost nothing. Normalization and standardization are preprocessing techniques that rescale features to a common range without altering their relative relationships. The three methods demonstrated here — Min-Max, Z-score, and Decimal Scaling — each have different properties and are appropriate in different situations.

### Min-Max Normalization

```python
from sklearn.preprocessing import MinMaxScaler

scaler = MinMaxScaler()
numeric_cols = df.select_dtypes(include=np.number).columns
df[numeric_cols] = scaler.fit_transform(df[numeric_cols])

print("===== MIN-MAX NORMALIZED DATA =====")
print(df.head())
```

> **Formula:** `x' = (x - x_min) / (x_max - x_min)`
>
> The minimum value in the column maps to exactly 0, the maximum maps to exactly 1, and everything else falls proportionally between. After normalization, all numeric columns will have values in the range [0, 1].
>
> **Best use cases:** Neural networks (especially those with sigmoid or tanh activation functions that expect inputs between 0 and 1), image processing (pixel values naturally in 0–255 range), and any algorithm where bounded output is required.
>
> **Key weakness:** Highly sensitive to outliers. If a column has values [1, 2, 3, 4, 10000], the single outlier 10000 maps to 1.0 and everything else gets compressed into a tiny slice near 0. In that case, remove or clip the outlier before applying Min-Max.

### Z-Score Standardization

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
numeric_cols = df.select_dtypes(include=np.number).columns
df[numeric_cols] = scaler.fit_transform(df[numeric_cols])

print("===== STANDARDIZED DATA =====")
print(df.head())
```

> **Formula:** `x' = (x - μ) / σ` where μ is the column mean and σ is the standard deviation.
>
> The resulting column has mean = 0 and standard deviation = 1. Values are expressed in units of "standard deviations from the mean" — so a value of `+2.0` means "2 standard deviations above average," and `-1.5` means "1.5 standard deviations below." Unlike Min-Max, there are no hard bounds.
>
> **Best use cases:** Linear regression, logistic regression, SVM, PCA, and any algorithm that assumes features are approximately Gaussian-distributed. Z-score is more robust to outliers than Min-Max because the outlier shifts the mean and standard deviation somewhat, spreading the effect across the entire scaled distribution rather than compressing everything else to near-zero.
>
> **Intuition:** A Z-score of 0 means "average." A Z-score of 3 means "extremely high — only ~0.1% of values are this high in a normal distribution." This statistical interpretability is a reason many researchers prefer Z-score over Min-Max.

### Min & Max Values Verification

```python
print("===== MIN VALUES =====")
print(df.min())

print("===== MAX VALUES =====")
print(df.max())
```

> **Always verify transformations.** After applying a scaling operation, checking the resulting min and max is a sanity check. After Min-Max scaling, every numeric column's min should be 0 and max should be 1. After Z-score scaling, values won't have fixed bounds, but the range should be centered around 0 with most values between -3 and +3. If your results don't look right, the scaler may have been applied to the wrong columns, or data types may have prevented the operation.

### Decimal Scaling

```python
df_decimal = df.copy()

for col in numeric_cols:
    max_val = df[col].abs().max()
    j = 0
    while max_val >= 1:
        max_val = max_val / 10
        j += 1
    df_decimal[col] = df[col] / (10 ** j)

print("===== DECIMAL SCALING NORMALIZED DATA =====")
print(df_decimal.head())
```

> **Formula:** `x' = x / 10^j`, where j is the smallest integer that makes `max(|x'|) < 1`.
>
> Decimal scaling simply shifts the decimal point. For a column with maximum value 9800, `j = 4` (since 9800 / 10^4 = 0.98 < 1), so every value is divided by 10,000. Unlike Min-Max, the relative spacing between values is perfectly preserved — only the scale changes, not the shape of the distribution.
>
> **`df.copy()`:** Creating a copy before modifying is critical here. Without `.copy()`, Python would create a "view" rather than an independent copy, and modifications to `df_decimal` would silently also modify `df`. This is called **chained assignment** and is a subtle, frustrating pandas bug. Always use `.copy()` when you need an independent modified version.
>
> **When is decimal scaling used?** It's less common in modern ML but appears in older literature and is conceptually simple to explain. It's useful when you want to scale down large numbers to be "less than 1" without fitting a statistical scaler object.

### Normalizing Multiple Columns Together

```python
from sklearn.preprocessing import MinMaxScaler

scaler = MinMaxScaler()
numeric_cols = df.select_dtypes(include=np.number).columns

df_multi_norm = df.copy()
df_multi_norm[numeric_cols] = scaler.fit_transform(df[numeric_cols])

print("===== NORMALIZED MULTIPLE COLUMNS (MIN-MAX) =====")
print(df_multi_norm.head())
```

> **Fitting all columns at once vs. one at a time:** Passing a DataFrame with multiple columns to `fit_transform()` applies independent scaling to each column — the scaler learns separate min/max for each. This is equivalent to calling `fit_transform()` on each column individually, but done in a single call for convenience. The scaler object then stores all the per-column parameters and can apply them consistently to new data.

### Standardizing Multiple Columns

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()

df_std = df.copy()
df_std[numeric_cols] = scaler.fit_transform(df[numeric_cols])

print("===== STANDARDIZED MULTIPLE COLUMNS =====")
print(df_std.head())
```

> **Comparing `df_multi_norm` and `df_std`:** At this point you have the same data represented in two different normalized forms. Both are valid for downstream analysis — the choice depends on the algorithm. For distance-based methods (KNN, SVM with RBF kernel) and neural networks, both work. For PCA (principal component analysis), Z-score is strongly preferred because it properly accounts for variance. For any algorithm that explicitly expects [0, 1] input, Min-Max is required. Testing both and comparing model performance is common practice.

---

## Experiment 16 — Visualization Techniques

**Theory:** "A picture is worth a thousand words" is especially true in data analysis. A well-chosen visualization can reveal a pattern in seconds that would take minutes of reading tables to discover — or might never be noticed in raw numbers at all. Each chart type is designed to answer a specific kind of question. Choosing the wrong chart type can actively mislead: a line chart on unordered categories implies trends that don't exist; a pie chart with 20 slices communicates nothing. Understanding which chart fits which question is as important as knowing how to code them.

### Bar Chart — Top Books

```python
book_counts = df['book name'].value_counts().head(10)
book_counts.plot(kind='bar', figsize=(12,6))
plt.xlabel("Book Name")
plt.ylabel("Count")
plt.title("Top 10 Books Read")
plt.xticks(rotation=45, ha='right')
plt.tight_layout()
plt.show()
```

> **Bar charts are for comparing discrete categories.** Each bar's height is directly proportional to the count, making relative comparisons immediate and accurate. The human eye is very good at judging relative bar heights — much better than judging pie slice areas, which is why bar charts are generally preferred for precise comparison.
>
> **`value_counts()` produces sorted output automatically**, so the chart naturally shows bars in descending order — tallest (most popular) on the left. This "Pareto ordering" makes it immediately obvious which books dominate and which are marginal.
>
> **`ha='right'` (horizontal alignment):** Without this, rotated labels are centered on their tick marks, causing the label text to extend to the right of the tick. `ha='right'` aligns the right end of each label to its tick mark, making angled labels read neatly toward their corresponding bar.

### Pie Chart — Book Distribution

```python
book_counts.plot.pie(autopct='%1.1f%%', figsize=(8,8))
plt.title("Top 10 Books Distribution")
plt.ylabel("")
plt.show()
```

> **Why `plt.ylabel("")`?** When you plot a pandas Series as a pie chart, matplotlib automatically uses the Series name as a y-axis label, which appears as awkward text beside the chart. Setting it to an empty string removes this artifact cleanly.
>
> **Pie vs. bar chart for the same data — when to choose which:** Both charts display the same top-10 book data. The bar chart is better when you want to compare specific books ("is Book A really more popular than Book B?") because bar heights are precisely comparable. The pie chart is better when the message is about proportions of the whole ("Book A accounts for nearly a quarter of all borrows"). Use bars for comparison; use pie for composition. When in doubt, use bars — they're almost always clearer.

### Bar Chart — Department Count

```python
dept_counts = df['department'].value_counts()
dept_counts.plot(kind='bar', figsize=(12,6))
plt.xlabel("Department")
plt.ylabel("Count")
plt.title("Department-wise Distribution")
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
```

> This chart directly answers "which department uses the library most?" — useful for institutional resource allocation. Uniform bar heights (all departments similar) would suggest equal library access across the institute. Highly uneven bars might indicate a collection skewed toward certain disciplines, or that certain departments have stronger reading cultures. Either way, the chart raises the right questions for a library administrator to investigate.

### Histogram — Numeric Distribution

```python
numeric_cols = df.select_dtypes(include=np.number).columns

if len(numeric_cols) > 0:
    df[numeric_cols[0]].plot(kind='hist', figsize=(10,5))
    plt.title(f"Distribution of {numeric_cols[0]}")
    plt.xlabel(numeric_cols[0])
    plt.ylabel("Frequency")
    plt.show()
else:
    print("No numeric column available\n")
```

> **Histograms vs. bar charts — the key visual distinction:** Bar charts have visible gaps between bars, emphasizing that categories are separate and discrete. Histograms have no gaps, emphasizing that the x-axis is a continuous range divided into bins. In a histogram, what matters is the shape: a **symmetric bell curve** suggests a normal distribution; **right skew** (long tail to the right) means most values are low but a few are very high; **bimodal** (two peaks) suggests the data contains two distinct sub-populations mixed together.
>
> **The defensive `if` check:** `if len(numeric_cols) > 0` prevents a crash if the dataset has no numeric columns at the time this runs (e.g., after encoding all categoricals to strings in a previous step). This kind of defensive guard is especially important in notebook code that runs in sequence.

### Scatter Plot

```python
if len(numeric_cols) >= 2:
    plt.figure(figsize=(8,6))
    plt.scatter(df[numeric_cols[0]], df[numeric_cols[1]])
    plt.xlabel(numeric_cols[0])
    plt.ylabel(numeric_cols[1])
    plt.title("Scatter Plot")
    plt.show()
else:
    print("Not enough numeric columns\n")
```

> **Scatter plots are for detecting relationships.** Each point is one row in your dataset, plotted at the (X, Y) coordinates of two of its feature values. The spatial pattern of points tells you about the relationship:
> - **Diagonal band (bottom-left to top-right):** Positive correlation — as X increases, Y tends to increase
> - **Diagonal band (top-left to bottom-right):** Negative correlation — as X increases, Y tends to decrease
> - **Horizontal/vertical band:** One variable has no relationship to the other
> - **Cloud of points:** No linear relationship (though nonlinear ones might exist)
>
> The scatter plot is the starting point for any correlation analysis and is what you look at before deciding whether regression makes sense.

### Color-Encoded Chart

```python
top_books = df['book name'].value_counts().head(10).index
filtered_df = df[df['book name'].isin(top_books)]
pivot_df = filtered_df.pivot_table(index='department', columns='book name', aggfunc='size', fill_value=0)

pivot_df.plot(kind='bar', figsize=(18,7))
plt.xlabel("Department")
plt.ylabel("Count")
plt.title("Top Books vs Department (Color Encoded)")
plt.xticks(rotation=45, ha='right')
plt.legend(title="Book Name", bbox_to_anchor=(1.05,1))
plt.tight_layout()
plt.show()
```

> **Visual encoding theory:** Human perception uses multiple channels to extract information from images — position (most accurate), then size, then color, then shape. In this chart, position encodes count (bar height), the x-axis position encodes department, and **color** encodes book title. Adding color brings in a third categorical dimension without making the chart three-dimensional.
>
> **`bbox_to_anchor=(1.05, 1)`:** This places the legend just outside the right edge of the plot (at x=1.05 in axes coordinates). Without explicit positioning, the legend appears inside the chart and overlaps bars. `plt.tight_layout()` then automatically resizes the figure to accommodate the external legend so it's not clipped.

---

## Experiment 17 — Statistical Measures & Advanced Plots

**Theory:** Descriptive statistics summarize a dataset's key properties in a handful of numbers. The three measures of central tendency — mean, median, and mode — tell you where the data is centered, but they reveal nothing about spread, shape, or outliers. Two datasets can have identical means yet completely different distributions. This experiment combines statistical computation with advanced plot types that reveal distributional properties that simple bar charts hide.

### Mean, Median, Mode

```python
numeric_cols = df.select_dtypes(include=np.number).columns

print("===== MEAN =====")
print(df[numeric_cols].mean())

print("===== MEDIAN =====")
print(df[numeric_cols].median())

print("===== MODE =====")
print(df.mode().iloc[0])
```

> **Mean (arithmetic average):** Sum all values, divide by count. Formula: `μ = Σx / n`. Sensitive to extreme values — one outlier can pull the mean far from where most data sits. If 99 students are in year 1 and one anomalous record says year 50, the mean would be ~1.5, misleadingly suggesting most students are in their first year but just barely.
>
> **Median (middle value):** Sort all values and take the middle one. Completely unaffected by extreme outliers. In the example above, the median is correctly 1. This is why economists report **median household income** rather than mean — billionaires would inflate the mean dramatically, making average people appear wealthier than they are.
>
> **Mode (most frequent value):** The only measure that works on categorical data. You can't average "Electronics" and "Mechanical," but you can find which appears most often. `df.mode()` can return multiple rows if values tie for most frequent; `.iloc[0]` takes the first.
>
> **Using all three together:** When mean ≈ median ≈ mode, the distribution is symmetric (likely bell-shaped). When mean > median, the distribution is right-skewed (a few very high values pull the mean up). When mean < median, it's left-skewed. Comparing mean and median is the fastest check for skewness without plotting anything.

### Area Plot

```python
pivot_df_small.plot(kind='area', stacked=False, figsize=(18,7))
plt.xlabel("Department")
plt.ylabel("Book Count")
plt.title("Area Plot (Top Books vs Departments)")
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
```

> **Area plots fill the space under a line.** The filled area emphasizes volume and magnitude visually — larger areas immediately signal higher values. With `stacked=False`, all series are drawn from the zero baseline and overlap each other. Transparency (applied automatically when series overlap) shows where multiple books share popularity across the same departments.
>
> **`stacked=True` variant:** With stacking, each series is drawn on top of the previous one. The total height at any x position represents the sum of all series — excellent for visualizing both individual contributions and the cumulative total simultaneously. Use stacked area charts when the "total over all books" is as interesting as the individual book breakdown.

### Box Plot

```python
plt.figure(figsize=(12,6))
pivot_df_small.plot(kind='box')
plt.title("Box Plot (Top Books Distribution)")
plt.tight_layout()
plt.show()
```

> **Anatomy of a box plot:**
> - **Box:** Spans from Q1 (25th percentile) to Q3 (75th percentile) — this range is called the **interquartile range (IQR)** and contains the middle 50% of values
> - **Line inside the box:** The **median (Q2)**, which tells you where the center of the data sits
> - **Whiskers:** Extend to the most extreme data points that are still within 1.5 × IQR of the box edges — covering roughly the top and bottom 24.65% in a normal distribution
> - **Points beyond whiskers:** Individual **outliers** — values that are unusually extreme relative to the rest of the data
>
> **What box plots reveal that bar charts don't:** A bar chart typically shows only one value per category (mean or total). A box plot shows the full distributional shape. Two books could have the same average borrow count but completely different distributions — one might be consistently borrowed across all departments (small IQR, compact box), while another might be popular in one department and ignored everywhere else (large IQR, tall box). The box plot makes this immediately visible.

### Heatmap

```python
import seaborn as sns

plt.figure(figsize=(12,6))
sns.heatmap(pivot_df_small, annot=True)
plt.title("Heatmap (Books vs Departments)")
plt.tight_layout()
plt.show()
```

> **Heatmaps encode numbers as color intensity.** The warmer (or darker, depending on colormap) a cell, the higher its value. This allows a human to scan an entire matrix in seconds and spot the highest-value cells — something that would take much longer reading a table of numbers.
>
> **`annot=True`:** Overlays the exact numeric value inside each cell. You get both the color signal (quick gestalt impression of the whole matrix) and the precise number (when you need exact values). Leaving `annot=False` gives a cleaner visual for large matrices where individual numbers are too small to read anyway.
>
> **Why seaborn over matplotlib here?** The equivalent in pure matplotlib requires manual calls to `plt.imshow()`, a colorbar, axis tick labels, and annotation loops. `sns.heatmap()` handles all of this in one call with sensible defaults.
>
> **Ideal use case:** Any time you have a matrix of values comparing two categorical dimensions — correlation matrices between features, confusion matrices in ML evaluation, cross-tabulations, or any pivot table — the heatmap is the best chart for revealing patterns at a glance.

### Bubble Plot

```python
if len(cols) >= 3:
    x = pivot_df_small[cols[0]]
    y = pivot_df_small[cols[1]]
    size = pivot_df_small[cols[2]] * 100

    plt.scatter(x, y, s=size)
    plt.xlabel(cols[0])
    plt.ylabel(cols[1])
    plt.title("Bubble Plot (Top Books)")
    plt.tight_layout()
    plt.show()
```

> **Bubble plots add a third dimension to scatter plots.** X position = Book A borrows, Y position = Book B borrows, bubble size = Book C borrows. Each point is a department. This allows visualizing three variables simultaneously in a 2D chart — a significant information density improvement over a standard scatter plot.
>
> **`s=size * 100`:** The `s` parameter in `plt.scatter()` controls marker size in "points squared." Raw borrow counts like 2, 5, 8 would produce dots barely visible. Multiplying by 100 scales them to clearly differentiated bubble sizes (200, 500, 800 points²). The multiplier is a visual tuning parameter — adjust it until bubbles are visually distinct without completely overlapping.
>
> **Limitation:** Bubble plots are hard to read when bubbles overlap significantly. They work best with a small number of points (departments here) where bubbles have space to breathe.

---

## Experiment 18 — Real-World & Interactive Visualization

**Theory:** Static matplotlib charts are ideal for reports, papers, and printed materials. But for data exploration — when you want to investigate what's driving a pattern, zoom into a region, or see exact values by hovering — interactive visualizations are far more powerful. Plotly is Python's leading interactive visualization library; it generates HTML-based charts that run in any browser, support zooming, panning, hovering for tooltips, and clicking to filter. This experiment also introduces hierarchical clustering — an unsupervised machine learning technique that groups similar items together based purely on their data values.

### Setup — Pivot Table

```python
book_col = 'book name'
dept_col = 'department'

top_books = df[book_col].value_counts().head(10).index
filtered_df = df[df[book_col].isin(top_books)]

pivot_df = filtered_df.pivot_table(index=dept_col, columns=book_col, aggfunc='size', fill_value=0)

top_depts = pivot_df.sum(axis=1).sort_values(ascending=False).head(8).index
pivot_df_small = pivot_df.loc[top_depts]

print("===== FINAL DATA =====")
print(pivot_df_small)
```

> This pivot table is the shared foundation for all five advanced visualizations in this experiment. Preparing data once and reusing it across multiple chart types is good practice — it ensures all charts tell consistent stories about the same underlying data, and avoids subtle inconsistencies from independently preprocessing data five times.

### Treemap (Plotly)

```python
import plotly.express as px

df_melt = pivot_df_small.reset_index().melt(id_vars='department', var_name='book name', value_name='count')

fig = px.treemap(df_melt,
                 path=['department', 'book name'],
                 values='count',
                 title="Treemap (Books within Departments)")
fig.show()
```

> **What a treemap shows:** Hierarchical data as nested rectangles. The outer rectangles represent departments, sized proportionally to their total borrow activity. Inside each department rectangle are books, again sized by borrow count. You see two levels of the hierarchy simultaneously, and the area of each rectangle is proportional to its value — making comparison across levels intuitive.
>
> **Why `.melt()` is necessary:** `px.treemap()` expects "long format" data — one observation per row. Our pivot table is "wide format" — one row per department, book counts spread across many columns. `.melt(id_vars='department')` unpivots the table: every department-book combination becomes its own row with columns `['department', 'book name', 'count']`. This "wide to long" transformation is called **melting** (the reverse, long to wide, is pivoting). These two transformations are the bread and butter of data reshaping.
>
> **Interactive features:** In Plotly's treemap, you can click on a department rectangle to zoom in and see only that department's books filling the entire canvas — a drill-down interaction. Hovering shows the exact count. This makes the treemap significantly more useful for exploration than a static version.

### Dendrogram (Hierarchical Clustering)

```python
from scipy.cluster.hierarchy import dendrogram, linkage

linked = linkage(pivot_df_small, method='ward')

plt.figure(figsize=(12,6))
dendrogram(linked, labels=pivot_df_small.index.tolist())
plt.title("Hierarchical Clustering of Departments")
plt.xlabel("Department")
plt.ylabel("Distance")
plt.tight_layout()
plt.show()
```

> **What hierarchical clustering does:** It starts with every department as its own cluster. Then it iteratively merges the two most similar clusters until only one remains. The entire merge history is recorded as a tree structure — the **dendrogram**. Leaves are individual departments; internal nodes are merge events; the height of each merge on the y-axis represents how different the two merged groups were.
>
> **`method='ward'`:** Ward's linkage minimizes the total within-cluster variance at each merge. It tends to produce balanced, compact clusters and is generally considered the best default linkage method. Other options include `single` (nearest neighbor), `complete` (farthest neighbor), and `average` (mean pairwise distance) — each produces different cluster shapes and may be preferable depending on the data structure.
>
> **Reading the dendrogram for insight:** Departments that merge at a low y-value (short vertical stem) have very similar borrowing patterns. Departments that only merge at a high y-value are very different. Drawing a horizontal "cut line" at a given height determines how many clusters you extract — everything below the cut that hasn't merged becomes a separate cluster. Knowing which departments cluster together could inform collection development, shared reading lists, or inter-departmental collaboration.

### Sankey Diagram (Plotly)

```python
import plotly.graph_objects as go

labels = list(pivot_df_small.index) + list(pivot_df_small.columns)

sources, targets, values = [], [], []

for i, dept in enumerate(pivot_df_small.index):
    for j, book in enumerate(pivot_df_small.columns):
        val = pivot_df_small.iloc[i, j]
        if val > 0:
            sources.append(i)
            targets.append(len(pivot_df_small.index) + j)
            values.append(val)

fig = go.Figure(data=[go.Sankey(
    node=dict(label=labels),
    link=dict(source=sources, target=targets, value=values)
)])
fig.show()
```

> **What a Sankey diagram shows:** Flow between nodes. Departments on the left connect to books on the right via bands — the width of each band is proportional to how many times that department borrowed that book. It's visually striking for showing how a "resource" (library borrowing activity) distributes across categories.
>
> **The data structure Plotly needs:** `sources`, `targets`, and `values` are parallel lists — the i-th element of each corresponds to one flow. All nodes (departments and books) share a single `labels` list, with departments at indices 0 through N-1 and books at indices N onward. The nested loop iterates every department-book combination; `if val > 0` skips zero-count pairs to keep the diagram uncluttered.
>
> **Interactive features:** Hovering over a node shows the total flow through it. Hovering over a link shows the exact flow value between that department and book. You can drag nodes vertically to rearrange the layout and untangle crossing flows, making complex diagrams much more readable.

### 3D Scatter Plot (Plotly)

```python
if len(cols) >= 3:
    df_plot = pivot_df_small.reset_index()
    fig = px.scatter_3d(df_plot,
                        x=cols[0], y=cols[1], z=cols[2],
                        color='department',
                        title="3D Scatter Plot")
    fig.show()
```

> **Why go 3D?** Sometimes two variables aren't sufficient to reveal the structure in your data. Points that appear to overlap in a 2D scatter plot might be clearly separated in 3D. Here, three book-count dimensions define a 3D space and each department is a point — departments with similar reading patterns for all three books cluster together in 3D.
>
> **Interactive rotation is what makes 3D plots valuable:** A static screenshot of a 3D scatter is almost always misleading because depth is collapsed onto the image plane, making spatially separated points appear overlapping. Plotly's 3D chart lets you click-drag to rotate, zoom, and inspect the cloud from any angle. This interactivity is precisely why 3D plots belong in notebooks rather than printed reports.
>
> **`color='department'`:** Plotly Express automatically assigns each unique value a color from a built-in palette and generates a legend — no additional code needed. Color adds a fourth dimension (categorical identity) to the 3D spatial dimensions.

### Radar Chart (Spider Plot)

```python
import plotly.graph_objects as go

categories = list(pivot_df_small.columns)
dept = pivot_df_small.index[0]
values = pivot_df_small.loc[dept].values.tolist()

fig = go.Figure()
fig.add_trace(go.Scatterpolar(
    r=values,
    theta=categories,
    fill='toself',
    name=dept
))
fig.update_layout(title="Radar Chart (Department vs Books)")
fig.show()
```

> **What a radar (spider/web) chart shows:** Multiple variables mapped onto axes radiating outward from a central point, each at equal angular spacing. Here, each axis represents a book title, and the value plotted on that axis is the borrow count for that department. Connecting all the plotted points creates a polygon — the shape is the "fingerprint" of that department's reading preferences.
>
> **Reading radar charts:** A department that borrows all books equally would produce a nearly circular polygon. A department with strong preferences creates an irregular polygon with spikes toward preferred books and inward indentations toward rarely-borrowed ones.
>
> **`fill='toself'`:** Fills the interior of the polygon with a semi-transparent color, making the shape much easier to perceive at a glance than an unfilled perimeter line.
>
> **Adding multiple traces for comparison:** Call `fig.add_trace(go.Scatterpolar(...))` again with different department data to overlay two departments on the same chart. Where the polygons diverge sharply, the departments have very different preferences for that book. Where they overlap, they read similarly. This is the primary use case for radar charts — comparing multi-dimensional profiles between groups.
>
> **Known limitations:** Radar charts can distort perception because the area enclosed by the polygon grows as the square of the radius, so a small increase in one variable visually expands the area disproportionately. They're also sensitive to the angular ordering of variables — reordering the axes changes the visual shape even with identical data. Best used with 4–8 variables and 2–3 overlaid groups.

---

## Dataset Description

The dataset `Library_records_500.csv` contains 500 student book borrowing records with these columns:

| Column | Description |
|---|---|
| `unique id` | Student's unique identifier |
| `name` | Student's full name |
| `year of study` | Current year (1–4) |
| `department` | Engineering department |
| `book serial no.` | Unique book identifier |
| `book name` | Title of the borrowed book |
| `date of issue` | Date the book was borrowed |
| `submit date` | Expected return date |
| `phone no.` | Student's contact number |
| `email id` | Student's email for receipts |

The late records file (`LATE_submission_Records.csv`) stores the same fields plus a `Net Charges` column — the fine in INR, calculated at ₹10 per day past the due date.

---
