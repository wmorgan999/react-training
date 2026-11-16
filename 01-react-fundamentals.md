# 01 · React Fundamentals

Goal: understand the React ideas behind `web/src/App.jsx` in the homelab project, and see how a React app's load-and-update flow differs from a thick client.

> **About the examples.** The snippets in this lesson (`Greeting`, `CustomerRow`, `Counter`, `Spinner`, `Table`, ...) are small teaching examples. They are **not** code from the repo. Where a concept appears in the real app, an **"In `App.jsx`"** note shows where. [Lesson 04](04-homelab-walkthrough.md) walks through the actual file.

## 0. Coming from thick clients: what's different

If you've built thick-client apps, you already know the loop: on load, **put** data into widgets (`grid.setRows(rows)`, `nameField.setText(x)`); on a button click, **get** values back out of the widgets (`nameField.getText()`) and send them to the server. You change the screen directly, and the widgets themselves hold the state.

React works the opposite way. You never put values into widgets or get them back out. You change **state**, and React redraws the screen to match.

| | Thick client | React app in the browser (the homelab) |
|---|---|---|
| Where it runs | Installed process, long-lived | Downloaded on each visit, runs inside a browser tab |
| Where state lives | In widgets and objects, often plus a server session | In JavaScript variables (React state) in the tab; **lost on refresh** |
| Updating the screen | Imperative: you set widget values | Declarative: you change state, React redraws |
| Reading user input | `get` the value from the widget on click | Already in state (controlled input); there is no "get" |
| Talking to the server | Often a persistent connection or direct database access | Separate, stateless HTTP requests (`fetch`) exchanging JSON; the homelab API has no session |
| Startup | Launch, connect, load, show form | Browser fetches `index.html`, then the JS, which renders an **empty** screen, then fetches data |
| Server-initiated updates | Common (push, events) | None in the homelab; the screen only changes when it asks |
| Deploying a change | Install or update on every machine | Static files on nginx; users get the new version on their next page load |

### How the homelab loads, step by step

1. The browser requests `/`. nginx (in the `web` container) returns `index.html`.
2. `index.html` has an empty `<div id="root">` and a script tag for the JavaScript. (In the Docker build, Vite bundles `src/main.jsx` and everything it imports into that script.)
3. `main.jsx` runs `createRoot(...).render(<App />)`, which calls the `App` function for the first time.
4. **First render:** `customers` is `[]`, so the heading, form and an **empty table** are drawn. The screen appears *before* any data arrives.
5. After that first render, the `useEffect` runs `load()`, which sends `GET /api/customers` (nginx, then the API, then Postgres).
6. The response arrives and `setCustomers(data)` is called. React runs `App` again with the new state and the rows appear.
7. Then the app **idles**. Nothing happens until the user acts. Each action is a new request, then a state change, then a redraw.

The thick-client equivalent of steps 4-6 is a `form_load` that queries the database and fills the grid before showing the window. React has no such moment: it draws first, fetches second, and redraws when data arrives. That's why loading, empty and error states matter (lesson 03 §7).

### What "Add" looks like, side by side

**Thick client:**
```
onClick(Save):
    name  = txtName.getText()        // get
    email = txtEmail.getText()       // get
    server.save(name, email)
    txtName.setText("")              // put
    grid.setRows(server.list())      // put
```

**React (homelab `App.jsx`):**
```
onChange (every keystroke): setName(...) / setEmail(...)    // widgets stay in sync with state
onSubmit:
    name and email are already variables       // nothing to "get"
    POST /api/customers
    setName(''); setEmail('')                  // change state; the inputs follow
    load()  ->  GET list, setCustomers(data)   // change state; the table follows
```

The mental shift: **don't push values into the screen; change state and describe what the screen looks like for that state.** (The HTTP verbs `GET` and `POST` are a separate matter: they're the network calls React makes, not how the UI is updated.)

## 1. The core idea

React is a library for building UIs as a **function of state**:

```
UI = f(state)
```

You don't tell the browser "append a row to the table." You change state, and React re-runs your component function, compares the new output with the old, and updates only what changed in the DOM.

## 2. Components and JSX

A **component** is a function that returns UI. Names start with a capital letter.

```jsx
function Greeting() {
  return <h1>Hello, Wayne</h1>
}
```

That HTML-looking syntax is **JSX**, compiled to JavaScript function calls. Rules worth memorizing:

- A component returns **one root element** (use `<>...</>` fragments to avoid an extra `<div>`).
- JavaScript goes inside `{ }`: `<p>{count + 1}</p>`.
- Attributes use camelCase: `className`, `onClick`, `htmlFor`.
- Inline `style` takes an **object**, hence the double braces: `style={{ color: 'red' }}`.
- Every tag must close: `<input />`.

## 3. Props: data flowing down

Props are a component's inputs. They are read-only.

```jsx
function CustomerRow({ customer }) {
  return (
    <tr>
      <td>{customer.name}</td>
      <td>{customer.email}</td>
    </tr>
  )
}

// usage
<CustomerRow customer={{ name: 'Ada', email: 'ada@example.com' }} />
```

