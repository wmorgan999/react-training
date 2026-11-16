# 04 · Homelab Walkthrough: What We Built

This lesson maps the concepts from lessons 01-03 onto the real code in the `wmorgan999/homelab` repository. File paths are relative to the repo root. The training files live **outside** the repo on purpose.

## 1. The big picture

```
                         ┌──────────────────────────────┐
 Browser ──► :80  ──►    │ web  (nginx + React build)   │
                         └───────────┬──────────────────┘
                                     │  /api/*  (proxy)
                                     ▼
                         ┌──────────────────────────────┐
                         │ api  (Spring Boot, :8080)    │──► Postgres (customers)
                         └───────────┬──────────────────┘
                                     │ publishes "customer-events"
                                     ▼
                         ┌──────────────────────────────┐
                         │ Kafka (:9092)                │
                         └───────────┬──────────────────┘
                                     ▼
                         ┌──────────────────────────────┐
                         │ consumer (Spring Boot)       │──► Postgres (customer_audit)
                         └──────────────────────────────┘

 reports (FastAPI, :8000) ──► reads customers + customer_audit from Postgres
 kafka-ui (:8081)         ──► browse Kafka topics in a browser
```

| Folder | Role | Tech |
|---|---|---|
| `web/` | The UI you've been studying | React 18, Vite 5, nginx |
| `api/` | REST API for customers; publishes events | Java 21, Spring Boot 3.5, JPA, Flyway |
| `consumer/` | Listens to events, writes an audit trail | Spring Boot, Spring Kafka |
| `reports/` | Read-only reporting endpoints | Python, FastAPI, SQLAlchemy |
| `db/init/` | Creates the table and seeds two customers on first Postgres start | SQL |
| `db/import_quotes.py` | Standalone script that loads villa quote CSVs into a separate `villa_quotes` table. **Not** part of the customers app | Python |
| `scripts/backup.sh` | Dumps the database with `pg_dump` and prunes old backups | Bash |
| `docker-compose.yml` | Runs everything together | Docker Compose |

> `db/import_quotes.py` is not used by the React app, the API, or the Kafka flow, so the rest of this lesson ignores it. Its `--create-table` option refers to `V3__create_villa_quotes.sql` and `V4__add_villa_name_to_quotes.sql`, which are not in the repo as of commit `74ea021` (the repo state this lesson was last checked against).

React only talks to **one** thing: the API, through `/api/customers`.

## 2. The API contract the frontend depends on

Defined in `api/src/main/java/com/lab/api/CustomerController.java`:

| Method | URL | Body | Returns |
|---|---|---|---|
| GET | `/api/customers` | none | array of customers |
| GET | `/api/customers/{id}` | none | one customer, or 404 |
| POST | `/api/customers` | `{ "name": "...", "email": "..." }` | the saved customer |

A customer looks like:

```json
{ "id": 1, "name": "Ada Lovelace", "email": "ada@example.com", "createdAt": "2026-10-20T13:00:00Z" }
```

There is **no** update or delete endpoint yet. Anything you want the UI to do beyond list/add/get-one needs a backend change first (see exercises).

Why this matters: the frontend is only as capable as the API contract. When you design a screen, write down the request and response shapes first.

## 3. `web/src/main.jsx` and `index.html`

`index.html` contains `<div id="root">` and one script tag pointing at `/src/main.jsx`. `main.jsx` mounts `<App />` into that div. That's the entire bootstrap. Everything else is React.

## 4. `web/src/App.jsx`, section by section

### State

```jsx
const [customers, setCustomers] = useState([])
const [name, setName] = useState('')
const [email, setEmail] = useState('')
const [error, setError] = useState('')
```

Four pieces of state: the list from the server, the two form fields (controlled inputs, lesson 01 §7), and an error message string.

### Loading data

```jsx
const load = () =>
  fetch('/api/customers')
    .then(r => r.json())
    .then(setCustomers)
    .catch(e => setError(String(e)))

useEffect(() => { load() }, [])
```

`load` fetches the list and stores it in state. The effect runs it **once** after the first render (the `[]`, lesson 01 §8). Setting state re-renders the component, and the table fills in.

### Adding a customer

```jsx
const add = async e => {
  e.preventDefault()                       // stop the browser's form reload
  const r = await fetch('/api/customers', { method: 'POST', ... body: JSON.stringify({ name, email }) })
  if (!r.ok) { setError('Save failed: ' + r.status); return }
  setName(''); setEmail(''); setError('')  // clear form
  load()                                   // re-fetch the list
}
```

After a successful save, the code **re-fetches the whole list** instead of appending the response. That's a simple, reliable pattern: the server stays the source of truth. (The POST response's `createdAt` will be empty, because the database fills it in after the insert, which is one more reason re-fetching is safe.)

### Rendering

