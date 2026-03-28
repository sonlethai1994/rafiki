# rafiki/users/

Custom User model and authentication. The most non-standard part of the codebase — read this before touching auth.

## User model (`models.py`)

- Extends `AbstractUser`
- **Email is the login identifier** — `USERNAME_FIELD = "email"`, `REQUIRED_FIELDS = []`
- `username` field is removed
- Single `name` field (255 chars) replaces `first_name` / `last_name`
- Custom `UserManager` in `managers.py` handles email normalization

```python
# Always create users via the manager
user = User.objects.create_user(email="foo@bar.com", password="...", name="Foo")
```

## Auth (allauth)

- Email-only signup and login via `django-allauth`
- MFA support included (`django-allauth[mfa]`)
- Custom adapters in `adapters.py` handle signup form fields
- Email verification is mandatory (set in `base.py`)
- Admin login goes through allauth when `HEADLESS_ONLY` is not set

Do not bypass allauth for user creation in views — always use the adapter or the manager.

## API (`api/`)

- `UserViewSet` — Retrieve, List, Update only (no create/delete via API)
- `/api/users/me/` — custom action returning the current user
- Users can only see their own record (queryset filtered by `request.user`)
- Serializer uses `HyperlinkedModelSerializer` — relations are URLs, not PKs

```python
# Adding a new user API endpoint
@action(detail=False, methods=["get"])
def me(self, request):
    serializer = self.get_serializer(request.user)
    return Response(serializer.data)
```

## Tests

Always use the factory — never create users directly:

```python
from rafiki.users.tests.factories import UserFactory

user = UserFactory()                    # random user
user = UserFactory(email="a@b.com")    # specific email
```

Factory defined in `tests/factories.py` using `factory-boy`.

## File map

| File                    | Purpose                                     |
| ----------------------- | ------------------------------------------- |
| `models.py`             | Custom User model                           |
| `managers.py`           | CustomUserManager (email normalization)     |
| `adapters.py`           | Allauth signup/social adapters              |
| `forms.py`              | Signup form (name field)                    |
| `views.py`              | Detail, Update, Redirect views              |
| `urls.py`               | User URL patterns                           |
| `admin.py`              | Admin registration with allauth integration |
| `context_processors.py` | Injects allauth settings into templates     |
| `tasks.py`              | Celery tasks for user-related async work    |
| `api/views.py`          | DRF UserViewSet                             |
| `api/serializers.py`    | UserSerializer (hyperlinking)               |
| `tests/factories.py`    | UserFactory                                 |