Data flows **down** (parent to child). To send information **up**, the parent passes a function as a prop and the child calls it.

> **In `App.jsx`:** there is no `CustomerRow`. Each row is written inline inside the table's `customers.map(c => <tr key={c.id}>...</tr>)`. `CustomerRow` is what you'd get by extracting that `<tr>` into its own component and passing `c` as a prop; that's exercise 2 in lesson 05.

## 4. State: data that changes

`useState` gives a component memory between renders.

```jsx
import { useState } from 'react'

function Counter() {
  const [count, setCount] = useState(0)   // [current value, setter]
  return <button onClick={() => setCount(count + 1)}>Clicked {count}</button>
}
```

Key rules:

1. **Never mutate state directly.** `count++` or `list.push(x)` won't trigger a re-render. Call the setter with a **new** value or array: `setList([...list, x])`.
2. Calling a setter **schedules** a re-render; the variable doesn't change in the current run. If the next value depends on the previous one, use the functional form: `setCount(c => c + 1)`.
3. State is **per component instance**. Two `<Counter />` elements each have their own count.

> **In `App.jsx`:** four `useState` calls at the top of `App`: `customers`, `name`, `email`, `error`.

## 5. Events

```jsx
<button onClick={handleClick}>Save</button>        // pass the function
<button onClick={() => remove(id)}>Delete</button> // wrap when you need arguments
```

Common mistake: `onClick={handleClick()}` calls it during render. No parentheses unless you're inside an arrow function.

For forms, `e.preventDefault()` stops the browser's default page reload.

> **In `App.jsx`:** `<form onSubmit={add}>` passes the function itself, and `add` begins with `e.preventDefault()`.

## 6. Rendering lists and conditionals

```jsx
{customers.map(c => (
  <tr key={c.id}>...</tr>
))}

{error && <p>{error}</p>}                 // render only if truthy
{loading ? <Spinner /> : <Table />}        // either/or
```

**`key`** must be a stable, unique identifier among siblings (a database id, not the array index if the list can reorder). It lets React match old and new items correctly.

> **In `App.jsx`:** `{error && <p ...>{error}</p>}` is the conditional, and `customers.map(c => <tr key={c.id}>...)` is the list with a database id as the key. There is no `Spinner` or `Table` component; those are illustrative.

## 7. Controlled inputs (forms)

A controlled input's value lives in state; React is the single source of truth.

```jsx
const [name, setName] = useState('')
<input value={name} onChange={e => setName(e.target.value)} />
```

Every keystroke: event fires, setter runs, component re-renders with the new `value`. This is exactly how the name/email fields work in the homelab app.

> **In `App.jsx`:** the two `<input>`s use `value={name}` / `onChange={e => setName(e.target.value)}` (and the same for `email`).

## 8. Effects: talking to the outside world

Rendering must be a pure calculation. Anything else (fetching data, timers, subscriptions) belongs in `useEffect`.

```jsx
useEffect(() => {
  fetch('/api/customers').then(r => r.json()).then(setCustomers)
}, [])        // dependency array
```

The dependency array controls when the effect runs:

| Array | Runs |
|---|---|
| `[]` | once, after the first render |
| `[id]` | after first render and whenever `id` changes |
| *(omitted)* | after **every** render (rarely what you want) |

> **In `App.jsx`:** `useEffect(() => { load() }, [])`, where `load` does the `fetch('/api/customers')`. It runs once, after the first (empty) render. This is step 5 of the load flow in §0.

If the effect starts something ongoing, return a **cleanup** function:

```jsx
useEffect(() => {
  const t = setInterval(tick, 1000)
  return () => clearInterval(t)
}, [])
```

Note: in development, React's `<StrictMode>` runs effects twice on purpose to expose missing cleanup. Don't be alarmed by double requests in dev.

## 9. Rules of hooks

1. Call hooks **only at the top level** of a component (not inside `if`, loops, or nested functions).
2. Call hooks **only from components or other hooks**.
3. Custom hooks are functions whose names start with `use` and that call other hooks. They're how you share logic (e.g. `useCustomers()`).

## 10. Thinking in React (the workflow)

1. Break the UI into a **component hierarchy**.
2. Build a **static version** with props only.
3. Decide the **minimal state** and which component should own it (the lowest common parent of everything that needs it, called "lifting state up").
4. Add **inverse data flow** (callbacks passed down for children to update parent state).

## Check yourself

1. Why does `items.push(x); setItems(items)` not reliably update the UI?
2. What happens if you forget the `[]` in an effect that calls `setState`?
3. When is `key={index}` a bug waiting to happen?
4. Where would you put the shared `customers` state if both a table and a "total count" badge need it?

<details><summary>Answers</summary>

1. It's the same array reference, so React sees "no change." Create a new array: `setItems([...items, x])`.
2. The effect runs after every render, sets state, which re-renders, which runs the effect again: an infinite loop.
3. When items can be reordered, inserted, or deleted: state attached to rows (input text, focus) sticks to the wrong item.
4. In their closest common parent, passing the data (and any update functions) down as props.
</details>

Next: [02 · React vs Angular](02-react-vs-angular.md)
