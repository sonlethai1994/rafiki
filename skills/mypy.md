# mypy + Type Hints — Expert Best Practices

## Mental model

Type hints are not just documentation — they are machine-checked contracts. mypy catches a class of bugs at analysis time that tests would only catch at runtime. The goal is not 100% annotation coverage; it's catching real bugs. Start strict on new code, annotate legacy incrementally.

---

## Expert rules

### Annotations

**1. Annotate function signatures fully — return types included. Skip annotation on obvious locals.**

```python
# Good — signature is the contract
def get_user_tasks(user_id: int, include_done: bool = False) -> QuerySet["Task"]:
    qs = Task.objects.filter(user_id=user_id)
    if not include_done:
        qs = qs.filter(done=False)
    return qs
```

**2. Use `Optional[X]` (or `X | None` in Python 3.10+) explicitly — never leave nullable params unannotated.**

```python
def find_task(name: str, user_id: int | None = None) -> Task | None:
    ...
```

**3. Use `TypeAlias` for complex repeated types.**

```python
from typing import TypeAlias

TaskDict: TypeAlias = dict[str, str | int | bool]
```

---

### Django-stubs

**4. Use `QuerySet[Model]` not bare `QuerySet` for typed querysets.**

```python
from django.db.models import QuerySet

def active_tasks() -> QuerySet[Task]:
    return Task.objects.filter(archived=False)
```

**5. Use `Manager` from `django-stubs` generic type for custom managers.**

```python
from django.db import models
from django.db.models.manager import Manager

class Task(models.Model):
    objects: Manager["Task"]
```

**6. Annotate `request` in views with `HttpRequest` or `AuthenticatedRequest` (custom type).**

```python
from django.http import HttpRequest

def my_view(request: HttpRequest) -> HttpResponse:
    ...
```

For DRF:

```python
from rest_framework.request import Request

def get_queryset(self) -> QuerySet[Task]:
    request: Request = self.request
    return Task.objects.filter(user=request.user)
```

---

### Advanced patterns

**7. Use `Protocol` to define structural interfaces — prefer over abstract base classes for loose coupling.**

```python
from typing import Protocol

class Notifiable(Protocol):
    def notify(self, message: str) -> None: ...

def send_notification(target: Notifiable, message: str) -> None:
    target.notify(message)
```

**8. Use `TypedDict` for dict shapes — especially useful for serializer `validated_data` and settings dicts.**

```python
from typing import TypedDict

class TaskData(TypedDict):
    name: str
    due_date: date
    user_id: int

def create_task(data: TaskData) -> Task:
    return Task.objects.create(**data)
```

**9. Use `TypeVar` for generic functions that preserve input type.**

```python
from typing import TypeVar

T = TypeVar("T")

def first_or_none(items: list[T]) -> T | None:
    return items[0] if items else None
```

**10. Use `cast()` only as a last resort. Prefer type narrowing with `isinstance()` or `assert`.**

```python
from typing import cast

# Avoid this unless you're certain
user = cast(User, request.user)

# Better — narrow with isinstance
if isinstance(request.user, User):
    ...  # mypy knows it's User here
```

---

### Class-based views

**11. Annotate CBV attributes and override return types for full coverage.**

```python
class TaskDetailView(LoginRequiredMixin, DetailView):
    model = Task
    template_name: str = "tasks/task_detail.html"

    def get_object(self, queryset: QuerySet[Task] | None = None) -> Task:
        return super().get_object(queryset)
```

---

### Handling false positives

**12. Use `# type: ignore[specific-error]` — never bare `# type: ignore`. Always name the error code.**

```python
result = some_dynamic_function()  # type: ignore[no-any-return]
```

**13. Use `reveal_type()` during development to inspect inferred types — remove before committing.**

```python
reveal_type(task.user)  # Revealed type: "rafiki.users.models.User"
```

**14. Exclude third-party packages without stubs using `ignore_missing_imports = true` in `pyproject.toml` (already set in Rafiki). Add per-module overrides for stricter control.**

```toml
[[tool.mypy.overrides]]
module = "some_untyped_package.*"
ignore_missing_imports = true
```

---

## Common mistakes

- Using `Any` as an escape hatch — defeats the purpose; use `object` or a proper Union
- Annotating `self` in methods — mypy infers it automatically
- Using string literals for forward references when `from __future__ import annotations` is available
- Not running mypy in CI — annotations drift and stop being accurate
- Ignoring mypy errors from django-stubs that are actually real bugs

---

## References

- [mypy docs](https://mypy.readthedocs.io/)
- [django-stubs](https://github.com/typeddjango/django-stubs)
- [djangorestframework-stubs](https://github.com/typeddjango/djangorestframework-stubs)
- [Python typing cheatsheet](https://mypy.readthedocs.io/en/stable/cheat_sheet_py3.html)
