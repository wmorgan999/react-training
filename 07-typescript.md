# 07 · React + TypeScript (with Spring Boot and JPA)

Goal: understand what TypeScript adds to a React app, how a Java/JPA entity becomes a TypeScript type, and how the course app (`training-react-myapp`) was converted. The modernization stack is React + TypeScript on the front end and Spring Boot + JPA on the back end, so this lesson ties those together.

The converted app is on the branch `claude/typescript` of `wmorgan999/training-react-myapp`. The plain JavaScript version stays on `main`, so you can compare them file by file.

## 1. What TypeScript is (and isn't)

TypeScript is JavaScript plus **type annotations**. A checker (`tsc`) reads them and reports mistakes **before you run the code**. Then the types are thrown away: the browser only ever sees JavaScript.

```ts
const name: string = 'Ada'
name.toFixed(2)          // error: Property 'toFixed' does not exist on type 'string'
```

What it gives you:
- **Typos caught early.** `customer.nmae` is an error in the editor, not an empty cell in the browser.
- **Contracts between components.** A component's props are written down, so passing the wrong thing, or forgetting something, is an error.
- **Safe refactoring.** Rename a field and the checker lists everything that breaks.
- **Autocomplete** that knows your data.

What it does **not** do:
- It does not check data at **runtime**. `response.json()` is typed `any`, so if the server sends something different from your type, TypeScript can't know.
- It does not make slow code fast, or wrong logic right.

If you know Java, the idea is the same as compile-time checking in Java. The difference is that the types are erased afterwards, so there is no runtime type check at all.

## 2. Java entity to TypeScript type

This is the most useful connection for your stack. The JSON the API sends comes from the Java class, so the TypeScript type should mirror it.

```java
// api/.../Customer.java (JPA entity)
@Entity
public class Customer {
    @Id @GeneratedValue private Long id;
    private String name;
    private String email;
    @Column(name = "created_at", insertable = false, updatable = false)
    private OffsetDateTime createdAt;
}
```

```ts
// src/types.ts
export interface Customer {
  id: number
  name: string
  email: string
  createdAt: string | null
}
```

Why `createdAt` is `string | null`: JSON has no date type, so Spring sends `OffsetDateTime` as ISO text (`"2026-11-19T18:00:00Z"`). And the POST response has `createdAt` empty because the database fills it in after the insert (lesson 04 §4). The type has to describe what the API really sends.

| Java / JPA | JSON on the wire | TypeScript | Notes |
|---|---|---|---|
| `Long`, `Integer` | number | `number` | `Long` above 2^53 loses precision in JavaScript; send as a string if ids can be that large |
| `String` | string | `string` | |
| `boolean` | true/false | `boolean` | |
| `BigDecimal` (money) | number | `string` or a decimal library | JavaScript numbers are floating point; don't do money math with them |
| `OffsetDateTime`, `Instant`, `LocalDate` | ISO string | `string` | Convert with `new Date(s)` only when you display it |
| `enum Status { OPEN, CLOSED }` | `"OPEN"` | `'OPEN' \| 'CLOSED'` | a string literal union |
| `List<Customer>` | array | `Customer[]` | |
| field that can be null / `Optional` | `null` or missing | `string \| null` or `name?: string` | match what Jackson actually sends |
| request body without an id | object | `Pick<Customer, 'name' \| 'email'>` | what the form sends is not a full `Customer` |

Two habits that keep the two sides honest:

1. **Return DTOs, not entities**, from the API when the screen needs a different shape. Then the TypeScript type mirrors the DTO, and a change to the database table doesn't silently change your front end.
2. **Generate the types instead of typing them by hand.** Spring can publish an OpenAPI description of the API (the `springdoc-openapi` library), and a tool such as `openapi-typescript` turns it into TypeScript types. Then the Java controller is the single source of truth, and a changed field breaks the front-end build instead of breaking production. The homelab does not have this set up; it is the natural next step for a modernization project, and it is a good stretch exercise (exercise 12).

