# Django — Expert Best Practices

## Mental model

Django's ORM is a query builder that generates SQL lazily. Querysets are not evaluated until iterated, sliced, or forced. Every extra query you trigger in a loop is an N+1 bug waiting to cause a production incident. Think in SQL, write in ORM.

---

## Expert rules

### ORM & Queries

**1. Always use `select_related` for ForeignKey/OneToOne and `prefetch_related` for ManyToMany/reverse FK.**

```python
# Bad — N+1: one query per user to get their profile
users = User.objects.all()
for user in users:
    print(user.profile.bio)

# Good — 1 JOIN
users = User.objects.select_related("profile").all()
```

**2. Use `prefetch_related` with `Prefetch()` for filtered or annotated reverse relations.**

```python
from django.db import models

tasks = Task.objects.prefetch_related(
    models.Prefetch(
        "subtasks",
        queryset=SubTask.objects.filter(done=False).order_by("due_date"),
        to_attr="pending_subtasks",
    )
)
# Access without extra queries
for task in tasks:
    print(task.pending_subtasks)
```

**3. Push aggregations into the database with `annotate()`, never in Python.**

```python
# Bad — loads all objects into memory
total = sum(task.duration for task in Task.objects.all())

# Good — single SUM query
from django.db.models import Sum
total = Task.objects.aggregate(total=Sum("duration"))["total"]

# Annotate per-object
tasks = Task.objects.annotate(subtask_count=models.Count("subtasks"))
```

**4. Use `only()` / `defer()` when you need a subset of fields on large models.**

```python
# Fetches only name and email — no other columns
users = User.objects.only("name", "email")
```

**5. Use `iterator()` for large querysets to avoid loading everything into memory.**

```python
for task in Task.objects.filter(done=False).iterator(chunk_size=500):
    process(task)
```

**6. Use `update()` and `delete()` for bulk operations — never loop.**

```python
# Bad — one UPDATE per object
for task in tasks:
    task.status = "archived"
    task.save()

# Good — single UPDATE
Task.objects.filter(due_date__lt=today).update(status="archived")
```

**7. Use `F()` expressions for atomic field updates — avoids race conditions.**

```python
from django.db.models import F

# Bad — read-modify-write, race condition under concurrency
task.view_count += 1
task.save()

# Good — single atomic UPDATE
Task.objects.filter(pk=task.pk).update(view_count=F("view_count") + 1)
```

---

### Transactions

**8. Use `transaction.atomic()` to group related writes. Nest with `savepoint=True` for partial rollback.**

```python
from django.db import transaction

@transaction.atomic
def create_task_with_subtasks(data):
    task = Task.objects.create(**data["task"])
    for subtask_data in data["subtasks"]:
        SubTask.objects.create(task=task, **subtask_data)
    return task
```

**9. Never put slow I/O (HTTP calls, emails, Celery tasks) inside `atomic()` blocks. The transaction holds a DB lock for its duration.**

```python
# Bad — lock held while sending email
with transaction.atomic():
    task = Task.objects.create(...)
    send_email(task)  # slow, holds lock

# Good — use on_commit to fire side effects after commit
with transaction.atomic():
    task = Task.objects.create(...)
    transaction.on_commit(lambda: send_task_email.delay(task.pk))
```

**10. `ATOMIC_REQUESTS = True` (already set in Rafiki) wraps every request in a transaction. Don't open nested transactions unless you need savepoint semantics.**

---

### Models

**11. Use custom managers for reusable querysets — never duplicate filter logic across views.**

```python
class ActiveTaskManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(archived=False)

class Task(models.Model):
    objects = models.Manager()      # default
    active = ActiveTaskManager()    # Task.active.all()
```

**12. Add `__str__`, `Meta.ordering`, and `get_absolute_url()` to every model.**

**13. Use `model_utils.TimeStampedModel` (already a dep) for `created` / `modified` timestamps.**

```python
from model_utils.models import TimeStampedModel

class Task(TimeStampedModel):
    name = models.CharField(max_length=255)
```

**14. Avoid `null=True` on string fields (`CharField`, `TextField`). Use `blank=True` + empty string as the "no value" sentinel. `null=True` on string fields creates two empty states.**

---

### Signals

**15. Use signals only for cross-app side effects. If the logic is in the same app, call it directly.**

**16. Always connect signals in `AppConfig.ready()`, not at module level.**

```python
# users/apps.py
class UsersConfig(AppConfig):
    def ready(self):
        import rafiki.users.signals  # noqa: F401
```

**17. Make signal handlers idempotent. They can fire multiple times (e.g., `post_save` with `created=False`).**

---

### Caching

**18. Cache at the view level with `@cache_page` or at the queryset level with `django-redis`.**

```python
from django.core.cache import cache

def get_user_task_count(user_id):
    key = f"user:{user_id}:task_count"
    count = cache.get(key)
    if count is None:
        count = Task.objects.filter(user_id=user_id).count()
        cache.set(key, count, timeout=300)
    return count
```

**19. Use `cache.get_or_set()` to avoid cache stampede on cold starts.**

```python
count = cache.get_or_set(key, lambda: Task.objects.filter(...).count(), 300)
```

---

## Common mistakes

- Forgetting `.select_related()` in `ListView` — Django debug toolbar will show 50+ duplicate queries
- Using `save()` in a loop instead of `bulk_create()` / `bulk_update()`
- Putting `transaction.on_commit()` inside a non-atomic block — it fires immediately, not after commit
- Using `get()` without a try/except for `DoesNotExist` — always use `get_object_or_404()` in views
- Overriding `save()` for side effects — use signals or service functions to keep models thin

---

## References

- [Django ORM Optimization Guide](https://docs.djangoproject.com/en/stable/topics/db/optimization/)
- [Django transactions](https://docs.djangoproject.com/en/stable/topics/db/transactions/)
- [django-model-utils](https://django-model-utils.readthedocs.io/)
