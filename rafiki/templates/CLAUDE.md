# rafiki/templates/

Django HTML templates. All templates for all apps live here.

## Structure

```
templates/
  base.html                  # Root layout (head, navbar, footer, webpack bundles)
  pages/
    home.html
    about.html
  account/                   # Allauth overrides
  users/
    user_detail.html
    user_form.html
```

## Inheritance

All templates extend `base.html`:

```html
{% extends "base.html" %}

{% block content %}
  <!-- page content -->
{% endblock content %}
```

Key blocks in `base.html`:

| Block | Purpose |
|---|---|
| `title` | Page `<title>` tag |
| `css` | Extra stylesheets |
| `content` | Main page body |
| `modal` | Bootstrap modals |
| `javascript` | Extra scripts |

## Conventions

- **2-space indentation** (enforced by djLint)
- **Max line length:** 119 characters
- **Void tags must be closed:** `<img />`, `<input />`
- Always add a blank line after `{% load %}` and `{% extends %}`
- Use `{% url %}` for all links — never hardcode paths

## Linting

djLint runs as a pre-commit hook. To check and fix manually:

```bash
uv run djlint . --check    # Check only
uv run djlint . --reformat # Auto-fix
```

Profile is `django` (set in `pyproject.toml`). CSS and JS inside templates are also formatted.

## Allauth templates

Allauth templates are overridden in `templates/account/` and `templates/allauth/`. When allauth adds new views, check if a template override is needed.

## Bootstrap 5

Use Bootstrap 5 utility classes and components. Custom variables are in `rafiki/static/sass/custom_bootstrap_vars`. Avoid inline styles.
