# 06 · MCP and Agent Harnesses

Goal: understand what an agent harness is, what MCP adds to it, when to use each, and how to design both well. You'll finish by wrapping the homelab API as an MCP server.

## 1. Why this lesson exists

An LLM on its own only produces text. To *do* things (query a database, call your API, edit files) it needs two things around it:

1. A **harness** that runs the loop and enforces the rules.
2. **Tools** the model can call, which can be wired in directly or through **MCP**.

The homelab is a good practice target: it already has a clean API contract (lesson 04 §2) that an agent could use.

## 2. What is a harness?

The model is the engine. The **harness** is the rest of the car: everything that turns a text predictor into an agent.

```
        ┌─────────────────────── harness ───────────────────────┐
 user ─►│ build prompt + tool list ─► MODEL ─► tool call?        │
        │        ▲                               │ yes           │
        │        │                               ▼               │
        │   append result  ◄── run tool (permissions, limits)    │
        │        │                               │ no            │
        │        └───────────────────────────────┴─► final answer│
        └────────────────────────────────────────────────────────┘
```

A harness is responsible for:

| Job | What it means |
|---|---|
| The loop | Call the model, execute requested tools, feed results back, repeat until done |
| Tool definitions | Names, descriptions and JSON schemas the model sees |
| Context management | What goes in the window; trimming, summarizing, memory |
| Permissions | Which tools may run, what needs human approval, sandboxing |
| Limits | Max steps, timeouts, token and cost budgets |
| Observability | Logging every prompt, tool call and result |
| Verification | Checks that the work is right (tests, re-reads, validators) |

Claude Code and the Claude chat app are harnesses. So is a 50-line script you write yourself that loops over an API.

The core loop in pseudocode:

```python
messages = [user_request]
for step in range(MAX_STEPS):
    reply = model(messages, tools=TOOLS)
    messages.append(reply)
    if not reply.tool_calls:
        return reply.text                 # done
    for call in reply.tool_calls:
        result = run_tool(call)           # permission checks happen here
        messages.append(tool_result(call, result))
raise Exception("step limit reached")
```

Everything else in harness design is refinement of this loop.

## 3. What is MCP?

**Model Context Protocol** is an open standard for connecting AI apps to tools and data. The analogy to lesson 04: just as the React app only needs the **API contract** and doesn't care what language the backend is written in, a harness only needs the **MCP contract** and doesn't care how a tool is implemented.

Three roles:

| Role | Example |
|---|---|
| **Host** | The AI app the user sees (Claude desktop, an IDE) |
| **Client** | A connector inside the host that speaks MCP to one server |
| **Server** | A program that exposes capabilities |

A server can expose three kinds of things:

| Primitive | Controlled by | Example |
|---|---|---|
| **Tools** | The model decides to call them | `add_customer(name, email)` |
| **Resources** | The app/user attaches them as context | A schema file, a log, a document |
| **Prompts** | The user picks them | A reusable "summarize these customers" template |

Tools are by far the most used.

Two common transports:

- **stdio**: the host launches the server as a local subprocess and talks over stdin/stdout. Simple; good for local tools.
- **Streamable HTTP**: the server is a web service. Good for shared or remote servers, usually with OAuth.

What a tool looks like on the wire (the model sees roughly this):

```json
{
  "name": "find_customer_by_email",
  "description": "Look up one customer by exact email address. Returns the customer or a 'not found' message.",
  "inputSchema": {
    "type": "object",
    "properties": { "email": { "type": "string" } },
    "required": ["email"]
  }
}
```

The model reads the **name and description** to decide when to call it. They are documentation written for a model.

## 4. MCP vs plain function tools: when to use which

| Situation | Choose |
|---|---|
| One app, one team, tool only used there | Plain function tool in your own code |
| Same tool wanted in several apps or agents | **MCP server** |
| Wrapping a system with login (Gmail, Drive, a database) | **MCP server** (auth lives in one place) |
| Someone already published a server for it | Use theirs; don't rebuild |
| Quick one-off script | Plain code, no protocol |
| Tool must run on the user's machine | Local (stdio) MCP server |