## 3. Setting up a Vite project for TypeScript

New project: `npm create vite@latest my-app -- --template react-ts`. Converting an existing one (what we did):

```bash
npm install -D typescript          # @types/react and @types/react-dom were already there
```

Add `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noEmit": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  },
  "include": ["src"]
}
```

| Option | What it does |
|---|---|
| `strict` | Turns on the full set of checks. **Always on.** Without it, TypeScript lets many mistakes through |
| `jsx: "react-jsx"` | Understands JSX, with no `import React` needed |
| `moduleResolution: "bundler"` | Resolves imports the way Vite does |
| `noEmit` | Only check; Vite does the building |
| `isolatedModules` | Each file must be understandable on its own, which Vite needs |
| `noUnusedLocals/Parameters` | Unused variables are errors |
| `include: ["src"]` | Which folders to check |

Then:
1. **Rename the files**: `.jsx` becomes `.tsx` (files with JSX), `.js` becomes `.ts` (no JSX). Use `git mv` so history follows.
2. In `index.html`, change `/src/main.jsx` to `/src/main.tsx`.
3. In `package.json`, add the check to the scripts:
   ```json
   "typecheck": "tsc --noEmit",
   "build": "tsc --noEmit && vite build"
   ```

**Why `build` runs `tsc` first:** Vite and Vitest strip the types and run the code **without checking them**. So `npm run dev` and `npm test` can pass with type errors in the code. Only `tsc` (via `npm run typecheck` or `npm run build`) and your editor report them. Run `typecheck` before you commit.

## 4. The conversion, file by file

### `src/types.ts` (new)

The `Customer` type from §2, plus the type for what the form sends:

```ts
export type CustomerInput = Pick<Customer, 'name' | 'email'>
```

`Pick` builds a type from some of another type's fields, so the two can't drift apart.

### `src/api/customers.ts`

```ts
export async function listCustomers(): Promise<Customer[]> {
  const r = await fetch('/api/customers')
  if (!r.ok) throw new Error(`Load failed: ${r.status}`)
  return r.json()          // typed `any`: TypeScript trusts you here
}

export async function getCustomer(id: string | number): Promise<Customer | null> { ... }
export async function createCustomer(data: CustomerInput): Promise<Customer> { ... }

function fail(r: Response, what: string): never {   // `never`: always throws, never returns
  ...
}
```

What's new: parameter types, the return type (`Promise<Customer[]>` means "an async function that gives back an array of customers"), `Customer | null` for "found, or not found" (callers are forced to handle the `null`), and `never` for a helper that always throws.

### `src/hooks/useCustomers.ts`

```ts
const [customers, setCustomers] = useState<Customer[]>([])   // <...> says what the state holds
const [loading, setLoading] = useState(true)                 // inferred: boolean, no annotation needed
```

Without `<Customer[]>`, `useState([])` would be typed as "an array of nothing", and `setCustomers(list)` would be an error. Plain values like `true` and `''` are inferred, so you don't annotate them.

The `catch` block changes too:

```ts
} catch (e) {                       // `e` is `unknown`: anything can be thrown in JavaScript
  setError(errorMessage(e))
}
```
```ts
// src/errors.ts
export function errorMessage(e: unknown): string {
  return e instanceof Error ? e.message : String(e)
}
```
`e.message` directly would be an error, because `e` might not be an `Error`. Checking with `instanceof` is the safe way to find out.

### `src/hooks/useCustomer.ts`: a discriminated union

The JavaScript version kept `{ status, customer, error }` together, where `customer` was `null` unless `status === 'ready'`. TypeScript can say that precisely:

```ts
export type CustomerState =
  | { status: 'loading' }
  | { status: 'ready'; customer: Customer }
  | { status: 'notfound' }
  | { status: 'error'; error: string }
```

Each possibility lists the fields it really has. The `status` field is the "tag". Once you check it, TypeScript **narrows** the type:

