Response :

You’re hitting `ORA-00907: missing right parenthesis` because the LLM produced a **truncated / malformed subquery**:

```
... JOIN (SELECT SYMBOL_ID, ROW_NUMBER() OVER (PARTITION BY SYMBOL_ID ORDER BY total_value DESC;
                                                  ↑ window clause never closed
                                         ↑ subquery never closed
```

So Oracle is right: there’s an unmatched `(` and the subquery never ends.

Let’s fix this fast so your “generate → execute” loop is stable:

---

## 1) Make the generator refuse malformed SQL and auto-fallback

Add 3 tiny guards to `sql_generator.py`:

* **Parenthesis balance** check
* **Basic shape** check (`SELECT … FROM …`)
* **Ban subqueries for now** (no `(SELECT …` inside) — we can re-enable later once we add a repair step

> If any of these checks fail, we immediately use your **rule-based fallback** which always returns a clean single `SELECT`.

Drop these helpers somewhere near your other utils:

```python
def _balanced_parens(s: str) -> bool:
    depth = 0
    for ch in s:
        if ch == "(":
            depth += 1
        elif ch == ")":
            depth -= 1
            if depth < 0:
                return False
    return depth == 0

def _has_required_clauses(sql: str) -> bool:
    low = sql.lower()
    return (" select " in f" {low} ") and (" from " in f" {low} ")

def _has_subquery(sql: str) -> bool:
    # very simple: any "(select" indicates a subquery
    return bool(re.search(r"\(\s*select\b", sql, flags=re.IGNORECASE))
```

Then, inside `SQLGenerator._llm_sql()` **after** you sanitize and inject dates, but **before** returning:

```python
# Existing steps: candidate -> _sanitize_sql -> _looks_like_select check -> date inject -> strip literals -> ensure order/limit

# Hard safety: reject malformed or complex shapes (for now)
if not _has_required_clauses(sql) or not _balanced_parens(sql) or _has_subquery(sql):
    # force fallback until we add a repair step
    return self.fallback_rule_based({
        "query": context.get("query", ""),
        "relevant_tables": context.get("relevant_tables", []),
        "columns_by_table": context.get("columns_by_table", {}),
        "main_table": (context.get("main_table") or (context.get("relevant_tables") or ["TRADE_DATA"])[0])
    })
```

This alone will stop the broken `(SELECT … ROW_NUMBER() …` outputs from ever reaching Oracle.

---

## 2) Make sure we pick a **real date column** like `ENTRY_DATETIME`

Your trade tables show date columns: `ENTRY_DATETIME`, `INITIAL_DATE`, `TERMINATION_DATE`. Your date-col picker already prefers likely names, but let’s bias harder to `ENTRY_DATETIME` if present:

```python
LIKELY_DATE_NAMES = [
    "entry_datetime", "trade_date", "txn_date", "transaction_date", "order_date",
    "initial_date", "termination_date", "date", "business_date", "timestamp", "created_at"
]
```

(If not already exactly like this, include `entry_datetime` at the front. That way, injected filter becomes `t.ENTRY_DATETIME BETWEEN :from_date AND :to_date`.)

---

## 3) Improve executor error messages (optional but helpful)

If Oracle still errors, you’ll see *which* SQL ran. In `executor.execute_select` you already log it; that’s fine. You don’t need to change anything here for the 00907 case—guard #1 will prevent those from being executed.

---

## 4) Quick re-run sanity check

With those guards, for the same NL query the generator will **fallback** and return a simple, valid SELECT (using the **single** chosen fact table + date filter), e.g. something like:

```sql
SELECT i.instrument_name AS instrument,
       t.instrument_id   AS instrument_id,
       SUM(t.trade_volume)                 AS total_volume,
       SUM(t.trade_volume * t.trade_price) AS total_value
FROM ATS.TRADES21MAY25 t
LEFT JOIN INSTRUMENTS i ON i.instrument_id = t.instrument_id
WHERE t.ENTRY_DATETIME BETWEEN :from_date AND :to_date
GROUP BY i.instrument_name, t.instrument_id
ORDER BY total_value DESC
FETCH FIRST 1000 ROWS ONLY;
```

