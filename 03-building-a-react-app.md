# 03 · Building a React App

Goal: go from an empty folder to a React app that talks to a real API, and understand how it gets built and served in production.

## 1. Prerequisites

- Node.js (an LTS release) and npm. Check with `node -v` and `npm -v`.
- A code editor (VS Code) with the ESLint extension.
- Browser with the **React Developer Tools** extension.

## 2. Create the project with Vite

Vite is the build tool and dev server. It's what the homelab `web/` folder uses.

```bash
npm create vite@latest my-app -- --template react
cd my-app
npm install
npm run dev          # dev server with hot reload, usually http://localhost:5173
```

What you get:

```
my-app/
  index.html          # the single HTML page; loads /src/main.jsx
  package.json        # dependencies + scripts (dev, build, preview)
  vite.config.js      # build/dev-server config
  src/
    main.jsx          # entry point: mounts <App /> into #root
    App.jsx           # your root component
```

The key line in `main.jsx`:

```jsx
createRoot(document.getElementById('root')).render(<App />)
```

This is a **single-page application (SPA)**: the server hands out one HTML file, and React draws everything else in the browser.

### Scripts you'll use

| Command | What it does |
|---|---|
| `npm run dev` | Dev server, instant reload |
| `npm run build` | Produces optimized static files in `dist/` |
| `npm run preview` | Serves `dist/` locally to sanity-check a production build |

## 3. Structure as the app grows

Start flat, split when a file gets hard to read. A common shape:

```
src/
  components/      # reusable pieces: CustomerTable.jsx, CustomerForm.jsx
  hooks/           # custom hooks: useCustomers.js
  api/             # fetch wrappers: customers.js
  pages/           # route-level screens: CustomersPage.jsx
  App.jsx          # layout + routes
  main.jsx
```

Rule of thumb: **one component per file**, file name matches the component name.

## 4. Breaking `App.jsx` into components

`App.jsx` changes shape as an app grows. The versions you'll see in this course are **stages**, not competing answers:

| Stage | What `App.jsx` looks like | Where |
|---|---|---|
| 1. The repo today | One big component: state, fetching, form and table together | `homelab/web/src/App.jsx` (lesson 04) |
| 2. Split up | Composition only: calls a hook, renders a form and a table | This section, exercises 2-3 |
| 3. With routing | A small shell with a nav and `<Routes>`; the stage 2 body moves into `pages/CustomersPage.jsx` | §6, exercise 5 |

Take the stage 1 screen that does everything and split it by responsibility:

```jsx
// api/customers.js: network code, no React
export async function listCustomers() {
  const r = await fetch('/api/customers')
  if (!r.ok) throw new Error(`Load failed: ${r.status}`)
  return r.json()
}

export async function createCustomer(data) {
  const r = await fetch('/api/customers', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  })
  if (!r.ok) throw new Error(`Save failed: ${r.status}`)
  return r.json()
}
```

```jsx
// hooks/useCustomers.js: state + effects, no markup
import { useCallback, useEffect, useState } from 'react'
import { listCustomers } from '../api/customers'

export function useCustomers() {
  const [customers, setCustomers] = useState([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState('')

  const reload = useCallback(async () => {
    try {
      setCustomers(await listCustomers())
      setError('')
    } catch (e) {
      setError(e.message)
    } finally {
      setLoading(false)
    }
  }, [])

  useEffect(() => { reload() }, [reload])

  return { customers, loading, error, reload }
}
```

```jsx
// components/CustomerForm.jsx: owns its own input state
import { useState } from 'react'

export default function CustomerForm({ onSubmit }) {
  const [name, setName] = useState('')
  const [email, setEmail] = useState('')

  const handle = async e => {
    e.preventDefault()
    await onSubmit({ name, email })   // parent decides what "submit" means
    setName(''); setEmail('')
  }

  return (
    <form onSubmit={handle}>
      <input placeholder="Name" value={name} onChange={e => setName(e.target.value)} required />
      <input placeholder="Email" type="email" value={email} onChange={e => setEmail(e.target.value)} required />
      <button type="submit">Add</button>
    </form>
  )
}
```

