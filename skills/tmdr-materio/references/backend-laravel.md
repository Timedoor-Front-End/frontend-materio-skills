# Backend — Laravel (`[BE]` items)

Read before any `[BE]` item (SKILL.md Step 3B). Laravel variant only: the
backend lives in this repo next to `resources/ts`. If the code disagrees with
this file, **trust the code** — this file gives the defaults and the contract
the FE side depends on, not a house style to impose on an existing project.

## Step A — learn the backend's conventions (once per session)

Before the first `[BE]` item, read the repo and fill in this table. Every later
`[BE]` item reuses it. Don't guess a row; if nothing in the repo answers it and
there's no existing feature to copy, use the default in the last column and say
so in your response.

| Convention | Where to look | Default when the repo has no precedent |
|---|---|---|
| Laravel / PHP version | `composer.json` | — (always read it) |
| Route file + prefix | `bootstrap/app.php` (L11+) or `RouteServiceProvider`, `routes/api.php`, `routes/web.php` | `routes/api.php`, versioned prefix `/api/v1` (`withRouting(api: ..., apiPrefix: 'api/v1')`; checklist CD7) |
| Auth | `config/auth.php`, `config/sanctum.php`, middleware on existing routes; must match `auth-modes.md` mode | Sanctum SPA session (mode B), `auth:sanctum` |
| Authorization | `app/Policies`, `Gate::define`, `spatie/laravel-permission` in `composer.json` | Policy per model when the PRD restricts who may act |
| Controller style | `app/Http/Controllers` — resource controllers? single-action? thin controller + `app/Actions` / `app/Services`? | Resource controller (`index/store/show/update/destroy`), logic inline unless it's reused |
| Validation | `app/Http/Requests` | One FormRequest per write action (`StoreXRequest`, `UpdateXRequest`) |
| Response shape | `app/Http/Resources`, existing controller returns | `JsonResource`; collections via `XResource::collection($paginator)` |
| Enums / statuses | `app/Enums`, model `$casts` | Backed PHP enum + cast |
| Tests | `tests/Feature`, `pest` vs `phpunit` in `composer.json`, `phpunit.xml` DB settings | Feature tests in the framework the repo uses |
| Code style | `pint.json`, existing formatting | Laravel Pint preset |

Read **one existing feature end to end** (route → controller → request →
resource → test) and copy its shape. If two features disagree, ask which one
is current.

## The contract with the FE side

These keep `[BE]` output plug-compatible with the starter kit's FE helpers.
Where the repo already does otherwise, follow the repo and adapt the FE
helper once (never per page) — and flag it.

- **Validation errors:** Laravel's default `422 { message, errors: { field: [msg] } }`.
  Business-rule failures (stock insufficient, already cancelled) are also a
  `422` with a field key (`ValidationException::withMessages([...])`) so
  `apiErrorMessage()` shows the right text. Don't return `400`/`200` with an
  error flag.
- **Lists:** accept what `useServerTable` sends — `page`, `per_page`,
  `sort_by`, `sort_order` (`asc`/`desc`), plus the filter names the page uses
  (`q` for search) — and return Laravel's paginator JSON
  (`->paginate($request->integer('per_page', 10))`, wrapped in a resource collection).
  - Whitelist `sort_by` against an explicit column list; unknown → default sort.
  - Cap `per_page` (e.g. max 100).
  - Search/filter/sort in the query (including computed values via
    `withCount`/`withSum`/subselects), never after `->get()`.
- **Key casing:** `snake_case` in requests and responses unless the repo
  already uses camelCase.