```tsx
{state.status === 'error' && <p role="alert">{state.error}</p>}     // fine: error exists here
{state.status === 'ready' && <dd>{state.customer.name}</dd>}        // fine: customer exists here
{state.status === 'loading' && <p>{state.customer.name}</p>}        // ERROR: no `customer` while loading
```

This is the single most useful TypeScript pattern for React screens. It makes "impossible states" (loading *and* has data) impossible to write. Note the page now uses `state.status`, not destructured variables: narrowing works on the object you checked.

### `src/components/CustomerForm.tsx`: typing props

```tsx
interface Props {
  onSubmit: (data: CustomerInput) => Promise<unknown> | void
  initial?: CustomerInput
  submitLabel?: string
  clearOnSubmit?: boolean
  onCancel?: () => void
}

export default function CustomerForm({ onSubmit, initial = { name: '', email: '' }, ... }: Props) {
```

- The `interface Props` is the component's contract. Forget `onSubmit` and the parent file turns red.
- A `?` marks an optional prop, whose type is then `T | undefined`.
- `onSubmit` is a **function type**: `(data) => result`. It is typed as returning a promise or nothing, so both `async` handlers and plain ones fit.
- Events are typed too: `(e: FormEvent<HTMLFormElement>) => ...`. Inline handlers like `onChange={e => setName(e.target.value)}` need no annotation, because React already knows what `e` is there.

### `src/components/CustomerTable.tsx`: `keyof`

```tsx
const COLUMNS: { key: keyof Customer; label: string }[] = [ ... ]
const [sortKey, setSortKey] = useState<keyof Customer>('id')
const [sortDir, setSortDir] = useState<'asc' | 'desc'>('asc')
const [confirmId, setConfirmId] = useState<number | null>(null)
```

- `keyof Customer` means "one of `'id' | 'name' | 'email' | 'createdAt'`". A typo like `'nmae'` is an error, and renaming a field in `Customer` flags every column that uses it.
- `'asc' | 'desc'` is a **literal union**: only those two strings are allowed. It replaces a boolean or a bare string, and does the job of a Java `enum`.
- `useState<number | null>(null)` is needed because plain `useState(null)` would be typed as "always null".

### `src/pages/CustomerDetailPage.tsx`: values that might be missing

```tsx
const { id } = useParams()          // type: string | undefined
if (!id) return <p>Customer not found.</p>
return <CustomerDetail id={id} />   // from here TypeScript knows id is a string
```

React Router can't promise the URL has an `id`, so the type includes `undefined`. We check once and then pass a definite string to a child component. (A hook can't sit after an early `return`, which is why the page is split in two.)

### `src/main.tsx`

```tsx
const root = document.getElementById('root')
if (!root) throw new Error('index.html has no #root element')
createRoot(root).render(...)
```

`getElementById` returns `HTMLElement | null`. The `if` narrows it. You'll see people write `getElementById('root')!` (the `!` means "trust me, it's not null"); avoid `!` where a real check is cheap, because a wrong `!` crashes at runtime.

### The tests

Tests are TypeScript too, so they need types, but only a few spots:

- Helper function parameters: `renderTable = (rows: Customer[], onDelete?: ...) => ...`
- `screen.getByPlaceholderText('Name')` returns a plain `HTMLElement`, which has no `.value`. Tell TypeScript what it really is: `(el as HTMLInputElement).value`.
- Fake `fetch` responses and test data are typed with the real `Customer` type, so a test can't drift from the API shape.

## 5. Errors you will see (real ones from this app)

| Message | What it means |
|---|---|
| `Property 'nmae' does not exist on type 'Customer'` | A typo, or a field that isn't in the type |
| `Property 'customers' is missing in type '{ onDelete: ... }' but required in type 'Props'` | You forgot a required prop |
| `Argument of type 'number' is not assignable to parameter of type 'boolean'` | Wrong type passed to a function |
| `'customers' is declared but its value is never read` | Unused variable (from `noUnusedLocals`) |
| `Type 'string' is not assignable to type 'number'` on `border="1"` | The React type wants a number: `border={1}` |
| `Object is possibly 'undefined'` / `'null'` | Check for it first, or narrow with `if` |
| `Parameter 'x' implicitly has an 'any' type` | Add a type to that parameter |
| `Property 'value' does not exist on type 'HTMLElement'` | You need a more specific element type (an `as` cast in tests) |