(Your actual column names may differ; the fallback uses what it can infer. The key is: **no subqueries, balanced parentheses, and bind params present**.)

---

## 5) Why this fixes your exact error right now

* The malformed `JOIN (SELECT … ROW_NUMBER() …` never passes our new checks → we use the rule-based query instead → Oracle executes fine.
* Your executor’s new **bind filtering** already avoids `DPY-4008` when binds are missing, so you won’t get that again.

---

## 6) Next step (when you’re ready)

Once smoke is stable, we can:

* add a **repair** step (LLM “fix this SQL given ORA-009xx” prompt)
* re-enable **subqueries/window functions** safely
* add a **partition router** for `ATS.TRADES_*` to build a UNION ALL across the exact date range (so you don’t depend on one day’s table)

But for now, the three guards above will stop the `ORA-00907` and let you proceed to the reporting step.


-----------------------------

Error :
```
(doc-rag) PS D:\OneDrive - Infotech (Private) Limited\Desktop\Infotech Group\October\RAG\RAG\src> python .\test_execution.py

SQL to execute:
 SELECT SYMBOL_ID AS instrument, TRUNC(ENTRY_DATETIME) AS trade_date, SUM(PRICE * VOLUME) AS total_value FROM ATS.TRADES21MAY25 t JOIN (SELECT SYMBOL_ID, ROW_NUMBER() OVER (PARTITION BY SYMBOL_ID ORDER BY total_value DESC;
WARNING:root:SQL has no bind placeholders; ignoring provided params.
Traceback (most recent call last):
  File "D:\OneDrive - Infotech (Private) Limited\Desktop\Infotech Group\October\RAG\RAG\src\test_execution.py", line 62, in <module>
    main()
    ~~~~^^
  File "D:\OneDrive - Infotech (Private) Limited\Desktop\Infotech Group\October\RAG\RAG\src\test_execution.py", line 52, in main
    rows = SQLExecutor().run_query(
        sql, params=params,
        mode="oracle", user=ORACLE_USER, password=ORACLE_PASSWORD, dsn=ORACLE_DSN,
        max_rows=2000, timeout_seconds=60
    )
  File "D:\OneDrive - Infotech (Private) Limited\Desktop\Infotech Group\October\RAG\RAG\src\executor.py", line 549, in run_query
    return fetch_all(
        sql_text,
    ...<6 lines>...
        dsn=dsn,
    )
  File "D:\OneDrive - Infotech (Private) Limited\Desktop\Infotech Group\October\RAG\RAG\src\executor.py", line 500, in fetch_all
    return list(
        execute_select(
    ...<9 lines>...
        )
    )
  File "D:\OneDrive - Infotech (Private) Limited\Desktop\Infotech Group\October\RAG\RAG\src\executor.py", line 463, in execute_select
    cur.execute(sanitized, params)
    ~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^
  File "D:\OneDrive - Infotech (Private) Limited\Desktop\Infotech Group\October\RAG\doc-rag\Lib\site-packages\oracledb\cursor.py", line 844, in execute
    impl.execute(self)
    ~~~~~~~~~~~~^^^^^^
  File "src/oracledb/impl/thin/cursor.pyx", line 277, in oracledb.thin_impl.ThinCursorImpl.execute
  File "src/oracledb/impl/thin/protocol.pyx", line 501, in oracledb.thin_impl.Protocol._process_single_message
  File "src/oracledb/impl/thin/protocol.pyx", line 502, in oracledb.thin_impl.Protocol._process_single_message
  File "src/oracledb/impl/thin/protocol.pyx", line 494, in oracledb.thin_impl.Protocol._process_message
  File "src/oracledb/impl/thin/messages/base.pyx", line 102, in oracledb.thin_impl.Message._check_and_raise_exception
oracledb.exceptions.DatabaseError: ORA-00907: missing right parenthesis
Help: https://docs.oracle.com/error-help/db/ora-00907/

```