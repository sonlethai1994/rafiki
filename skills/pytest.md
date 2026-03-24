# pytest + factory-boy — Expert Best Practices

## Mental model

Tests are code. The same quality standards apply. A test that tests nothing (always passes) is worse than no test — it creates false confidence. A slow test suite is a test suite developers stop running. Optimize for speed, isolation, and clarity.

---

## Expert rules

### Fixtures

**1. Scope fixtures appropriately. `session` scope for expensive setup (DB schema), `function` scope for mutable state.**

```python
@pytest.fixture(scope="session")
def django_db_setup():
    # runs once per test session
    ...

@pytest.fixture
def user(db):
    # runs fresh per test — mutable state must be function-scoped
    return UserFactory()
```

**2. Use `pytest-django`'s `@pytest.mark.django_db` or `db` fixture — never `TransactionTestCase` unless you specifically need transaction rollback behavior.**

**3. Use `django_db(transaction=True)` only for testing Celery tasks, signals that use `on_commit`, or actual DB transactions.**

```python
@pytest.mark.django_db(transaction=True)
def test_task_fires_after_commit():
    with transaction.atomic():
        task = TaskFactory()
        transaction.on_commit(lambda: process_task.delay(task.pk))
    # on_commit fires here
    assert process_task.called
```

---

### factory-boy

**4. Always use factories — never `Model.objects.create()` in tests. Factories handle defaults and relations.**

```python
# rafiki/users/tests/factories.py
import factory
from factory.django import DjangoModelFactory

class UserFactory(DjangoModelFactory):
    class Meta:
        model = "users.User"

    email = factory.Sequence(lambda n: f"user{n}@example.com")
    name = factory.Faker("name")
    password = factory.PostGenerationMethodCall("set_password", "password")
```

**5. Use `SubFactory` for related objects — never create them manually in tests.**

```python
class TaskFactory(DjangoModelFactory):
    class Meta:
        model = "tasks.Task"

    user = factory.SubFactory(UserFactory)
    name = factory.Faker("sentence", nb_words=4)
    due_date = factory.Faker("future_date")
```

**6. Use `Trait` for variant states — cleaner than multiple factories or long kwargs.**

```python
class TaskFactory(DjangoModelFactory):
    class Meta:
        model = "tasks.Task"
        exclude = ["is_done"]

    class Params:
        is_done = factory.Trait(
            status="done",
            completed_at=factory.Faker("past_datetime"),
        )

# Usage
task = TaskFactory(is_done=True)
```

**7. Use `factory.LazyAttribute` for fields that depend on other fields.**

```python
class UserFactory(DjangoModelFactory):
    name = factory.Faker("name")
    email = factory.LazyAttribute(lambda o: f"{o.name.lower().replace(' ', '.')}@example.com")
```

**8. Use `create_batch()` for list tests — avoids repetition.**

```python
tasks = TaskFactory.create_batch(10, user=user, status="pending")
```

---

### API testing

**9. Use DRF's `APIClient` — not Django's test `Client`. It handles content types and auth cleanly.**

```python
# conftest.py
@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def authenticated_client(api_client, user):
    api_client.force_authenticate(user=user)
    return api_client
```

**10. Test the full HTTP cycle — status code, response shape, and DB side effects.**

```python
def test_create_task(authenticated_client, user):
    payload = {"name": "Fix bug", "due_date": "2026-12-31"}
    response = authenticated_client.post("/api/tasks/", payload)

    assert response.status_code == status.HTTP_201_CREATED
    assert response.data["name"] == "Fix bug"
    assert Task.objects.filter(user=user, name="Fix bug").exists()
```

**11. Use `assertContains` / `assertNotContains` for template-rendered views.**

---

### Mocking

**12. Use `unittest.mock.patch` as a decorator or context manager. Patch where the object is used, not where it is defined.**

```python
# tasks.py imports requests — patch it there
@patch("rafiki.tasks.requests.post")
def test_webhook(mock_post, task):
    mock_post.return_value.status_code = 200
    call_webhook(task.pk)
    mock_post.assert_called_once()
```

**13. Use `pytest-mock`'s `mocker` fixture for cleaner mock teardown (auto-resets after test).**

```python
def test_email_sent(mocker, user):
    mock_send = mocker.patch("rafiki.users.tasks.send_mail")
    send_welcome_email(user.pk)
    mock_send.assert_called_once_with(subject="Welcome", ...)
```

**14. Mock at the boundary — mock external HTTP, file I/O, email. Never mock your own ORM calls.**

---

### Parametrize

**15. Use `@pytest.mark.parametrize` to test multiple inputs — avoids copy-pasted test functions.**

```python
@pytest.mark.parametrize("status,expected", [
    ("pending", False),
    ("done", True),
    ("archived", True),
])
def test_task_is_closed(status, expected):
    task = TaskFactory.build(status=status)
    assert task.is_closed == expected
```

---

### Speed

**16. Use `--reuse-db` (already set in `pyproject.toml`) to skip DB recreation between runs.**

**17. Use `factory.build()` instead of `factory.create()` when the test doesn't need DB persistence.**

**18. Mark slow tests with `@pytest.mark.slow` and skip them in fast feedback loops.**

---

## Common mistakes

- Using `Model.objects.create()` instead of factories — breaks when model gains required fields
- Creating objects in module-level scope instead of fixtures — shared mutable state between tests
- Not testing failure cases (invalid input, missing permissions, 404)
- Mocking the ORM — tests pass but miss real query errors
- Ignoring `pytest.warns` and `pytest.raises` — letting exceptions pass silently

---

## References

- [pytest-django docs](https://pytest-django.readthedocs.io/)
- [factory-boy docs](https://factoryboy.readthedocs.io/)
- [pytest parametrize](https://docs.pytest.org/en/stable/how-to/parametrize.html)