```jsx
// components/CustomerTable.jsx: markup only (the same table the homelab renders inline)
export default function CustomerTable({ customers }) {
  if (!customers.length) return <p>No customers yet.</p>
  return (
    <table border="1" cellPadding="6" style={{ borderCollapse: 'collapse', width: '100%' }}>
      <thead><tr><th>ID</th><th>Name</th><th>Email</th><th>Created</th></tr></thead>
      <tbody>
        {customers.map(c => (
          <tr key={c.id}>
            <td>{c.id}</td><td>{c.name}</td><td>{c.email}</td>
            <td>{c.createdAt ? new Date(c.createdAt).toLocaleString() : ''}</td>
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```

```jsx
// App.jsx (stage 2): composition only
import { useCustomers } from './hooks/useCustomers'
import { createCustomer } from './api/customers'
import CustomerForm from './components/CustomerForm'
import CustomerTable from './components/CustomerTable'

export default function App() {
  const { customers, loading, error, reload } = useCustomers()

  const add = async data => {
    await createCustomer(data)
    await reload()
  }

  if (loading) return <p>Loading…</p>
  return (
    <>
      {error && <p role="alert">{error}</p>}
      <CustomerForm onSubmit={add} />
      <CustomerTable customers={customers} />
    </>
  )
}
```

Notice the pattern: **API code** knows HTTP, the **hook** knows loading/error state, **components** know markup. Each is testable on its own.

## 5. Talking to a backend: the proxy idea

Your React code calls a **relative** URL like `/api/customers`. Something in front must forward `/api/*` to the real API. This avoids CORS problems and hard-coded hostnames.

**In development**, Vite does it:

```js
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': 'http://localhost:8080',   // your backend
    },
  },
})
```

**In production**, a web server does it (nginx in the homelab; see lesson 04).

If you need config values in the browser, put them in `.env` files with the `VITE_` prefix and read them with `import.meta.env.VITE_SOMETHING`. **Everything in the browser bundle is public**: never put secrets there.

### No backend running? Use a stand-in API

If the homelab stack isn't running on your machine (for example you're on a Windows PC and the stack is elsewhere), the proxy has nothing to forward to. The symptom is this error on the page:

```
Unexpected token '<', "<!doctype "... is not valid JSON
```

It happens because Vite answers the unknown `/api/customers` request with its own `index.html`, and your code tries to read that as JSON. The fix is a tiny stand-in server with the same URLs and JSON shapes as the real API. It uses only Node's built-in modules (no installs).

Save this as `mock-api.mjs` in the project root, next to `package.json`:

```js
// A tiny stand-in for the homelab API. Same URLs and JSON shapes, data kept in memory.
// Run:  node mock-api.mjs      (stop with Ctrl+C; data resets on restart)
import http from 'node:http'

const customers = [
  { id: 1, name: 'Ada Lovelace', email: 'ada@example.com', createdAt: new Date().toISOString() },
  { id: 2, name: 'Grace Hopper', email: 'grace@example.com', createdAt: new Date().toISOString() },
]
let nextId = 3

const send = (res, status, body) => {
  res.writeHead(status, { 'Content-Type': 'application/json' })
  res.end(body === undefined ? '' : JSON.stringify(body))
}

http.createServer((req, res) => {
  const url = new URL(req.url, 'http://localhost')
  const one = url.pathname.match(/^\/api\/customers\/(\d+)$/)

  if (req.method === 'GET' && url.pathname === '/api/customers') return send(res, 200, customers)

  if (req.method === 'GET' && one) {
    const c = customers.find(x => x.id === Number(one[1]))
    return c ? send(res, 200, c) : send(res, 404)
  }

  if (req.method === 'POST' && url.pathname === '/api/customers') {
    let raw = ''
    req.on('data', chunk => (raw += chunk))
    req.on('end', () => {
      let data
      try { data = JSON.parse(raw) } catch { return send(res, 400, { error: 'bad json' }) }
      // The real API's database rejects a duplicate email, which surfaces as a 500
      if (customers.some(c => c.email === data.email)) return send(res, 500, { error: 'duplicate email' })
      const saved = { id: nextId++, name: data.name, email: data.email, createdAt: new Date().toISOString() }
      customers.push(saved)
      // The real API's POST response has createdAt null (the database fills it in); the next GET has it
      send(res, 200, { ...saved, createdAt: null })
    })
    return
  }

  send(res, 404, { error: 'not found' })
}).listen(8080, () => console.log('Mock API on http://localhost:8080'))
```

