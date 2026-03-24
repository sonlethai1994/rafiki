# Celery — Expert Best Practices

## Mental model

A Celery task is a unit of work that must be safe to run multiple times (idempotent), safe to fail and retry, and completely decoupled from the HTTP request cycle. If your task is not idempotent, you have a bug waiting to happen on retry.

---

## Expert rules

### Task design

**1. Make every task idempotent — retrying must produce the same result as running once.**

```python
@shared_task
def send_welcome_email(user_id: int) -> None:
    user = User.objects.get(pk=user_id)
    if user.welcome_email_sent:
        return  # already done, safe to retry
    send_email(user)
    User.objects.filter(pk=user_id).update(welcome_email_sent=True)
```

**2. Pass PKs, not model instances. Instances are serialized to JSON — they go stale and are large.**

```python
# Bad
send_report.delay(user=user_obj)

# Good
send_report.delay(user_id=user.pk)
```

**3. Keep tasks small and focused. A task should do one thing. Chain tasks for multi-step workflows.**

**4. Always set `bind=True` and use `self.retry()` for retries — never call the task recursively.**

```python
@shared_task(bind=True, max_retries=3)
def fetch_external_data(self, resource_id: int) -> dict:
    try:
        return api_client.get(resource_id)
    except RateLimitError as exc:
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)  # exponential backoff
```

**5. Use `autoretry_for` for clean retry declarations on known transient errors.**

```python
@shared_task(
    bind=True,
    autoretry_for=(RequestException, TimeoutError),
    retry_backoff=True,          # exponential backoff
    retry_backoff_max=600,       # cap at 10 minutes
    retry_jitter=True,           # randomize to avoid thundering herd
    max_retries=5,
)
def call_webhook(self, url: str, payload: dict) -> None:
    requests.post(url, json=payload, timeout=10)
```

---

### Reliability

**6. Use `transaction.on_commit()` to enqueue tasks — ensures the task only fires after the DB transaction commits.**

```python
# Bad — task may run before DB row is visible
task = Task.objects.create(...)
process_task.delay(task.pk)

# Good
with transaction.atomic():
    task = Task.objects.create(...)
    transaction.on_commit(lambda: process_task.delay(task.pk))
```

**7. Set task time limits in `base.py`. Always set both hard and soft limits.**

```python
CELERY_TASK_TIME_LIMIT = 300       # hard kill after 5 min
CELERY_TASK_SOFT_TIME_LIMIT = 60   # SoftTimeLimitExceeded at 1 min — allows cleanup
```

**8. Catch `SoftTimeLimitExceeded` for graceful shutdown.**

```python
from celery.exceptions import SoftTimeLimitExceeded

@shared_task(bind=True)
def long_running(self):
    try:
        do_work()
    except SoftTimeLimitExceeded:
        cleanup()
        raise
```

---

### Workflows (canvas)

**9. Use `chain` for sequential tasks, `group` for parallel, `chord` for parallel + callback.**

```python
from celery import chain, group, chord

# Sequential
pipeline = chain(fetch_data.s(url), transform.s(), save.s())
pipeline.delay()

# Parallel then aggregate
result = chord(
    group(process_chunk.s(chunk) for chunk in chunks),
    aggregate_results.s(),
)()
```

**10. Prefer `chain` over nesting `.delay()` calls inside tasks — it makes the workflow explicit and traceable in Flower.**

---

### Beat scheduling

**11. Store schedules in the DB via `django-celery-beat` — never hardcode in `CELERY_BEAT_SCHEDULE`. DB schedules can be changed without redeployment.**

**12. Always set `run_every` or `crontab` with explicit timezone.**

```python
from django_celery_beat.models import PeriodicTask, CrontabSchedule

schedule, _ = CrontabSchedule.objects.get_or_create(
    minute="0", hour="9", day_of_week="mon-fri",
    timezone="UTC",
)
PeriodicTask.objects.get_or_create(
    crontab=schedule,
    name="Daily digest",
    task="rafiki.tasks.send_daily_digest",
)
```

**13. Beat is a single process — it only schedules tasks, it does not execute them. Always run at least one worker alongside beat.**

---

### Memory & performance

**14. Workers leak memory over time. Set `CELERY_WORKER_MAX_TASKS_PER_CHILD` to recycle worker processes.**

```python
CELERY_WORKER_MAX_TASKS_PER_CHILD = 1000
```

**15. Use separate queues for fast vs slow tasks. Prevents long-running tasks from blocking short ones.**

```python
# In task definition
@shared_task(queue="low_priority")
def generate_report(): ...

# Start workers per queue
celery -A config.celery_app worker -Q default -c 4
celery -A config.celery_app worker -Q low_priority -c 2
```

---

## Common mistakes

- Not making tasks idempotent — retries create duplicate records, emails, charges
- Passing model objects instead of PKs — stale data on retry
- Firing tasks inside transactions without `on_commit` — task runs before DB commit, gets `DoesNotExist`
- No time limits — a hung task blocks a worker slot forever
- Using `beat` without a `--pidfile` — multiple beat instances run and double-schedule tasks (already handled in Rafiki's start script)
- Catching all exceptions with bare `except Exception` and not re-raising — hides failures from Flower and Sentry

---

## References

- [Celery Best Practices (Blog)](https://denibertovic.com/posts/celery-best-practices/)
- [Celery Canvas docs](https://docs.celeryq.dev/en/stable/userguide/canvas.html)
- [django-celery-beat](https://django-celery-beat.readthedocs.io/)
