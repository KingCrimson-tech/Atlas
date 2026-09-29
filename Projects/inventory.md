# Musical Inventory — Technical Deep Dive (Relearn Guide)

> Stack: **Node.js + Express 5 + EJS + PostgreSQL (`pg`)**  
> Pattern: **Server-Rendered MVC (no separate frontend API, no ORM)**  
> Domain: Music shop inventory — `categories` (1) → (N) `items`

Use this file to relearn the whole project in ~20 minutes. It explains *why* each file exists, how a request flows end-to-end, and where the sharp edges are.

---

## 1. The Big Picture

This is a classic **CRUD inventory app** with server-side rendering:

```
Browser <-> Express Routes <-> Controllers <-> pg Pool <-> PostgreSQL
                 |
                 v
            EJS Templates + public/css/style.css
```

There is no React, no REST JSON API, no ORM. Every mutation is an HTML `<form POST>` + `res.redirect()`. Every read is a `pool.query()` + `res.render('*.ejs')`.

Three route mounts in `app.js:19-21`:

| Mount | Router file | Purpose |
|-------|-------------|---------|
| `GET /` | `routes/index.js` | Dashboard: counts + 5 recent items |
| `/categories/*` | `routes/categories.js` | CRUD for categories |
| `/items/*` | `routes/items.js` | CRUD for items |

### Request lifecycle example: `GET /items`

1. `app.js` parses body (`express.urlencoded`, `express.json`), serves `public/` statically.
2. `routes/items.js:5` → `itemsController.getAllItems`.
3. Controller runs SQL `JOIN items + categories` via shared `db/pool.js`.
4. `res.render('items/index', { title, items })` → `views/items/index.ejs` loops `items` into `.items-grid` cards.
5. `partials/header.ejs` + `partials/footer.ejs` wrap every page. `layout.ejs` exists but is **currently unused** — pages are full HTML documents, not `layout + <%- body %>`.

---

## 2. Tech Stack — Why Each Dependency Matters

From `package.json`:

- `express@5.1.0`: routing, middleware, `res.render`, static serving. Note: Express 5 is used, but code is written in Express 4 style (still works).
- `ejs@3.1.10`: templating. `<%= %>` = escaped output, `<%- include() %>` = raw partial include, `<% %>` = control flow (`forEach`, `if`).
- `pg@8.16.3`: raw PostgreSQL driver. No Sequelize/Prisma/Knex. You write SQL strings + `$1, $2` placeholders yourself.
- `dotenv@17.2.3`: loads `DATABASE_URL` and `PORT` from `.env` into `process.env`.
- `express-validator@7.3.0`: **installed but never used**. This is a gap — currently there is zero server-side validation/sanitization.

Scripts:

```json
"start": "node app.js",
"dev": "node --watch app.js",
"seed": "node seed.js"  // broken path — real file is db/seed.js
```

Correct commands:

```bash
npm run dev        # auto-reload via node --watch
node db/initDB.js  # create tables from db/schema.sql
node db/seed.js    # insert demo categories + items
```

---

## 3. Project Structure

```
app.js                  # Express bootstrap: view engine, middleware, routers, 404, listen
routes/
  index.js              # GET / dashboard
  categories.js         # 7 routes for categories CRUD
  items.js              # 7 routes for items CRUD
controllers/
  categoriesController.js  # 7 handlers, raw SQL via pool
  itemsController.js       # 7 handlers, JOINs for category_name
db/
  pool.js               # single shared pg.Pool instance
  schema.sql            # DDL: categories + items + FK + index
  initDB.js             # reads schema.sql with fs, runs pool.query(schema)
  seed.js               # ON CONFLICT DO NOTHING demo data
views/
  index.ejs             # dashboard: stat-cards + recentItems grid
  404.ejs               # catch-all 404 page
  layout.ejs            # UNUSED layout template
  partials/header.ejs + footer.ejs  # actually used on every page
  categories/{index,show,create,edit}.ejs
  items/{index,create,edit}.ejs  # NOTE: detail.ejs is missing (see §7)
public/css/style.css    # hand-rolled CSS: navbar, grids, cards, forms, responsive
```

---

## 4. Database Layer

### 4.1 Connection: `db/pool.js`

```js
const { Pool } = require('pg');
new Pool({ connectionString: process.env.DATABASE_URL, ssl: { rejectUnauthorized: false } })
```

- Single exported `pool` shared by all controllers and `routes/index.js`. This is connection pooling, not one connection per request.
- `ssl.rejectUnauthorized: false` is needed for hosted Postgres (Neon/Supabase/Heroku) but is insecure for strict prod — fine for learning.
- No `DATABASE_URL` → runtime error. You must have `.env`:
  ```
  DATABASE_URL=postgres://user:pass@localhost:5432/inventory_db
  PORT=3000
  ```

### 4.2 Schema: `db/schema.sql`