Read them bottom-up if the message is long: the last line usually says the actual mismatch.

## 6. Pitfalls

| Pitfall | What to do |
|---|---|
| Typing things as `any` to silence errors | It switches the checker off for that value. Use `unknown` and narrow it, or write the real type |
| `value as Customer` casts | A cast is a promise, not a check. If it's wrong, you get a runtime failure. Prefer narrowing |
| `!` on something that may be null | Same: a promise. Use a real check |
| Trusting `response.json()` | It returns `any`. If the server's shape can change or can't be trusted, validate at runtime (the library **zod** does this and gives you the type from the same definition) |
| Tests pass but `typecheck` fails | Vitest doesn't check types; run `npm run typecheck` |
| Types differ from what the API actually sends | Check the real JSON (browser Network tab, `Invoke-RestMethod`). Better: generate the types from OpenAPI (§2) |
| `enum` | Prefer a string literal union (`'asc' \| 'desc'`): simpler, and it needs no extra runtime code |

## 7. Run it

On your machine, in the my-app folder (commit or stash your current work first):

```powershell
git fetch
git checkout -b typescript origin/claude/typescript
npm install
npm run typecheck
npm test
npm run dev -- --port 5173 --strictPort
```

Notes:
- The branch includes exercises 5 and 6 (detail page, delete and edit), because the TypeScript version was converted from the finished app.
- `npm install` is needed because `typescript` is a new dependency.
- `npm run typecheck` should print nothing and finish quietly. Silence means success.

**Try breaking it on purpose** (then undo with `git checkout -- src`):
1. In `CustomerDetailPage.tsx`, change `state.customer.name` to `state.customer.nmae`. Run `npm run typecheck`.
2. In `CustomersPage.tsx`, delete `customers={customers}` from `<CustomerTable ... />`.
3. In `useCustomer.ts`, add `customer` to the `loading` case of the union and see what changes.
4. Run `npm test` after one of these. It still passes. That's the point from §3.

## Check yourself

1. TypeScript's types are erased before the code runs. So what does `r.json()` returning `any` mean for the safety of your app?
2. How does `Long`, `OffsetDateTime` and a nullable column in a JPA entity map to TypeScript?
3. Why is `{ status: 'ready'; customer: Customer } | { status: 'error'; error: string } | ...` better than `{ status: string; customer: Customer | null; error: string }`?
4. What does the `?` mean in `initial?: CustomerInput`, and what is the type of `initial` inside the component without a default value?
5. `npm test` passes but `npm run build` fails. Why?
6. Name two ways to stop the Java entity and the TypeScript type from drifting apart.

<details><summary>Answers</summary>

1. TypeScript trusts you: it assumes the server sent a `Customer[]`. If the server sends something else, nothing complains until the code fails at runtime. Types check your code against your own declarations, not against the network.
2. `Long` is `number` (with a precision caveat above 2^53), `OffsetDateTime` is `string` (ISO text in JSON), and a nullable column is `T | null`.
3. The union lists exactly which fields exist in each state, so TypeScript rejects reading `customer` while loading and reading `error` when ready. The loose version allows impossible combinations and needs `null` checks everywhere.
4. The prop is optional. Its type is `CustomerInput | undefined`, so you must handle `undefined` (a default value does that).
5. Vitest strips types and doesn't check them. `build` runs `tsc --noEmit` first, which does.
6. Return DTOs and mirror them; or generate the TypeScript types from the API's OpenAPI description (springdoc + openapi-typescript), so the Java code is the single source of truth.
</details>

Next: [05 · Exercises](05-exercises.md) (exercise 12)