MCP costs you a process, a protocol and some setup. It pays off through **reuse** and **separation**: the tool author and the agent author can be different people.

A third option worth knowing is **just give the agent a shell and a CLI** (as Claude Code does with `git`, `curl`, `docker`). For tools that already have good command-line interfaces, this can be simpler than building a server.

## 5. Designing a good MCP server

### Fewer, outcome-shaped tools

Don't mirror your REST API one-to-one. Model the *tasks* an agent wants to accomplish.

| Weak | Better |
|---|---|
| `get_customers`, `get_customer_by_id`, `get_audit_rows`, `join_them_yourself` | `find_customer(query)`, `customer_history(id)` |
| `post_customers_raw(json)` | `add_customer(name, email)` |

Every extra tool is more text in the context window and one more choice the model can get wrong.

### Descriptions and errors are the interface

- Say **what it does, when to use it, and what it returns.**
- Name parameters plainly and give examples in descriptions.
- Return errors the model can act on: `"Email already registered (customer id 7). Use get_customer(7) to view it."` beats `"500"`.

### Control output size

Everything a tool returns is pasted into the model's context. Return the 10 most relevant rows, not 10,000. Offer `limit` and `offset`, or summarize. Return only fields the model needs.

### Least privilege and safe writes

- Start **read-only**. Add write tools deliberately.
- Separate dangerous actions (`delete_customer`) into their own tool so the host can require approval for just that one.
- Make writes **idempotent** where you can (a retry shouldn't create a duplicate).
- Never let tool arguments become raw SQL or shell strings.
- Treat everything the server returns as **untrusted input** on the harness side (see §7).

### Keep secrets out of the model's view

Credentials belong in the server's environment, not in tool arguments or descriptions. The model never needs to see an API key.

## 6. Designing a harness

Start simple, then add structure only where the simple version fails.

1. **Single loop, few tools.** Get this working first.
2. **Hard limits.** Max steps, per-tool timeouts, a budget. An unbounded loop is a bug, as with `useEffect` without `[]` (lesson 01).
3. **Verification.** Give the agent a way to check itself: run the tests, re-read the file, call the API again. Agents that can verify are far more reliable than agents that only act.
4. **Permissions.** Decide per tool: auto-allow, ask first, or deny. Irreversible or external actions (delete, send, pay) should ask.
5. **Context hygiene.** Summarize old turns, truncate huge tool outputs, keep durable notes in files or memory instead of the window.
6. **Subagents for noisy work.** A subagent can search 50 files and return a 5-line summary, keeping the main context clean. Don't add them until context pressure is real; they cost extra and lose detail.
7. **Logging.** Record every model call and tool call. You will need it to debug.

A useful rule: **put rules in the harness, not just the prompt.** "Never delete without asking" in a prompt is a request; a permission check in code is a guarantee.

## 7. Security: the part people skip

- **Prompt injection:** text returned by a tool (a web page, an email, a customer name) can contain instructions. The model may follow them. Mitigate by limiting what tools can do, requiring approval for risky actions, and never treating tool output as commands.
- **The "lethal trifecta":** an agent that can read private data, ingest untrusted content, *and* send data out can be tricked into leaking. Remove at least one of the three.
- **Trust servers like you trust software:** an MCP server runs code and its descriptions steer the model. Install only servers you trust, prefer read-only scopes, and review what each tool can do.

## 8. How this maps to the homelab

| Homelab piece | Agent concept |
|---|---|
| `CustomerController` REST API | The system a server would wrap |
| `/api/customers` contract (lesson 04 §2) | The raw material for tool design |
| Kafka `customer-events` + `customer_audit` | A natural audit log for agent writes |
| `reports` FastAPI service | A read-only tool source (`summary`, `audit`) |
| nginx proxy | A "single front door" idea: an MCP server is the front door for agents |

Sketch of the server you'll build in exercise 11:

| Tool | Maps to | Kind |
|---|---|---|
| `list_customers(limit)` | `GET /api/customers` | read |
| `get_customer(id)` | `GET /api/customers/{id}` | read |
| `add_customer(name, email)` | `POST /api/customers` | write |
| `report_summary()` | `GET /reports/summary` | read (optional) |

Notice there is no delete tool: the API doesn't have one yet (lesson 04 §2), and least privilege says don't add power you don't need.

## 9. A minimal server, to read before you build

This uses the official Python SDK (`pip install "mcp[cli]"`). Check the SDK's current docs for exact APIs, since they evolve.

```python
# customers_mcp.py
import httpx
from mcp.server.fastmcp import FastMCP

API = "http://localhost:8080/api"
mcp = FastMCP("homelab-customers")

@mcp.tool()
def list_customers(limit: int = 20) -> list[dict]:
    """List customers (newest data from the API). Returns at most `limit` rows."""
    r = httpx.get(f"{API}/customers", timeout=10)
    r.raise_for_status()
    return r.json()[:limit]

@mcp.tool()
def get_customer(id: int) -> dict | str:
    """Get one customer by numeric id. Returns 'not found' text if missing."""
    r = httpx.get(f"{API}/customers/{id}", timeout=10)
    if r.status_code == 404:
        return f"No customer with id {id}."
    r.raise_for_status()
    return r.json()

@mcp.tool()
def add_customer(name: str, email: str) -> dict | str:
    """Create a customer. Both fields required. Returns the saved customer."""
    r = httpx.post(f"{API}/customers", json={"name": name, "email": email}, timeout=10)
    if not r.is_success:
        return f"Save failed with HTTP {r.status_code}. The email may already exist."
    return r.json()

if __name__ == "__main__":
    mcp.run()   # stdio transport by default
```

What to notice: the docstrings are the tool descriptions, errors come back as readable text, output is capped, and there's no secret or raw SQL anywhere.

To test before connecting any app, use the MCP Inspector (`npx @modelcontextprotocol/inspector`), which lets you call tools by hand.

## 10. Common pitfalls

| Symptom | Likely cause |
|---|---|
| Model never calls your tool | Vague name/description; the model can't tell when it applies |
| Model picks the wrong tool | Overlapping tools; merge them or sharpen descriptions |
| Answers get worse as sessions grow | Tool outputs flooding context; cap and summarize |
| Agent loops forever | No step limit, or no clear "done" condition |
| Duplicate records after retries | Non-idempotent write tools |
| Server works in the inspector, not in the app | Wrong command/path in the host config, or the server wrote logs to stdout (breaks stdio; log to stderr) |

## Check yourself

1. What does the harness do that the model cannot do for itself?
2. Why is "a tool per REST endpoint" usually a poor MCP design?
3. Give two situations where a plain function tool is better than an MCP server.
4. Why is a permission check in code stronger than an instruction in the prompt?
5. What is prompt injection, and which tool-design choices reduce its impact?

<details><summary>Answers</summary>

1. Runs the loop, executes tools, enforces permissions and limits, manages context, logs, and verifies. The model only produces the next message or tool request.
2. It floods the context with many low-level tools, pushes orchestration onto the model, and returns raw payloads. Outcome-shaped tools are fewer, clearer and cheaper.
3. A tool used by only one app you control; a quick one-off script; or a tool that is trivial with an existing CLI.
4. The model may ignore or be talked out of a prompt instruction. Code runs regardless of what the model "decides".
5. Instructions hidden in content a tool returns that the model then follows. Reduce impact with read-only defaults, approval on writes/deletes, small outputs, and not combining private-data access, untrusted input and outbound channels in one agent.
</details>

Next: [05 · Exercises](05-exercises.md) (exercise 11)