- **Money:** an integer in **minor units** (cents, sen — 100 per unit by default)
  in the database (`unsignedBigInteger` / `bigInteger` — a 32-bit `integer`
  overflows at 21,474,836.47 with 2 decimals) and in JSON, matching
  `NumberField currency` and the money formatter. Never floats or `decimal`
  strings. Max `9000000000000000` (the FE cap, `data-patterns.md` → "Input
  limits"). If the project stores whole units instead, follow it and note that
  FE needs `:minor-units="1"` (`starter-kit.md` → "Conventions").
- **Other decimals** (weight, temperature, rates): `decimal(p,s)` column, a JSON
  **number** in the resource (`(float) $this->weight` — Laravel's `decimal:2`
  cast returns a string), matching `NumberField :decimals`.
- **Dates/times:** dates as `Y-m-d`, times as `H:i` (cast/format in the
  resource), matching `DateTimeField`. Timestamps as ISO 8601.
- **Delete:** only if the PRD asks for it. `204` on success; a delete blocked by
  a business rule is a `422` with a message.
- **Auth endpoints** (when an auth `[BE]` item is planned): login, logout and
  "me", in the shape `auth-modes.md` describes for the project's mode. All
  `POST` except "me". Throttle login. Don't reveal which credential was wrong.
  Every non-public route sits behind `auth:*`.
- **Password reset** (when the PRD has one): verification always goes to the
  user's **registered** email or phone.
  - Link (default): Laravel's `Password` broker (`sendResetLink` / `reset`),
    which gives a signed, expiring, single-use token. `forgot-password` answers
    the same whether or not the email exists, and is throttled.
  - OTP: store the code **hashed**, expiring (≤ 10 minutes), with a maximum
    number of attempts, single-use, and resends throttled.
  - Both: the send response includes `expires_in` and `resend_in` (seconds) for
    the FE countdown. A successful reset deletes the user's other tokens and
    sessions.
  - Public auth forms (register, forgot password) validate `recaptcha_token`
    (checklist S9).

## Building one `[BE]` item

Only what the PRD/prompt asks for — no speculative columns, endpoints or
features. Ambiguous fields, rules or permissions → ask.

1. **Migration** — new file via `php artisan make:migration`; foreign keys
   with `constrained()` and an explicit `cascadeOnDelete`/`restrictOnDelete`
   choice; indexes on columns that are filtered, sorted or searched; column
   sizes match the field limits (`string('name', 100)`, money `bigInteger`,
   rich text `mediumText`). **Never edit a migration that's already committed**
   — add a new one.
2. **Model** — `$fillable` (or the repo's `$guarded` habit), `$casts` (enums,
   dates, booleans), relations, query scopes for list filters.
3. **Factory** (+ seeder only if the repo seeds that kind of data).
4. **Form requests** — `rules()` mirrors the PRD; the FE's client `:rules` are
   only a fast-feedback copy, these are the source of truth. Every field has an
   upper bound (`max:` / `between:` / `digits:` / `size:`; files `max:` in KB +
   `mimes:`; arrays `max:` items) from `data-patterns.md` → "Input limits" —
   without one, an over-long value hits the column and MySQL strict mode answers
   `500 Data too long` instead of a `422`. Rich-text fields: see "Rich text".
   Uploads: see "Uploads". User-entered URLs: `url:https` + `max:2048`, so
   only `https` links get saved (checklist AE4).
   `authorize()` delegates to the policy (or returns `true` if routes are
   already guarded).
5. **Policy** — when the PRD says who may view/create/update/delete. Use the
   repo's permission system; don't invent permission names — take them from
   the PRD or ask.
6. **Controller** — thin; eager-load what the resource prints (no N+1);
   wrap multi-table writes in `DB::transaction`.
7. **API resource** — only the fields the FE pages need; formats per the
   contract above; nested resources with `whenLoaded`.
8. **Routes** — in the route file/group from Step A, behind the project's auth
   middleware; `Route::apiResource` when it fits; names consistent with
   existing routes.
9. **Feature tests** — per endpoint: happy path, validation `422` (one
   representative field), authorization (`403`/`401`), and list
   search/sort/pagination. Use factories.
10. **`docs/API.md`** — add or update the resource's section: method + path,
    auth, request fields (type, required, rules), response example, error
    cases, list params. This is the contract the `[FE]` items read in
    `data-patterns.md` Step A — keep it exactly in sync with the code.

`docs/API.md` section shape:

````markdown
## Customers
Auth: session (Sanctum) · Policy: CustomerPolicy

### GET /api/customers
Query: `q` (name/email), `status` (`active`|`inactive`), `page`, `per_page` (≤100),
`sort_by` (`name`|`created_at`|`balance`), `sort_order` (`asc`|`desc`)
Response: Laravel paginator, `data[]` = Customer

### POST /api/customers
Body: `name` string required ≤100 · `email` email required unique · `balance` integer (minor units) 0–9000000000000000
201 → Customer · 422 → `{ message, errors }`

Customer: `{ id, name, email, status, balance, created_at }`
````

## Rich text (HTML fields)

A field filled by a rich-text editor holds HTML that pages render as is
(`v-html`), so it is sanitized **on write**. Plain-text fields are not
sanitized — output escaping keeps them safe.

- Use the sanitizer the project already has (`composer.json`:
  `symfony/html-sanitizer`, `mews/purifier`, `stevebauman/purify`) and copy its
  existing config. None → flag `⚠️ needs symfony/html-sanitizer — please
  confirm` before adding it.
- Sanitize in a cast, so every write path (controller, seeder, import) goes
  through it: `'body' => SafeHtml::class` in `casts()`. If the project already
  sanitizes one field in a model mutator and a second field needs it, move that
  code into the cast and use it for both.
- One limit in three places: the request's `max:`, the sanitizer's max input
  length, and a `mediumText` column (`data-patterns.md` → "Input limits").
- One feature test per field: store a payload with `<script>`, an `on*`
  attribute and a `javascript:` link; assert the saved value has none of them.

```php
// app/Casts/SafeHtml.php — symfony/html-sanitizer
class SafeHtml implements CastsAttributes
{
    public function get(Model $model, string $key, mixed $value, array $attributes): ?string
    {
        return $value;
    }

    public function set(Model $model, string $key, mixed $value, array $attributes): ?string
    {
        if ($value === null) {
            return null;
        }

        $config = (new HtmlSanitizerConfig)
            ->allowSafeElements()
            ->allowLinkSchemes(['http', 'https', 'mailto'])
            ->forceAttribute('a', 'rel', 'noopener noreferrer')
            ->forceAttribute('a', 'target', '_blank') // external links open in a new tab (checklist AE3)
            ->withMaxInputLength(200000); // same as the request's max:

        return (new HtmlSanitizer($config))->sanitize($value);
    }
}
```

`allowSafeElements()` drops every `style` attribute, including the editor's
text alignment. If alignment must survive, allow `style` on block elements and
keep only its `text-align` declaration — never pass `style` through unfiltered.

## Uploads

- Validate type **by content**: `File::types([...])` or `mimes:` both check the
  type detected from the file's bytes (finfo). `extensions:` alone only reads the
  name, so never use it without one of those. Images also get `image` (+
  `dimensions:` if the PRD sets a size).
- Size: `max:` in KB, from `data-patterns.md` → "Input limits" (image 2 MB,
  document 5 MB).
- Store under a generated name (`$file->store(...)`), never the client's
  filename. Private files (IDs, medical documents) go on a non-public disk and
  are served through an authorized endpoint or a temporary signed URL, never as a
  public path (checklist SA7).

## Logs

- Never log request bodies, passwords, tokens, OTPs or personal identifiers:
  no `Log::info($request->all())`, no dumping a model that holds them.
- Models hide secrets from serialization: `$hidden = ['password', 'remember_token', ...]`.
- Audit/activity logs record which fields changed. For sensitive fields they
  store a masked value or none, never the password or token.
- Production runs with `APP_DEBUG=false`, so a 500 returns a generic message and
  the FE shows its own text (checklist S3/SI4). Put this in the deploy notes in
  your response when you add the first `[BE]` item.

## Safety

- **Never run** `migrate:fresh`, `migrate:refresh`, `migrate:reset`,
  `db:wipe`, `db:seed` or anything else that drops/overwrites data on the
  developer's database. Ask before running even `php artisan migrate` there;
  otherwise tell the developer to run it.
- **Before running tests,** check `phpunit.xml` (and `.env.testing` if present):
  `DB_CONNECTION`/`DB_DATABASE` must point to a test database (e.g. `sqlite`
  `:memory:` or a `*_test` database). Tests with `RefreshDatabase` **wipe**
  whatever database they connect to. If it points at the development
  database, don't run the tests — tell the developer.
- Don't install composer packages, publish vendor config or change
  `config/*`, `bootstrap/app.php` or middleware without flagging it first
  (`⚠️ needs <package/config change> — please confirm`). New infrastructure
  (roles/permissions package, Sanctum not installed, queues) is an escalation,
  same as a new FE component.
- **Env keys:** no secrets in code or `docs/API.md`. A key the item introduces
  goes into `.env.example` (empty, or a safe local default like
  `MAIL_MAILER=log`) and is read through `config/*.php` (`config/services.php`
  for third parties) — never `env()` outside config files, which returns `null`
  once config is cached. Add it to the plan's `## Environment` table (SKILL.md
  Step 2).
- Mass assignment only through validated data (`$request->validated()`).

## Validate

Only on the files you created or changed:

- `vendor/bin/pint <your files>` — never bare `pint` (it reformats the whole repo).
- `php artisan route:list --path=<your prefix>` — routes registered with the
  intended middleware.
- `php artisan test --filter=<YourFeatureTest>` (or `vendor/bin/pest <file>`) —
  only after the database check above.
- Static analysis if the repo has it (`vendor/bin/phpstan analyse <files>`);
  report errors outside your files, don't fix them.

Report anything you couldn't run (no test DB, migration not applied) in your
response, and leave the item unchecked until it passes or the developer
accepts it.