```sql
CREATE TABLE categories (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL UNIQUE,
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE items (
  id SERIAL PRIMARY KEY,
  name VARCHAR(200) NOT NULL,
  description TEXT,
  price DECIMAL(10,2) NOT NULL,
  quantity INTEGER NOT NULL DEFAULT 0,
  category_id INTEGER NOT NULL,
  brand VARCHAR(100),
  image_url VARCHAR(500),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE CASCADE
);
CREATE INDEX idx_items_category ON items(category_id);
```

Key design decisions to remember:

1. **One-to-Many**: `items.category_id → categories.id`. Every item *must* belong to a category (`NOT NULL`).
2. **`ON DELETE CASCADE`**: deleting a category auto-deletes all its items. The UI even warns: `"Delete this category and all its items?"` in `categories/show.ejs:21`.
3. **`categories.name UNIQUE`**: enables `ON CONFLICT (name) DO NOTHING` in seed.
4. **`idx_items_category`**: speeds up `WHERE category_id = $1` and `JOIN ... ON category_id`, both used on every items page.
5. **`DECIMAL(10,2)` for price**: correct for money (avoids float errors). Rendered as `$<%= item.price %>` in EJS.

ER sketch:

```
categories 1 ────< items N
  id PK ──────── category_id FK (CASCADE)
```

### 4.3 Init + Seed

- `db/initDB.js`: `fs.readFileSync(schema.sql) → pool.query(schema) → pool.end()`. Idempotent? No — running twice errors because `CREATE TABLE` without `IF NOT EXISTS`. Relearn tip: add `IF NOT EXISTS` if you re-run often.
- `db/seed.js`: inserts 6 categories (Guitars, Keyboards, Drums, Wind Instruments, String Instruments, Accessories) + 5 items (Stratocaster, Les Paul, Yamaha P-45, Roland TD-17, Yamaha Sax). Uses `ON CONFLICT DO NOTHING` so re-seeding is safe.

---

## 5. Backend Deep Dive

### 5.1 `app.js` — Bootstrap

```js
app.set("views", path.join(__dirname, "views"))
app.set("view engine", "ejs")
app.use(express.urlencoded({ extended: true })) // HTML forms → req.body
app.use(express.json())                         // (unused now, useful if you add fetch API)
app.use(express.static("public"))               // /css/style.css served directly
app.use('/', indexRouter)
app.use('/categories', categoriesRouter)
app.use('/items', itemsRouter)
app.use((req,res)=> res.status(404).render("404",...)) // must be LAST
```

Order matters: static → routers → 404. `module.exports = app` at the bottom is harmless but `app.listen` runs on import — remember to guard it if you ever write tests (`if (require.main === module)`).

### 5.2 Routes — REST-ish via HTML Forms

Both resources follow the same 7-route convention (no `PUT`/`DELETE` because HTML forms only do `GET`/`POST`):

```
GET  /items/            → list
GET  /items/create      → show create form  (must be BEFORE /:id !)
POST /items/create      → insert + redirect /items
GET  /items/:id         → detail (currently broken — missing view)
GET  /items/:id/edit    → show edit form (pre-filled + category <select>)
POST /items/:id/edit    → update + redirect /items
POST /items/:id/delete  → delete + redirect /items
```

Same for `/categories/*`, except `updateCategory` redirects to `/categories/:id` (detail) instead of the list.

Route-order trap to remember: `/create` is defined before `/:id` in both routers. If reversed, `create` would be captured as `id="create"`.

### 5.3 Controllers — What Each Handler Does

**`itemsController.js`:**

- `getAllItems`: `SELECT items.*, categories.name AS category_name ... JOIN ... ORDER BY items.name`. The `category_name` alias is what `items/index.ejs:25` displays and links to `/categories/:category_id`.
- `getItemById`: same JOIN + `WHERE items.id=$1`. 404 if empty. Renders `items/detail` — **file does not exist**, so this route 500s. Easiest fix: create `views/items/detail.ejs`.
- `getCreateForm` / `getEditForm`: always fetch `SELECT * FROM categories ORDER BY name` to populate the `<select name="category_id">`. Edit also fetches the item; 404 if missing.
- `createItem` / `updateItem`: destructures `{name, description, price, quantity, category_id, brand, image_url}` from `req.body`, uses parameterized `$1..$7` (SQL-injection safe). No validation — empty strings / negative price currently accepted.
- `deleteItem`: `DELETE FROM items WHERE id=$1` + redirect. Delete button in EJS is a `POST` form with `onclick="return confirm(...)"`.

**`categoriesController.js`:** mirrors the above.

- `getCategoryById` does **two queries**: one for the category, one for `SELECT * FROM items WHERE category_id=$1`. Passes both to `categories/show.ejs`, which handles the empty case (`if items.length===0 → "Add one!"`).
- Error responses are inconsistent: categories sends `res.status(500).send({error:...})` (JSON), items sends `send("Internal Server Error")` (text). Pick one if you refactor.

