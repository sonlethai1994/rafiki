# Django REST Framework — Expert Best Practices

## Mental model

DRF is a layered system: Router → View → Serializer → Renderer. Each layer has a single responsibility. Violations (business logic in serializers, DB queries in views) create untestable, slow code. Keep views thin, serializers for shape, business logic in service functions.

---

## Expert rules

### ViewSets

**1. Use `get_queryset()` — never set `queryset` as a class attribute when the result depends on the request.**

```python
# Bad — queryset evaluated at class definition time, ignores request context
class TaskViewSet(viewsets.ModelViewSet):
    queryset = Task.objects.all()

# Good
class TaskViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        return Task.objects.filter(user=self.request.user).select_related("category")
```

**2. Override `get_serializer_class()` to return different serializers per action.**

```python
class TaskViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.action in ("create", "update", "partial_update"):
            return TaskWriteSerializer
        return TaskReadSerializer
```

**3. Use `get_permissions()` for action-level permissions instead of a single `permission_classes`.**

```python
def get_permissions(self):
    if self.action == "destroy":
        return [IsAdminUser()]
    return [IsAuthenticated()]
```

**4. Use `@action` for non-CRUD endpoints. Set `url_path` explicitly for clean URLs.**

```python
@action(detail=True, methods=["post"], url_path="archive")
def archive(self, request, pk=None):
    task = self.get_object()
    task.archive()
    return Response(status=status.HTTP_204_NO_CONTENT)
```

**5. Always call `self.get_object()` (not `Task.objects.get(pk=pk)`) — it applies `get_queryset()` filters and calls `check_object_permissions()`.**

---

### Serializers

**6. Use `SerializerMethodField` for computed/derived fields. Keep it read-only and documented.**

```python
class TaskSerializer(serializers.ModelSerializer):
    is_overdue = serializers.SerializerMethodField()

    def get_is_overdue(self, obj) -> bool:
        return obj.due_date < date.today() and not obj.done
```

**7. Use `validate_<field>()` for single-field validation, `validate()` for cross-field validation.**

```python
def validate_due_date(self, value):
    if value < date.today():
        raise serializers.ValidationError("Due date cannot be in the past.")
    return value

def validate(self, attrs):
    if attrs["end_date"] < attrs["start_date"]:
        raise serializers.ValidationError("End must be after start.")
    return attrs
```

**8. Use `source` to map serializer fields to model fields cleanly — avoid renaming in views.**

```python
class UserSerializer(serializers.ModelSerializer):
    full_name = serializers.CharField(source="name", read_only=True)
```

**9. Use `to_representation()` to transform output — cleaner than overriding every field.**

```python
def to_representation(self, instance):
    data = super().to_representation(instance)
    data["tags"] = [t.lower() for t in data["tags"]]
    return data
```

**10. Avoid N+1 in nested serializers — always annotate or prefetch in the parent viewset's queryset.**

---

### Permissions

**11. Write granular object-level permissions — subclass `BasePermission` and implement `has_object_permission()`.**

```python
class IsOwner(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.user == request.user
```

**12. Compose permissions with `and`/`or` operators (DRF 3.14+).**

```python
permission_classes = [IsAuthenticated & (IsOwner | IsAdminUser)]
```

---

### Authentication

**13. Rafiki uses both `SessionAuthentication` and `TokenAuthentication`. Session for browser, token for API clients. Always set `DEFAULT_AUTHENTICATION_CLASSES` in `base.py` — not per-view.**

**14. For Celery tasks that need to act on behalf of a user, pass `user.pk` — never pass the User object directly (not serializable).**

---

### OpenAPI (drf-spectacular)

**15. Use `@extend_schema` to enrich auto-generated docs — especially for custom actions and non-obvious responses.**

```python
from drf_spectacular.utils import extend_schema, OpenApiParameter

@extend_schema(
    parameters=[OpenApiParameter("status", str, description="Filter by status")],
    responses={200: TaskSerializer(many=True)},
)
@action(detail=False, methods=["get"])
def by_status(self, request):
    ...
```

**16. Use `@extend_schema_view` to annotate all actions on a ViewSet at once.**

```python
@extend_schema_view(
    list=extend_schema(summary="List all tasks"),
    create=extend_schema(summary="Create a task"),
)
class TaskViewSet(viewsets.ModelViewSet):
    ...
```

---

### Performance

**17. Use `select_related` / `prefetch_related` in `get_queryset()`. Serializers trigger DB queries for every nested relation if you don't.**

**18. Use pagination — never return unbounded querysets. DRF's `PageNumberPagination` with a sane `PAGE_SIZE` (e.g., 25).**

**19. Throttle public endpoints. Use `AnonRateThrottle` + `UserRateThrottle` in `DEFAULT_THROTTLE_CLASSES`.**

---

## Common mistakes

- Putting DB queries in `validate()` or `to_representation()` — these run per object in list views
- Not calling `serializer.is_valid(raise_exception=True)` — swallows validation errors silently
- Using `ModelSerializer` with `fields = "__all__"` — exposes fields you didn't intend to
- Forgetting `read_only_fields` — allows clients to set `user`, `created_at`, etc.
- Returning `Response(serializer.data)` after a write without re-fetching — stale data if `annotate()` adds computed fields

---

## References

- [DRF Docs](https://www.django-rest-framework.org/)
- [drf-spectacular](https://drf-spectacular.readthedocs.io/)
- [Classy DRF](https://www.cdrf.co/) — CBV method resolution at a glance
