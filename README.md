# bstackdemo-offline

A static, offline mirror of [bstackdemo.com](https://www.bstackdemo.com) - the home page (`/`)
and sign-in page (`/signin`) only, trimmed to what the
[test-observability-samples wdio specs](https://github.com/browserstack/test-observability-samples/tree/main/test-samples/nodejs/wdio)
actually assert on.

Built to avoid two problems with pointing automated tests at the real site:
- bstackdemo.com occasionally fails to load under load from the many test suites hitting it.
- The app itself simulates a "slow network" screen (it reads the browser's Network
  Information API - `navigator.connection.effectiveType` - and hides the whole product
  grid if that heuristic ever reports `"2g"`). That heuristic is unrelated to whether the
  page actually loaded, and can misfire on any browser, including cloud/virtualized ones.
  This mirror neutralizes it since no test asserts on it.

## What's here

- `index.html`, `signin/index.html` - captured HTML shells (CSS stripped; no test asserts
  on styling)
- `_next/static/**` - the JS chunks needed to hydrate both pages (product/logo images
  dropped - no test asserts on them)
- `api/products` - static JSON, served as a plain file at the exact path the app fetches
- `.nojekyll` - required so GitHub Pages doesn't drop the `_next/` directory (Jekyll
  ignores underscore-prefixed paths by default)

Total size: ~780 KB (vs. ~4.7 MB captured before trimming unused assets).

### The one shim

`bstackdemo` calls `POST /api/signin` via `axios` (XMLHttpRequest, not `fetch`) on login.
Static hosting can't run that server logic, and every test here only ever exercises the
*failure* path (empty/invalid credentials), so both HTML files carry a small inline script
that patches `XMLHttpRequest` to short-circuit that one endpoint and return exactly what
the real API returns for any credentials:

```
422 {"errorMessage":"Invalid Username"}
```

Everything else (all `GET`s, the product grid, all other pages) passes through untouched.

The same script also pins `navigator.connection` to report `"4g"`, neutralizing the
slow-network screen described above.

## Verified against the real site

- `/` renders `.Navbar_logo__26S5Y`, `#signin` ("Sign In"), `.products-found`
  ("25 Product(s) found."), `.sort` ("Order by...") - matches `home.e2e.js`.
- `/signin` renders `.Modal_modal__3I0HK`, `#username`, `#password`, `#login-btn`, and
  clicking submit with empty fields shows `.api-error` with text "Invalid Username" -
  matches `signIn.e2e.js`.

## Local preview

Any static file server works, e.g.:

```
npx serve .
```

`/signin` needs directory-index resolution (`signin/index.html` for the `/signin` path) -
`npx serve` and GitHub Pages both do this natively.

## Publishing

1. Push this repo to GitHub.
2. Repo Settings -> Pages -> deploy from the `main` branch, root folder.
3. Point the wdio specs' page objects at the Pages URL instead of `https://www.bstackdemo.com`
   (see `pageobjects/bstack-demo/page.js` in test-observability-samples).

Note: BrowserStack Automate runs the browser on BrowserStack's own machines, not on your
laptop - `localhost` isn't reachable from there without BrowserStack Local. A Pages URL is
a real, publicly reachable endpoint, so Automate can load it directly with no tunnel.

## Regenerating the mirror

If bstackdemo.com's build changes (new asset hashes), re-capture with a small Node script
that fetches `/` and `/signin`, extracts every `/_next/static/...` reference, downloads
each, and re-applies the same trims (drop CSS + images, add the shim, add `.nojekyll`).