**`routes/index.js` (dashboard):** three queries per page load — `COUNT(*) categories`, `COUNT(*) items`, `JOIN ... ORDER BY created_at DESC LIMIT 5`. Renders `views/index.ejs` with `stat-card`s and `recent-items` grid. Good example of fan-out reads on a single page.

### 5.4 Security Notes (for relearning)

- ✅ Parameterized queries everywhere (`$1`) — no string interpolation SQLi.
- ❌ No `express-validator` usage despite dependency. Add `body('price').isFloat({min:0})` etc. before `createItem`.
- ❌ No auth — anyone can POST delete. The `confirm()` dialog is client-side only.
- ❌ `image_url` is rendered raw into `<img src="">` — stored XSS possible if you allow arbitrary URLs. EJS `<%= %>` escapes HTML but not URL context.
- ❌ No CSRF tokens on delete/edit forms.

---

## 6. Frontend (EJS + CSS) Deep Dive

- **No build step, no JS framework.** Each `.ejs` is a full HTML doc with `<link rel="stylesheet" href="/css/style.css">`.
- **Partials pattern**: `<%- include('../partials/header') %>` (nav) + `<%- include('../partials/footer') %>`. `<%-` = unescaped (needed for HTML). `<%= title %>` = escaped.
- **Forms**: `express.urlencoded({extended:true})` makes `<input name="price">` appear as `req.body.price` (string — Postgres coerces to DECIMAL/INT). Category dropdown: `<select name="category_id">` built from `categories` array; edit view marks current one `selected`.
- **CSS** (`public/css/style.css`, ~350 lines): hand-rolled design system — `.navbar`, `.stat-card`, `.categories-grid/.items-grid` (responsive `repeat(auto-fill, minmax(280px,1fr))`), `.item-card` hover lift, `.btn/.btn-small/.btn-danger/.btn-secondary`, `.form` + `.form-row` 2-col grid, `.error-page`, media queries at 768px/480px. No Tailwind/Bootstrap.

Pages to re-open in order when relearning:

1. `views/index.ejs` — dashboard props: `categoriesCount, itemsCount, recentItems`.
2. `views/categories/index.ejs` → `show.ejs` — master/detail with nested items loop.
3. `views/items/index.ejs` → `create.ejs` → `edit.ejs` — full CRUD form cycle.

---

## 7. Known Gaps / Bugs (Good Relearn Exercises)

1. **Missing `views/items/detail.ejs`**: `GET /items/:id` crashes. Create it (copy `categories/show.ejs` structure, show brand/price/quantity/description + Edit/Delete).
2. **`package.json: seed` path wrong**: `"seed": "node seed.js"` → should be `node db/seed.js`.
3. **`layout.ejs` dead code**: either delete it or refactor all views to use `express-ejs-layouts` / `<%- body %>`.
4. **No validation**: wire `express-validator` into `routes/items.js` + `routes/categories.js`.
5. **`initDB.js` not idempotent**: add `IF NOT EXISTS` to `schema.sql` or `DROP TABLE IF EXISTS` for dev reset.
6. **Inconsistent error format** (JSON vs text) between the two controllers.
7. Typo: `title: "ALl items"` in `itemsController.js:13`.

Fixing #1 + #4 alone will reteach you 80% of the codebase.

---

## 8. How to Run / Relearn It (Cheatsheet)

```bash
# 0. Prereqs: Node 18+, Postgres running
createdb inventory_db

# 1. Env (.env in project root)
DATABASE_URL=postgres://USER:PASS@localhost:5432/inventory_db
PORT=3000

# 2. Install + init + seed
npm install
node db/initDB.js
node db/seed.js

# 3. Run
npm run dev   # http://localhost:3000
```

Click path to relearn: `/` (counts + recent) → `/categories` → `/categories/1` (items in category) → `/items` → `/items/create` (needs categories first) → `/items/:id/edit` → delete flows. Watch terminal + `psql` after each POST to see the SQL effect.

Suggested extension tasks (in difficulty order):

1. Create `items/detail.ejs` + link it.
2. Add `express-validator` checks (name required, price ≥ 0, quantity int ≥ 0).
3. Add search: `GET /items?q=strat` → `WHERE items.name ILIKE $1`.
4. Add pagination: `LIMIT 12 OFFSET $1`.
5. Add `ON DELETE RESTRICT` toggle vs `CASCADE` experiment to feel the FK difference.

---

## 9. One-Paragraph Recall

> Express serves EJS pages for two resources, categories and items, backed by two Postgres tables joined on `category_id` with cascade delete. Routes map HTML GET/POST to controller functions that run parameterized `pg` queries and render EJS or redirect. The dashboard aggregates counts + recent items; category show nests its items; item forms always load categories for the dropdown. Static CSS styles cards/grids/forms. Init via `schema.sql`, demo data via `seed.js`, connection via `DATABASE_URL` pool.

If you can redraw the schema, list the 7 routes per resource from memory, and trace `POST /items/create` from `<form>` → `req.body` → `INSERT` → `redirect /items` → `SELECT + render`, you know this project again.
