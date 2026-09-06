# Vulnerable vs. Secure Flask Demo

Two self-contained Flask apps for studying eight common web vulnerabilities
side by side with their fixes.

- `vulnerable_app.py` — intentionally insecure. **Local/lab use only.**
- `secure_app.py` — same features, each vulnerability fixed.

Both apps have identical routes and a matching SQLite-backed guestbook,
login, search, and account/transfer flow, so you can trigger the same
attack against each one and see the difference.

## ⚠️ Before you run this

- Only run `vulnerable_app.py` on an isolated machine/VM/container with no
  sensitive data, bound to `127.0.0.1` (already the default in the code).
  Never expose it to a network, and never point its "Fetch a URL" or
  "Ping" features at systems you don't own or have permission to test.
- Both apps create their own SQLite database and a `files/` folder on
  first run — nothing else on your system is touched.

## Setup

```bash
pip install flask requests flask-wtf --break-system-packages
python3 vulnerable_app.py   # http://127.0.0.1:5000
python3 secure_app.py       # http://127.0.0.1:5001 (run in a second terminal)
```

Seeded accounts on both apps: `admin` / `admin123`, `alice` / `alicepass`,
`bob` / `bobpass`.

## What's vulnerable, where, and how it's fixed

| # | Vulnerability | Vulnerable route | Try this | Fix in `secure_app.py` |
|---|---|---|---|---|
| 1 | OS Command Injection | `/ping` | host = `127.0.0.1; id` | Input restricted to a hostname/IP allowlist regex; `subprocess.run([...])` with a list and no `shell=True` |
| 2 | SSRF | `/fetch` | url = `http://127.0.0.1:5000/` | Scheme allowlist (`http`/`https` only) + resolves the hostname and rejects private/loopback/link-local IPs; redirects disabled |
| 3 | SSTI | `/greet?name=` | `{{7*7}}` renders `49` | User input passed as Jinja **data** via the render context, never concatenated into the template source |
| 4 | SQL Injection | `/login`, `/search` | username = `admin' -- ` | Parameterized (`?`) queries everywhere; passwords hashed with `werkzeug.security` |
| 5 | Path Traversal | `/download?file=` | `../vulnerable_app.py` | `secure_filename()` + `send_from_directory()`, which refuses to serve outside the target folder |
| 6 | Information Disclosure | `/debug`, Flask debug mode | visit `/debug` | `debug=False`, no `/debug` route, generic 404/500 error pages, secrets loaded from environment |
| 7 | XSS (stored + reflected) | `/comment`, `/search` | post `<script>alert(1)</script>` | Jinja2 autoescaping on every `{{ variable }}`; no `\|safe` is ever applied to user-supplied data |
| 8 | CSRF | `/transfer` | submit `/transfer` from another origin | `Flask-WTF`'s `CSRFProtect` rejects any POST without a valid, session-bound `csrf_token` |

## Notes on the fixes

- **SSRF** defense here uses a private-IP blocklist, which is a reasonable
  baseline but not bulletproof (DNS rebinding can swap the IP between the
  check and the actual request). For production, prefer an allowlist of
  exact destinations you control, and re-resolve/pin the IP at connection
  time.
- **CSRF** protection is applied globally via `CSRFProtect(app)`, so every
  POST/PUT/PATCH/DELETE across the app requires a token, not just
  `/transfer`.
- Both apps use the Flask development server (`app.run(...)`), which is
  fine for this demo but shouldn't be used as-is in production — use a
  proper WSGI server (e.g. gunicorn) behind HTTPS instead.
