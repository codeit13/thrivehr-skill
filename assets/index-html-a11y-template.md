# index.html Accessibility Baseline

Copy into `thrivehr-ui/index.html`.

```html
<!doctype html>
<html lang="en-IN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="color-scheme" content="light dark" />
    <meta name="description" content="Axis Bank ThriveHR — Hiring and Onboarding Portal" />
    <title>ThriveHR</title>
    <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### Rules

| Do | Don't |
|----|-------|
| `lang="en-IN"` on `<html>` | Omit `lang` |
| `width=device-width, initial-scale=1.0` | `maximum-scale=1` or `user-scalable=no` |
| `color-scheme: light dark` | Block system theme without user toggle |
| Unique titles per route (set in React) | Static "React App" title everywhere |

Skip link is rendered in React (`SkipToContent`), not in static HTML, so it works with client routing.
