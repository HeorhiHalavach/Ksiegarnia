# Księgarnia — an online bookshop assembled from parts, not from a framework

A full-stack bookshop: a paginated catalogue with search and category filters, user
accounts, a wishlist, orders, and an admin panel that adds books with cover uploads.
**Express + MySQL + EJS** — no ORM, no front-end framework, no scaffolding. Every layer is
hand-written, and that is the point of the project: the request path from URL to SQL is
visible end to end.

**68 tests** in 8 files (Jest + supertest) cover the models, the routes and the auth
service.

---

## The request path

```
routes/          → controllers/        → services/         → models/         → MySQL
routes/books.js    bookController.js     bookService.js       Book.js           books
routes/auth.js     authController.js     authService.js       User.js           users
routes/users.js    userController.js     userService.js       Wishlist.js       wishlist
                                                              Orders.js         orders
```

Four layers, one job each. A route never touches SQL; a model never touches `req`.
`middlewares/` holds the three things that cut across all of them: session→user
attachment, the two access guards, and the error handler.

---

## What it does

| Feature | Detail |
|---|---|
| Catalogue | 12 books per page, newest first, `LIMIT/OFFSET` pagination with a total count |
| Search | by title **or** author, one `LIKE` query |
| Sorting | by views, price or discounted price, ascending or descending |
| Categories | filter by genre, paginated the same way |
| Accounts | registration and login, **bcrypt** hashes at cost 10 |
| Wishlist | per user, stored in its own table |
| Orders | per user, resolved back to books in one `IN (...)` query |
| Admin panel | add books with a cover image upload (**multer**) |
| Logging | **winston** to file plus a request line for every call |
| Production build | **esbuild** bundles and minifies the browser JS and CSS; `compression` only outside development |

---

## Six decisions worth pointing at

**1. Every query is parameterised.** All database access goes through
`connection.execute(sql, [values])` — values never reach the SQL string, so injection has no
surface:

```js
const sql = 'SELECT * FROM books WHERE id = ?;';
const [rows] = await connection.execute(sql, [id]);
```

**2. Where a placeholder is impossible, there is a whitelist.** A column name cannot be a
`?` parameter, so `ORDER BY` is the one place user input would have to be interpolated. It is
checked against a closed list first, and an unknown value throws instead of being
interpolated:

```js
const validSortBy = ['views', 'price', 'superprice'];
if (!validSortBy.includes(sortBy)) throw new Error("Invalid sort column");
```

That is the difference between a guard and a hope: the query is not built at all unless the
column name is one of three known strings.

**3. The password hash never leaves the model.** `models/User.js` owns both `bcrypt.hash` on
create and `bcrypt.compare` on login. Nothing above it can accidentally read or log a hash —
and the test asserts the cost factor, so silently dropping it to a cheaper one fails the
suite.

**4. The same guard answers browsers and API clients differently.** `ensureAuthenticated`
looks at what the caller accepts: a fetch call gets `401 {success: false}`, a browser gets a
redirect to the login page. `ensureAdmin` is separate and returns `403` — being logged in and
being an admin are two different questions.

**5. The session cookie is locked down by default.** `httpOnly` so scripts cannot read it,
`sameSite: 'lax'` against CSRF, and `secure` switched on by `NODE_ENV === 'production'` — the
one flag that must not be on in local development, because an HTTPS-only cookie over
`http://localhost` simply never arrives.

**6. Uploads are filtered before they are stored, not after.** multer rejects anything whose
MIME type does not start with `image/`, caps files at 5 MB, and renames every upload to
`user-<id>-<timestamp><ext>` — so a client-supplied filename never becomes a path on disk.

---

## Tests

```bash
npm test
```

| File | Tests | What it pins |
|---|---|---|
| `__tests__/models/Book.test.js` | 13 | pagination maths, search, sort whitelist, single lookups |
| `__tests__/routes/users.test.js` | 20 | the whole user surface: wishlist, orders, profile |
| `__tests__/models/User.test.js` | 9 | registration, login, that bcrypt is called with cost 10 |
| `__tests__/models/Wishlist.test.js` | 7 | add, remove, list per user |
| `__tests__/models/Orders.test.js` | 6 | creating an order and reading it back |
| `__tests__/routes/books.test.js` | 6 | catalogue and detail routes over HTTP |
| `__tests__/routes/auth.test.js` | 5 | login and registration, right password and wrong |
| `__tests__/services/authService.test.js` | 2 | the service layer between route and model |

Route tests drive the app over HTTP with **supertest**, so validation, session handling and
error responses are exercised together with the logic instead of being mocked away.

---

## Run it

```bash
npm install
cp .env.example .env      # then fill it in
npm run dev               # nodemon, development mode
```

`.env` needs:

```env
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=
DB_NAME=ksiegarnia
SESSION_SECRET=
PORT=3000
NODE_ENV=development
```

Production:

```bash
npm run build:prod        # esbuild: bundle + minify JS and CSS
NODE_ENV=production npm start
```

⚠️ **The database schema is not in the repository.** Four tables are needed — `books`,
`users`, `wishlist`, `orders` — and their columns are only implied by the queries in
`models/`. See the limits below.

---

## Honest limits

Written as coursework, and it carries coursework debt:

- **no schema and no migrations.** You cannot clone this and run it: the tables have to be
  created by reading the queries. A `schema.sql` is the single most useful thing missing;
- **`sequelize` sits in `package.json` and is never imported.** The models are hand-written
  SQL — the dependency is left over from an early direction and should be dropped;
- **two error handlers, and the wrong one wins.** An inline `app.use((err, ...))` that sends
  a plain 500 is registered *before* `errorHandler`, so the real handler never runs. Express
  takes the first one that matches; the fix is deleting the inline one;
- **`dotenv.config()` runs twice** — once in `config/db.js` and once in `app.js`. Harmless,
  but it is the kind of duplication that hides which file owns configuration;
- **built bundles are committed.** `public/js/dist/` and `public/css/dist/` are listed in
  `.gitignore` but were added before the rule existed, and `.gitignore` never applies to
  files that are already tracked — they need `git rm --cached`;
- **comments and log lines are in Russian**, while the views are in Polish. Fine for a
  university deadline, wrong for a shared codebase.