- `{error && <p ...>{error}</p>}`: conditional rendering (lesson 01 §6).
- `<form onSubmit={add}>` with two controlled `<input>`s: `required` and `type="email"` give free browser validation.
- `customers.map(c => <tr key={c.id}>...)`: list rendering with a stable `key`.
- `new Date(c.createdAt).toLocaleString()`: converts the ISO timestamp string into the viewer's local time.

## 5. How a click on "Add" travels through the whole system

1. **Browser (React):** `add()` sends `POST /api/customers` with JSON.
2. **nginx** (`web/nginx.conf`): the `location /api/` block forwards to `http://api:8080/api/`. `api` is the **Docker Compose service name**, resolved on the compose network.
3. **Spring controller:** `create()` calls `repo.save(c)`, which inserts a row into `customers`.
4. **Kafka publish:** still in `create()`, it sends a `CUSTOMER_CREATED` JSON message to topic `customer-events`. If publishing fails, it only logs a warning and still returns the saved customer (so the audit trail can miss events; a design trade-off worth knowing).
5. **Consumer** (`AuditListener`): receives the message and inserts a row into `customer_audit`.
6. **Response back:** the API returns the saved customer; React clears the form and calls `load()`, which repeats `GET /api/customers` and re-renders the table.
7. **Reports service** can now count the new customer and show the audit event via `/reports/summary` and `/reports/audit`.

## 6. `web/Dockerfile` and `nginx.conf`

```
Stage 1: node:22-alpine   → npm install → npm run build  (creates /src/dist)
Stage 2: nginx:1.27-alpine → copy dist/ + nginx.conf
```

nginx has two jobs:
- `/api/` goes to the Java API.
- Everything else serves static files, falling back to `index.html` (`try_files $uri /index.html`) so client-side routes survive a refresh.

## 7. Running it

**Whole stack in Docker:**

```bash
cp .env.example .env          # then edit the password
docker compose up --build
```

| URL | What |
|---|---|
| http://localhost | the React app |
| http://localhost:8080/api/customers | API directly |
| http://localhost:8000/docs | FastAPI's auto-generated docs for the reports service |
| http://localhost:8081 | Kafka UI (watch `customer-events`) |

**Frontend only, with hot reload** (backend still in Docker): the repo's `vite.config.js` has **no proxy**, so `npm run dev` will request `/api/...` from Vite itself and fail. Add this to `web/vite.config.js` for local development:

```js
export default defineConfig({
  plugins: [react()],
  server: { proxy: { '/api': 'http://localhost:8080' } },
})
```

Then `cd web && npm install && npm run dev`, and open http://localhost:5173. (To keep the repo untouched, make this change on a branch or in your own copy.)

## 8. Honest code review: gaps worth learning from

These aren't criticisms of the lab; they're exactly the kind of improvements that teach good React habits. Each maps to an exercise.

| # | Observation | Why it matters |
|---|---|---|
| 1 | `load()` never checks `r.ok`. If the API returns an error, the JSON body is an **object**, not an array, and `customers.map` will throw and blank the page. | Always check `response.ok` before parsing as data. |
| 2 | `add()` has no `try/catch`. A network failure becomes an unhandled promise rejection with no message in the UI. | Every `await fetch` can throw. |
| 3 | Adding a duplicate email hits the table's `UNIQUE` constraint; the controller doesn't translate that, so the UI most likely shows a generic "Save failed: 500". | Good errors need cooperation from the backend (e.g. 409 Conflict) and the UI. |
| 4 | No loading or empty state. | Users see a blank table and can't tell "loading" from "no data". |
| 5 | `useEffect(() => { load() }, [])` calls a function re-created every render. ESLint's `react-hooks/exhaustive-deps` rule would warn. | Learn the idiom: move the fetch inside the effect, or use `useCallback`. |
| 6 | Everything is in one component with inline styles. | Fine for a lab; lesson 03 §4 shows the split. |
| 7 | No Vite dev proxy (see §7). | Dev and prod behave differently until fixed. |
| 8 | No update/delete endpoints. | The UI can't offer edit/delete until the API does. |

## Check yourself

1. Trace what happens, file by file, when a user adds a customer.
2. Why does the React code use `/api/customers` while the API listens on port 8080?
3. What would you change on the backend so the UI could show "That email is already registered"?

<details><summary>Answers</summary>

1. See §5. `App.jsx` → `nginx.conf` → `CustomerController.create` → Postgres + Kafka → `AuditListener` → back to `App.jsx`'s `load()`.
2. nginx (in production) or Vite's proxy (in dev) forwards `/api` to the API, so the browser only ever talks to one origin.
3. Catch the unique-constraint violation in the controller (or a `@ControllerAdvice`) and return **409 Conflict** with a message; then have the frontend read the status and message.
</details>

Next: [05 · Exercises](05-exercises.md)
