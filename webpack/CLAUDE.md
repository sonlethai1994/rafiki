# webpack/

Frontend asset bundling. Integrates with Django via `django-webpack-loader`.

## Structure

```
webpack/
  common.config.js  # Shared config (entry points, loaders, plugins)
  dev.config.js     # Dev server (port 3000, proxies API to Django)
  prod.config.js    # Production build (source maps, optimization)
```

## Entry points

Defined in `common.config.js`:

| Entry | Purpose |
|---|---|
| `project` | Main app JS (`rafiki/static/js/project.js`) |
| `vendors` | Third-party libs (`rafiki/static/js/vendors.js`) |

Output goes to `rafiki/static/webpack_bundles/`.

## Adding a new bundle

1. Add a new entry in `common.config.js`:
   ```js
   entry: {
     project: "./rafiki/static/js/project.js",
     vendors: "./rafiki/static/js/vendors.js",
     myFeature: "./rafiki/static/js/my-feature.js",  // new
   }
   ```
2. Load it in a template:
   ```html
   {% load render_bundle from webpack_loader %}
   {% render_bundle 'myFeature' %}
   ```

## SCSS

- Entry: `rafiki/static/sass/project.scss`
- Bootstrap 5 variables customized in `rafiki/static/sass/custom_bootstrap_vars`
- Processed by `sass-loader` → `postcss-loader` (autoprefixer) → `css-loader`

## Dev server

- Runs on port 3000
- Proxies `/api/` and Django static files to port 8000
- Webpack writes `webpack-stats.json` — Django reads it via `django-webpack-loader` to resolve bundle URLs
- Hot reload enabled for JS/SCSS changes

## Scripts

```bash
npm run dev    # Start dev server (used by the `node` Docker service)
npm run build  # Production build (run inside production Docker image)
```

## Django integration

`django-webpack-loader` reads `webpack-stats.json` and injects correct bundle URLs into templates. In development, bundles are served from the webpack dev server. In production, they are collected as static files.