To use it:
1. Make sure `vite.config.js` has the proxy from the dev-server example above (`'/api': 'http://localhost:8080'`).
2. In a **second terminal**, run `node mock-api.mjs` and leave it running.
3. **Restart** `npm run dev`. Changes to `vite.config.js` only apply after a restart. Then refresh the page.

Notes:
- It mirrors the real API's quirks on purpose: a duplicate email returns **500**, and the POST response has `createdAt: null` (the next list request has the date). That's why the homelab UI re-fetches the list after a save.
- Data lives in memory and resets when you restart the script.
- It uses port 8080, the same as the real API, so **stop it before starting the real stack**. Nothing else has to change.
- Try the states from §7 with it: stop the script to see the error state, change the starter `customers` array to `[]` and restart it to see the empty state.

## 6. Routing (multiple pages in an SPA)

```bash
npm install react-router-dom
```

This is **stage 3** of `App.jsx` (see the table in §4). The stage 2 `App` body moves, unchanged, into a page component, and `App.jsx` becomes a routing shell.

```jsx
// main.jsx: wrap the app in the router
import { createRoot } from 'react-dom/client'
import { BrowserRouter } from 'react-router-dom'
import App from './App.jsx'

createRoot(document.getElementById('root')).render(
  <BrowserRouter><App /></BrowserRouter>
)
```

```jsx
// pages/CustomersPage.jsx: the stage 2 App body, moved here
import { useCustomers } from '../hooks/useCustomers'
import { createCustomer } from '../api/customers'
import CustomerForm from '../components/CustomerForm'
import CustomerTable from '../components/CustomerTable'

export default function CustomersPage() {
  const { customers, loading, error, reload } = useCustomers()

  const add = async data => {
    await createCustomer(data)
    await reload()
  }

  if (loading) return <p>Loading…</p>
  return (
    <>
      {error && <p role="alert">{error}</p>}
      <CustomerForm onSubmit={add} />
      <CustomerTable customers={customers} />
    </>
  )
}
```

```jsx
// App.jsx (stage 3): routing shell only
import { Routes, Route, Link } from 'react-router-dom'
import CustomersPage from './pages/CustomersPage'

export default function App() {
  return (
    <>
      <nav><Link to="/">Customers</Link></nav>
      <Routes>
        <Route path="/" element={<CustomersPage />} />
      </Routes>
    </>
  )
}
```

Add more pages the same way. In exercises 5 and 7 you'll add routes like these (each needs its own page component):

```jsx
<Route path="/customers/:id" element={<CustomerDetail />} />   {/* read the id with useParams() */}
<Route path="/reports" element={<ReportsPage />} />
```

Production gotcha: refreshing on `/reports` asks the server for a file called `/reports`, which doesn't exist. The server must fall back to `index.html` for unknown paths. The homelab `nginx.conf` already does (`try_files $uri /index.html`).

## 7. Handling the three states of any request

Every data screen has **loading**, **error**, and **success** (plus **empty**). Beginners usually handle only success. Make it a habit:

```jsx
if (loading) return <Spinner />
if (error)   return <ErrorMessage text={error} />
if (!items.length) return <p>No customers yet.</p>
return <Table items={items} />
```

## 8. Testing basics

```bash
npm install -D vitest jsdom @testing-library/react @testing-library/user-event @testing-library/jest-dom
```

**Version gotcha: match Vitest to the Vite you actually have.** Run `npm ls vite` first.

