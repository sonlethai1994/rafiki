# PostgreSQL — Expert Best Practices

## Mental model

PostgreSQL is a relational database — it excels at set-based operations. The query planner decides how to execute your SQL. Your job is to give it accurate statistics, the right indexes, and queries that don't fight it. When something is slow, `EXPLAIN ANALYZE` is always the first tool to reach for.

---

## Expert rules

### Indexes

**1. Add indexes for every FK and every field used in `WHERE`, `ORDER BY`, or `JOIN`. Django does not create FK indexes by default in all cases.**

```python
class Task(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, db_index=True)
    status = models.CharField(max_length=20, db_index=True)
```

**2. Use partial indexes for queries on a subset of rows — smaller index, faster lookup.**

```python
from django.db.models import Index, Q

class Meta:
    indexes = [
        Index(
            fields=["user", "due_date"],
            condition=Q(status="pending"),
            name="pending_tasks_user_due_idx",
        )
    ]
```

**3. Use `GinIndex` for full-text search and `JSONField` queries.**

```python
from django.contrib.postgres.indexes import GinIndex
from django.contrib.postgres.search import SearchVectorField

class Meta:
    indexes = [GinIndex(fields=["search_vector"])]
```

**4. Use composite indexes (multi-column) when queries filter on multiple fields together — order matters (most selective column first).**

```python
indexes = [
    Index(fields=["user", "status", "due_date"], name="task_user_status_due_idx")
]
```

---

### Query analysis

**5. Use `EXPLAIN ANALYZE` to understand what PostgreSQL actually does.**

```sql
EXPLAIN ANALYZE
SELECT * FROM tasks_task
WHERE user_id = 42 AND status = 'pending'
ORDER BY due_date;
```

Look for:
- `Seq Scan` on large tables → missing index
- `Nested Loop` with high row estimates → statistics need refresh (`ANALYZE`)
- `Sort` with high cost → consider index on sort column

**6. In Django, use `.query` to see the generated SQL.**

```python
qs = Task.objects.filter(user=user, status="pending")
print(str(qs.query))
```

**7. Use `django-debug-toolbar` locally — it shows every query, its duration, and whether it's a duplicate.**

---

### N+1 queries

**8. N+1 is the most common Django performance bug. Detect with `assertNumQueries` in tests.**

```python
def test_task_list_no_n_plus_one(authenticated_client, user):
    TaskFactory.create_batch(10, user=user)
    with assertNumQueries(2):  # 1 for tasks, 1 for users
        response = authenticated_client.get("/api/tasks/")
    assert response.status_code == 200
```

---

### Migrations

**9. Never edit a migration after it has been applied to any environment. Create a new migration instead.**

**10. For large tables (millions of rows), adding a column with a default locks the table in older PostgreSQL. Use `SeparateDatabaseAndState` to add the column without default, then backfill in a task.**

```python
# Safe migration for large tables in PostgreSQL 11+
# PostgreSQL 11+ handles DEFAULT without table rewrite for constant defaults
operations = [
    migrations.AddField(
        model_name="task",
        name="priority",
        field=models.IntegerField(default=0),
    )
]
```

**11. Always run `--check` in CI to detect uncommitted migrations before deployment.**

```bash
python manage.py makemigrations --check
```

---

### Connection pooling

**12. Never open a new DB connection per request at high concurrency — use PgBouncer in production or Django's persistent connections.**

```python
# settings/production.py
DATABASES["default"]["CONN_MAX_AGE"] = 60  # reuse connections for 60s
```

**13. Set `CONN_HEALTH_CHECKS = True` (Django 4.1+) to avoid using stale connections.**

---

### psycopg3

**14. Rafiki uses psycopg3 (`psycopg[c]`). Use `%s` placeholders — never f-strings or `.format()` in raw SQL.**

```python
from django.db import connection

# Bad — SQL injection risk
cursor.execute(f"SELECT * FROM tasks WHERE user_id = {user_id}")

# Good — parameterized
cursor.execute("SELECT * FROM tasks WHERE user_id = %s", [user_id])
```

**15. Use `connection.execute_many()` for bulk inserts via raw SQL. Prefer `bulk_create()` via the ORM.**

---

## Common mistakes

- No index on FK columns used in `filter()` — Django creates the FK constraint but not always the index
- Using `len(queryset)` instead of `.count()` — loads all rows into memory just to count
- Running migrations during peak traffic without considering table locks
- Not setting `CONN_MAX_AGE` — creates a new connection per request under load
- Using `order_by()` on unindexed columns on large tables — full table sort

---

## References

- [PostgreSQL EXPLAIN docs](https://www.postgresql.org/docs/current/sql-explain.html)
- [Django DB optimization](https://docs.djangoproject.com/en/stable/topics/db/optimization/)
- [psycopg3 docs](https://www.psycopg.org/psycopg3/docs/)
- [pganalyze index advisor](https://pganalyze.com/docs/index-advisor)
