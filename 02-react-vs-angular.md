# 02 · React vs Angular

Goal: be able to explain the difference in philosophy, read both, and pick one for a given project. Examples use the same feature as the homelab app (list customers, add a customer).

## 1. The one-sentence difference

- **React** is a **library** for rendering UI. You assemble the rest (routing, HTTP, forms, state, build tooling) from ecosystem choices.
- **Angular** is a **framework**: routing, HTTP client, forms, dependency injection, testing setup and CLI come as one opinionated, integrated package.

Neither is "better." It's a trade between flexibility (React) and consistency out of the box (Angular).

## 2. Side-by-side

| Topic | React | Angular |
|---|---|---|
| Language | JavaScript or TypeScript (TS optional) | TypeScript (effectively required) |
| Templates | JSX: markup written inside JavaScript | HTML-like templates with their own syntax, separate from or inline in the component class |
| Component shape | Functions (hooks for state/effects) | Classes with decorators (`@Component`) |
| Local state | `useState` | **Signals** (`signal()`) in modern Angular; older code uses plain class fields + change detection |
| Reacting to change | Re-run the function; diff the virtual DOM | Fine-grained updates driven by signals / change detection |
| Data fetching | Your choice: `fetch`, TanStack Query, SWR, etc. | Built-in `HttpClient` (RxJS Observables) |
| Routing | Separate library (React Router, TanStack Router) | Built-in `@angular/router` |
| Forms | Controlled inputs, or libraries (React Hook Form) | Built-in template-driven and reactive forms |
| Dependency injection | Not built in (props, context, hooks) | Core feature: services injected via `inject()` / constructors |
| Tooling | Vite (or a meta-framework such as Next.js) | Angular CLI (`ng new`, `ng generate`, `ng build`) |
| Learning curve | Small core, then many choices | Larger up front (TS, DI, decorators, RxJS), then very uniform |
| Project structure | Convention is up to you / your team | Strongly prescribed |

## 3. Same feature, both ways

### React (what the homelab does, trimmed)

```jsx
export default function Customers() {
  const [customers, setCustomers] = useState([])
  const [name, setName] = useState('')

  useEffect(() => {
    fetch('/api/customers').then(r => r.json()).then(setCustomers)
  }, [])

  return (
    <ul>
      {customers.map(c => <li key={c.id}>{c.name}</li>)}
    </ul>
  )
}
```

### Angular (modern, standalone component with signals)

```ts
import { Component, inject, signal } from '@angular/core'
import { HttpClient } from '@angular/common/http'

interface Customer { id: number; name: string; email: string }

@Component({
  selector: 'app-customers',
  template: `
    <ul>
      @for (c of customers(); track c.id) {
        <li>{{ c.name }}</li>
      }
    </ul>
  `,
})
export class CustomersComponent {
  private http = inject(HttpClient)
  customers = signal<Customer[]>([])

  constructor() {
    this.http.get<Customer[]>('/api/customers')
      .subscribe(data => this.customers.set(data))
  }
}
```

Mapping the concepts:

| React | Angular |
|---|---|
| `useState` | `signal()` |
| `useEffect(() => {...}, [])` | constructor / `ngOnInit` |
| `{items.map(...)}` with `key` | `@for (...; track ...)` |
| `{cond && <X />}` | `@if (cond) { <X /> }` |
| `value` + `onChange` | `[(ngModel)]` (two-way binding) |
| props | `input()` signal inputs |
| callback props | `output()` events |
| context / custom hook | injectable **service** |

## 4. Things Angular gives you that React makes you choose

- **HTTP:** `HttpClient` with interceptors (add an auth header to every request in one place).
- **Routing:** guards, lazy loading, route resolvers.
- **Forms:** typed reactive forms with validators.
- **Services/DI:** one place for shared logic and state, easily swapped in tests.

## 5. Things React gives you that Angular makes you accept

- **Smaller core surface.** You can learn the model in an afternoon.
- **Ecosystem breadth.** Many component libraries, state managers, meta-frameworks (Next.js, Remix) and React Native for mobile.
- **Gradual adoption.** You can drop React into one page of an existing site.

## 6. How to choose (rules of thumb)

- Large team, long-lived enterprise app, want one way to do things → **Angular** is a strong fit.
- Small team or product that needs flexibility, a large hiring pool, or mobile reuse → **React** is a strong fit.
- Existing codebase and team skills usually outweigh both of the above.

## 7. Using this for the Training project

The homelab backend (Spring Boot REST API + Postgres + Kafka) is **frontend-agnostic**: it exposes `/api/customers` and doesn't care what calls it. That makes it a good base for a **second frontend**: after finishing the React exercises, you can build the same customer screen in Angular against the same API and compare effort directly. See exercise 8 in [05 · Exercises](05-exercises.md).

## Check yourself

1. Which one needs you to choose a router, and which ships with one?
2. What is the Angular equivalent of passing a callback prop to a child component?
3. Why does Angular's "services + DI" often replace what React does with custom hooks and context?

<details><summary>Answers</summary>

1. React needs you to choose one (React Router etc.); Angular ships `@angular/router`.
2. An `output()` (event emitter) that the parent listens to with `(eventName)="handler($event)"`.
3. Both solve "share logic/state across components." Angular does it with injectable classes provided at app, route, or component level; React does it with functions (hooks) and the context API.
</details>

Next: [03 · Building a React App](03-building-a-react-app.md)
