# book-store-api

A REST API for a book store, written in late 2018 as a showcase of Node.js,
Express, Sequelize and test-driven development. Users read books, authors
upload them as PDFs, and admins manage users, books and categories.

**Features**

- User authentication (JWT) and role-based authorization
- Users can view books and view/modify their own data
- Authors are users who can create/update their own books and upload PDFs
- Admins are users who can create users, books and categories
- Only admins can change another user's roles (`author`, `admin`)
- Admins can't remove or modify other admin accounts

**Topics showcased**

- `Node.js` (creating modules, using a variety of modules)
- Heavy use of Promises (chaining, controlling the execution path, ...)
- `Express.js` with modular routing
- RESTful API (see the last note under "Notes")
- File upload using `multer`
- An API built around serving JSON objects
- Sequelize: associations, validation, migrations, scopes, models and join queries
- Schemas extracted so that migrations and models share one definition of
  each table's attributes (DRY)
- `PostgreSQL`
- Token-based authentication with Passport and JWT
- A modular authorization system built on Passport and JWT: each route lists
  groups of guards, and a request passes if every guard in any one group passes
  (for example "is an admin" OR "is an author AND is that same user")
- TDD and BDD with automated tests (`mocha`, `chai` with `chai-http`, and
  `factory-girl`)

## Requirements

- Node.js 10. The dependencies date from 2018: `bcrypt` 3 has no build for
  current Node versions and fails to compile on them.
- PostgreSQL on `127.0.0.1:5432`

## Setup

The database settings are in `config/database.js`. Development and test use a
role `test` with password `cf123`:

```sql
CREATE ROLE test LOGIN PASSWORD 'cf123' CREATEDB;
CREATE DATABASE book_store_db_development OWNER test;
CREATE DATABASE database_test OWNER test;
```

Install, create the tables, and make the upload folder (it is git-ignored):

```bash
npm install
NODE_ENV=development npx -p sequelize-cli@5 sequelize db:migrate
NODE_ENV=test npx -p sequelize-cli@5 sequelize db:migrate
mkdir -p uploads/books
```

`sequelize-cli` is not a dependency of the project; `npx` fetches it, and
`.sequelizerc` points it at `db/migrations`.

## Running

The API needs two environment variables: `NODE_ENV` (`development`, `test` or
`production`) and `AUTH_SECRET`, the key that signs the JWTs.

```bash
NODE_ENV=development AUTH_SECRET=change-me node src/app.js
```

It listens on `PORT`, default `3000`. In production the database comes from
`DB_USERNAME`, `DB_PASSWORD`, `DB_NAME` and `DB_HOSTNAME`.

Nothing creates the first admin. Sign one up, then add a row with the same id
to the `admins` table (or `authors` for an author):

```sql
INSERT INTO admins (id, "createdAt", "updatedAt") VALUES (1, now(), now());
```

## Endpoints

| Method | Path | Who |
| --- | --- | --- |
| `POST` | `/auth/signup` | anyone (`username`, `email`, `password`) |
| `POST` | `/auth/login` | anyone; returns `{ token, user }` |
| `GET` | `/books`, `/books/:id` | signed-in users |
| `GET` | `/books/:id/view` | signed-in users; sends the PDF |
| `POST`, `PATCH`, `PUT`, `DELETE` | `/users/:authorId/books[/:id]` | that author, or an admin |
| `POST`, `PATCH`, `PUT`, `DELETE` | `/books[/:id]` | admins (pass `authorId` when creating) |
| `GET` | `/users` | admins |
| `GET` | `/users/:id` | admins, or that user |
| `POST` | `/users` | admins |
| `PATCH`, `PUT` | `/users/:id` | admins (not on other admins), or that user (without changing roles) |
| `DELETE` | `/users/:id` | admins (not on other admins) |
| `GET`, `POST`, `PATCH`, `PUT`, `DELETE` | `/categories[/:id]` | admins |

Send the token as `Authorization: Bearer <token>`. Books are uploaded as
`multipart/form-data` with the PDF in a `bookPDF` field.

## Tests

```bash
NODE_ENV=test AUTH_SECRET=test npm test
```

The suite covers the categories router: 120 tests. The common REST checks
(get all, get one, create, update, delete) are written once as functions that
take a model, so the same tests can be reused for any resource.

**Notes:**

- BDD is applied to the categories router only, to show how BDD is done.
  Under `NODE_ENV=test` the categories router skips authentication, so the
  tests exercise the CRUD behaviour on its own.
- To view a book's PDF, add `/view` to the single-book URL. It is the only
  route that is not RESTful.

## Known issues

- **Send `BookCategories` when uploading a book**, even as an empty field.
  Without it the book row is saved, but the response returns `book: null`
  and the PDF is never renamed to `<id>.pdf`, so `/books/:id/view` cannot
  find it.
- `POST /auth/signup` returns the new user's password hash in its response.
- Passwords are hashed with a bcrypt cost of 2, which bcrypt raises to its
  minimum of 4. Current advice is 10 or more.
- A failed login answers `200` with a plain-text message rather than a `401`.

**Planned**

- Ratings and comments (the associations, schema and model already exist)
- A downloads/views tracker (the `downloads` table already exists)
- Payments