| Your Vite | Install | Why |
|---|---|---|
| 6.4 or newer (a fresh `npm create vite@latest` project, currently Vite 8) | `vitest` (the latest, as in the command above) | The latest Vitest needs Vite 6.4+ |
| 5.x (the homelab's `web/` pins **5.4.11**) | `vitest@2` | The latest Vitest fails there with `ERESOLVE ... peer vite` |

```bash
# Vite 5 projects only (e.g. homelab/web):
npm install -D vitest@2 jsdom @testing-library/react @testing-library/user-event @testing-library/jest-dom
```

**Do not use `vitest@2` in a Vite 8 project.** Vitest 2 brings its own Vite 5 underneath, so `npm ls vite` shows **two** Vite versions. Your tests then run on the wrong one, and you get `ReferenceError: React is not defined`. If `npm ls vite` ever shows more than one major version, that's the mismatch: fix it with `npm uninstall vitest` then `npm install -D vitest@latest` (or `vitest@2` for Vite 5).

Don't reach for `--force` or `--legacy-peer-deps`; they hide the mismatch instead of fixing it.

Tell Vitest to use a simulated browser. **Keep the React plugin** and add a `test` block, in `vite.config.js`. This is the whole file:

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],                  // required: this is what makes JSX work in tests too
  test: { environment: 'jsdom' },
})
```

If your test fails with **`ReferenceError: React is not defined`**, Vitest is compiling your JSX without the React plugin's settings. The usual causes:
- **Mismatched Vitest and Vite versions** (the most common). See the version table above, and check that `npm ls vite` shows a single Vite version.
- `plugins: [react()]` is missing from the config Vitest is actually reading.
- A separate `vitest.config.js` (or `.ts`) exists. Vitest uses it **instead of** `vite.config.js`, so either delete it or put the plugin and `test` block in it.
- JSX is inside a `.js` file. Rename it to `.jsx`.

A component test describes what a user sees and does, not implementation details. This one tests the `CustomerForm` from §4 (it finds the inputs by the placeholders `Name` and `Email`):

```jsx
// src/components/CustomerForm.test.jsx
import { test, expect, vi } from 'vitest'
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import CustomerForm from './CustomerForm'

test('submits the form values', async () => {
  const onSubmit = vi.fn()
  render(<CustomerForm onSubmit={onSubmit} />)

  await userEvent.type(screen.getByPlaceholderText('Name'), 'Ada')
  await userEvent.type(screen.getByPlaceholderText('Email'), 'ada@example.com')
  await userEvent.click(screen.getByRole('button', { name: 'Add' }))

  expect(onSubmit).toHaveBeenCalledWith({ name: 'Ada', email: 'ada@example.com' })
})
```

Run it with `npx vitest run` (or add `"test": "vitest run"` to the `scripts` in `package.json` and use `npm test`).

## 9. Building and shipping

`npm run build` outputs plain static files (HTML, JS, CSS) into `dist/`. Any static file host can serve them. The homelab uses a **multi-stage Docker build**:

1. Stage 1 (Node image): `npm install` and `npm run build`.
2. Stage 2 (nginx image): copy only `dist/` and the nginx config. The final image has no Node, no source, and is small.

## 10. Common beginner pitfalls

| Symptom | Likely cause |
|---|---|
| UI doesn't update after changing a list | Mutated state instead of setting a new value |
| Request fires endlessly | `useEffect` missing `[]`, or a dependency that changes every render |
| `fetch` works in prod but 404s in dev | No Vite dev proxy for `/api` |
| `Unexpected token '<', "<!doctype "... is not valid JSON` | `/api` isn't reaching a backend (no proxy, or nothing listening), so Vite returned `index.html`; see the stand-in API in §5 |
| `Identifier ... has already been declared` / `A module cannot have multiple default exports` | Code was pasted on top of existing code instead of replacing the file; each file should have each import and `export default` once |
| `ReferenceError: React is not defined` in a test | Vitest and Vite versions don't match, or the React plugin is missing; see §8 |
| Page blank, console error | Look at the browser console first; it names the file and line |
| Refresh on a route gives 404 | Server missing the `index.html` fallback |
| Input "won't type" | `value` set without a matching `onChange` |

## Check yourself

1. Why does the app call `/api/customers` instead of `http://localhost:8080/api/customers`?
2. What belongs in a custom hook versus in a component?
3. What does the second stage of the homelab `web/Dockerfile` leave out, and why does that matter?

<details><summary>Answers</summary>

1. Relative URLs work in every environment: a proxy (Vite in dev, nginx in prod) forwards them to the API. It also avoids CORS and avoids baking hostnames into the bundle.
2. Hooks hold state and side effects (loading, errors, fetching). Components hold markup and wire things together.
3. It leaves out Node and the source code. The runtime image only contains nginx plus static files, so it's smaller and has less attack surface.
</details>

Next: [04 · Homelab Walkthrough](04-homelab-walkthrough.md)
