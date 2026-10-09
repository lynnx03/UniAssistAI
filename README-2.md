# Login & Fetch Flow — KMITL Registration System (regis.reg.kmitl.ac.th)

Summary for presenting to the advisor: we found that this system has **2 layers** of protection
that have nothing to do with "changing semesters", and there is now a clear way to handle them.

## Findings from DevTools

| Layer | Name | Purpose | Lifetime |
|---|---|---|---|
| 1 | JWT Access Token (Keycloak SSO) | Authenticates the user, sent in the `Authorization: Bearer ...` header | Short (minutes to hours), must be refreshed |
| 2 | Incapsula Cookie (`visid_incap`, `incap_ses`) | Protects the site from bots/scrapers | Rotates periodically, unrelated to login |

**Key finding:** the year/semester/faculty/curriculum parameters (`selected_year`,
`selected_semester`, ...) are sent **directly as a query string** and are not stored in
a server-side session at all. So changing the semester **does not invalidate the token or cookie
or require logging in again** — you only change the parameter values in the same request.

## API endpoint for fetching the course schedule

```
GET https://regis.reg.kmitl.ac.th/api/?function=get-teach-table-show
    &mode=by_class
    &selected_year=2569
    &selected_semester=1
    &selected_faculty=07
    &selected_department=01
    &selected_curriculum=x
    &selected_class_year=0
    &search_all_curriculum=true
    &search_all_class_year=true
Headers: Authorization: Bearer <access_token>
Cookies: (Incapsula cookies obtained during login)
```

## Workflow (2 files)

```
┌──────────────────────┐         ┌────────────────────────────┐
│  kmitl_login.py      │         │  kmitl_fetch.py            │
│  (run once)          │  --->   │  (can be run repeatedly)   │
│                      │         │                            │
│  1. Open a browser   │         │  1. Load token + cookies   │
│  2. User logs in     │         │  2. Check if the token is  │
│     manually         │ session │     about to expire        │
│  3. Capture token    │ _state  │     -> refresh             │
│     + cookies from   │ .json   │  3. Loop over each year/   │
│     network response │         │     semester/faculty and   │
│  4. Save to JSON     │         │     call the API           │
│                      │         │  4. Save results as JSON   │
└──────────────────────┘         └────────────────────────────┘
```

- `kmitl_login.py` — opens a real browser via Playwright so the user can log in
  themselves (supports 2FA/captcha), then listens for the network request that returns the
  `access_token` / `refresh_token`, collects all cookies, and saves them to `session_state.json`
- `kmitl_fetch.py` — loads `session_state.json` and uses it to call the API to fetch course
  schedules, looping over multiple years/semesters/faculties without logging in again. It
  automatically refreshes the token when it is about to expire, or warns when it gets a 401

## Still to verify (before production use)

1. **The actual token endpoint** — the code currently assumes
   `https://sso.reg.kmitl.ac.th/realms/registrar/protocol/openid-connect/token`.
   Open DevTools, click the request named `token`, and check the full Request URL
   to confirm (the URL was cut off in the screenshot)
2. **Whether the refresh token grant actually works** — some Keycloak realms disable
   refresh tokens. If disabled, the whole login flow must be repeated every time the token
   expires (still works, just less convenient)
3. **Incapsula rate limiting / bot detection** — sending requests too frequently
   may get you blocked. There should be a delay between requests (the code already has
   `time.sleep(1)`; increase it if you get blocked)

## How to run

```bash
pip install playwright requests --break-system-packages
playwright install chromium

python data/fetch_teach_table/kmitl_login.py     # log in once, produces session_state.json
python data/fetch_teach_table/kmitl_fetch.py     # fetch data for the years/semesters defined in jobs[]
```
