# 05 · Exercises

Work on **your own copy or a branch** of the homelab repo so the original stays clean. Do them in order; each builds on the last. Difficulty: ★ easy, ★★ medium, ★★★ harder. "(backend)" means the Java API or another service must change too.

Run the backend with `docker compose up --build` (see lesson 04 §7) and use the Vite dev server with the proxy added.

## Part A: React fundamentals on the existing screen

### 1 ★ Make the app robust (loading, errors, empty)
- Add a `loading` state. Show "Loading…" until the first fetch finishes.
- In `load()`, check `r.ok` and throw if not, so an API error can't reach `setCustomers`.
- Wrap `add()` in `try/catch/finally`.
- Show "No customers yet." when the list is empty.

*Test it:* stop the `api` container (`docker compose stop api`) and reload. You should see a clear error, not a blank page.

### 2 ★ Split into components
Extract `CustomerForm` and `CustomerTable` from `App.jsx` (lesson 03 §4). `App` should end up only holding the customers state and passing props/callbacks.

*Self-check:* can you explain which component owns which state and why?

### 3 ★★ Custom hook + API module
Create `src/api/customers.js` and `src/hooks/useCustomers.js` exactly as in lesson 03, and make `App` use them. Delete the now-unused code from `App.jsx`.

### 4 ★★ Client-side search and sort
Add a search box that filters rows by name or email as you type, and clickable column headers that sort. Keep the filter text and sort column in state, and **derive** the visible rows during render (don't store the filtered list as separate state).

## Part B: Building more of the app

### 5 ★★ Add routing and a detail page
- Install `react-router-dom`; wrap the app in `BrowserRouter`.
- Create `/` (list) and `/customers/:id` (detail) pages.
- The detail page reads the id with `useParams()` and loads it from the **existing** `GET /api/customers/{id}`. Handle the 404 case ("Customer not found").
- Link each name in the table to its detail page.

### 6 ★★★ (backend) Delete and edit
- API: add `DELETE /api/customers/{id}` and `PUT /api/customers/{id}` to `CustomerController`; publish `CUSTOMER_DELETED` / `CUSTOMER_UPDATED` events like `create()` does.
- UI: a Delete button per row (with a confirm step), and an Edit form on the detail page.
- Optional: handle the duplicate-email case. Make the API return **409** and show "That email is already registered."

### 7 ★★ A Reports page using the FastAPI service
The `reports` service (port 8000) already has `/reports/summary`, `/reports/signups-by-day` and `/reports/audit`. The nginx config only proxies `/api/`, so:

1. Add a second `location /reports/` block to `web/nginx.conf` that proxies to `http://reports:8000/reports/`.
2. Add the matching `'/reports': 'http://localhost:8000'` entry to the Vite dev proxy.
3. Build a `/reports` page showing the summary numbers, a signups-per-day table, and the latest audit events.
4. Bonus: add a small chart (a library like Recharts, or hand-built SVG bars).

*What you practice:* a second data source, a second proxy rule, and reusing the loading/error pattern.

## Part C: Compare and consolidate

### 8 ★★★ Build the same screen in Angular
Using the Angular CLI, build the customer list + add form against the **same** API (lesson 02 has the concept mapping).
- Use `HttpClient`, signals, and a service for the API calls.
- Add the same dev proxy (`proxy.conf.json`).
- Write down: what felt easier, what felt harder, and how many lines/files each version took. This is your own React-vs-Angular data.

### 9 ★★ Tests
Using Vitest and React Testing Library (lesson 03 §8):
- Test `CustomerForm` calls `onSubmit` with the typed values.
- Test the list shows "No customers yet." for an empty array.
- Stub `fetch` and test that an API failure shows the error message.

### 10 ★★★ Full-stack capstone
Pick one feature and ship it end to end, touching every layer:
**"Customer notes."** Table in Postgres via a new Flyway migration (`V3__...sql` in `api/src/main/resources/db/migration`), JPA entity + endpoints in the API, a Kafka event, an audit row, and a notes panel on the React detail page.

Write a short README section describing the data flow before you start coding. If you can draw the arrows between the browser, nginx, API, Postgres, Kafka, and consumer from memory, you understand the system.

## Part D: MCP and agent harnesses (lesson 06)

### 11 ★★★ Wrap the homelab API as an MCP server
Build a small MCP server so an AI app can work with your customer data.

1. **Design first.** On paper, list the tools you'll expose, their parameters, what each returns, and which are read vs write. Aim for 3-4 tools (lesson 06 §8 has a starting sketch). Write each description as if explaining it to a new colleague.
2. **Build it** with the Python or TypeScript MCP SDK (lesson 06 §9), calling `http://localhost:8080/api/customers`. Cap list output, return readable error text, keep it stdio, and log to stderr only.
3. **Test by hand** with the MCP Inspector before connecting anything. Call every tool, including failure cases (unknown id, duplicate email, API container stopped).
4. **Connect it** to an MCP-capable app and ask natural questions: "How many customers do we have?", "Add Grace Hopper, grace@example.com", "Who signed up most recently?" Watch which tool it picks and why.
5. **Break it on purpose.** Rename a tool to something vague, or make two tools overlap, and see how the model's choices change.

*Stretch goals:*
- Add `report_summary()` using the reports service (read-only).
- Put `add_customer` behind the host's approval prompt and observe what changes.
- Check the Kafka UI (http://localhost:8081): did the agent's write produce a `CUSTOMER_CREATED` event and an audit row, like a UI-created one?
- Write a 5-line loop of your own (lesson 06 §2 pseudocode) with a step limit and a log of every tool call.

*What you practice:* tool design, writing descriptions for a model, safe write tools, and seeing the host/harness decide when to call a tool.

## Part E: TypeScript (lesson 07)

### 12 ★★ Convert the app to TypeScript
Work in your own copy of `training-react-myapp` on a branch, starting from the plain JavaScript version (the finished exercise 6 code).

1. `npm install -D typescript`, add `tsconfig.json` (lesson 07 §3) and the `typecheck` script. Run `npm run typecheck` and confirm it passes with nothing to check yet.
2. Create `src/types.ts` with `Customer` and `CustomerInput`, mirroring the Java entity (`createdAt` is `string | null`).
3. Rename and type the files **one at a time**, running `npm run typecheck` after each: `api/customers`, then the hooks, then `CustomerForm`, `CustomerTable`, the pages, `main`, and last the tests. Update `index.html` for `main.tsx`.
4. Model the detail page's state as a **discriminated union** (lesson 07 §4) and fix the page so `state.customer` is only used when `status === 'ready'`.
5. Break it on purpose: misspell a field, omit a required prop, pass the wrong type. Read each error message and fix it.

*Self-check:* `npm run typecheck`, `npm test` and `npm run build` all pass, and you can explain why `npm test` can pass while `typecheck` fails.

*Stretch goals:*
- Add `springdoc-openapi-starter-webmvc-ui` to the Java API, open its `/v3/api-docs`, and generate the `Customer` type with `openapi-typescript` instead of writing it by hand. Compare it with yours.
- Add a `zod` schema for `Customer`, validate the response in `listCustomers()`, and see what happens when the API sends a bad shape.
- Turn on `noUncheckedIndexedAccess` in `tsconfig.json` and fix what it flags.

*What you practice:* mapping server entities to client types, typing props and state, narrowing, and treating the type checker as part of the build.

## Self-assessment checklist

- [ ] I can explain state vs props and why state is never mutated directly.
- [ ] I can explain what the `[]` in `useEffect` does and what happens without it.
- [ ] I handle loading, error, empty, and success in every data screen.
- [ ] I can explain why the app uses relative `/api` URLs and how that works in dev vs production.
- [ ] I can trace a request from button click to database to Kafka and back.
- [ ] I can say, in my own words, when I'd choose React and when Angular.
- [ ] I can explain what a harness does that the model can't, and draw its loop.
- [ ] I can say when I'd use an MCP server versus a plain function tool, and name three design rules for good tools.
- [ ] I can map a JPA entity to a TypeScript type, and explain why `response.json()` is not type-safe at runtime.
- [ ] I can type component props and `useState`, and use a discriminated union to model loading / ready / error.
