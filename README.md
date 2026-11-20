# React & Full-Stack Training

A self-paced course for learning React and understanding the homelab ("computing project") application. It lives **outside** the application repo so you can review it on its own.

- Reference application: https://github.com/wmorgan999/homelab (React front end, Spring Boot API, Kafka consumer, FastAPI reports, Postgres)
- Suggested pace: one lesson per sitting; exercises alongside lessons 03-04.

## Lessons

| # | File | What you'll get |
|---|---|---|
| 01 | [React fundamentals](01-react-fundamentals.md) | Components, JSX, props, state, events, lists, forms, effects, hook rules |
| 02 | [React vs Angular](02-react-vs-angular.md) | Philosophy, side-by-side comparison, same feature in both, how to choose |
| 03 | [Building a React app](03-building-a-react-app.md) | Vite setup, structure, hooks + API layer, proxy, routing, testing, build |
| 04 | [Homelab walkthrough](04-homelab-walkthrough.md) | Architecture, the API contract, `App.jsx` line by line, request flow, gaps to fix |
| 05 | [Exercises](05-exercises.md) | 12 progressive exercises on the real project, plus a self-assessment checklist |
| 06 | [MCP and harnesses](06-mcp-and-harnesses.md) | What a harness is, what MCP adds, when to use each, how to design them, security basics |
| 07 | [React + TypeScript](07-typescript.md) | Java/JPA entity to TypeScript type, converting the app, typing props, hooks and state, common errors |

## How to use it

1. Read 01 and 03 with a scratch Vite project open (`npm create vite@latest`) and type the examples yourself.
2. Read 04 with the homelab repo open beside it. Keep `web/src/App.jsx` in view.
3. Do the exercises on a branch. Use the "Check yourself" questions at the end of each lesson (answers are folded under each one).
4. Read 06 once the homelab runs locally, then do exercise 11 (an MCP server over the customer API).
5. Read 07 once the JavaScript app works (lesson 03 and exercises 1-6), then do exercise 12 (convert it to TypeScript). The modernization stack is React + TypeScript with Spring Boot and JPA, so this closes the loop.

## Prerequisites

- Basic JavaScript (functions, arrow functions, array `map`/`filter`, `async/await`).
- Node.js LTS, Docker with Compose, and Git.
- Java/Python knowledge is **not** required for the React work; you only need to read the backend code. Exercise 11 uses a little Python (or TypeScript) to write the MCP server. Lessons 01-06 use plain JavaScript on purpose; lesson 07 adds TypeScript.

## Not covered (yet)

Global state libraries, server-side rendering (Next.js), authentication, and CI/CD. Good follow-on topics once the exercises are done.
