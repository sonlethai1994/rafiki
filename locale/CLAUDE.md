# locale/

Internationalization (i18n) translation files.

## Supported languages

| Code    | Language             |
| ------- | -------------------- |
| `en_US` | English (default)    |
| `fr_FR` | French               |
| `pt_BR` | Brazilian Portuguese |

## How translations work

Django uses `gettext`. Strings are marked in Python and templates, extracted to `.po` files, translated, then compiled to `.mo` files.

## Workflow

### Mark strings for translation

In Python:

```python
from django.utils.translation import gettext_lazy as _

label = _("My string")
```

In templates:

```html
{% load i18n %} {% trans "My string" %} {% blocktrans %}Hello {{ name }}{%
endblocktrans %}
```

### Extract strings

```bash
just manage makemessages -l fr_FR
just manage makemessages -l pt_BR
```

This updates `.po` files in `locale/<lang>/LC_MESSAGES/django.po`.

### Translate

Edit the `.po` files — fill in `msgstr` for each `msgid`.

### Compile

```bash
just manage compilemessages
```

This generates `.mo` binary files that Django reads at runtime.

## Adding a new language

```bash
just manage makemessages -l <lang_code>
```

Then add the language code to `LANGUAGES` in `config/settings/base.py`.

## Notes

- `.mo` files are compiled artifacts — they are gitignored but must be compiled before running the app
- The Docker start scripts run `compilemessages` automatically in production
- Default language is `en-us` (set in `base.py`)
