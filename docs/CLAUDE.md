# docs/

Sphinx documentation. Written in reStructuredText (RST).

## Structure

```
docs/
  conf.py        # Sphinx configuration
  index.rst      # Documentation root / table of contents
  howto/         # How-to guides
  users.rst      # User management docs
  pwa.rst        # Progressive web app docs
```

## Building docs locally

```bash
# Inside Docker (uses docs service)
docker compose -f docker-compose.docs.yml up

# Locally with uv
uv run sphinx-autobuild docs docs/_build/html --port 9000
```

Live reload is enabled — changes to `.rst` files rebuild automatically.

## Adding a new page

1. Create a `.rst` file in `docs/`
2. Add it to the `toctree` in `index.rst`:

```rst
.. toctree::
   :maxdepth: 2

   users
   my-new-page
```

## RST basics

```rst
Page Title
==========

Section
-------

Subsection
~~~~~~~~~~

**bold**, *italic*, ``code``

.. code-block:: python

    print("hello")

.. note::
   This is a note.
```

## Hosting

Docs are hosted on Read the Docs (configured in `.readthedocs.yml`). Builds trigger automatically on push to `main`.
