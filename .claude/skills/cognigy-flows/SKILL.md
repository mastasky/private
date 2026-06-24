# cognigy-flows

Manage Cognigy.AI Flows: read structure, understand nodes, create/edit/move/delete nodes.

## Constants

```
BASE_URL = https://api-trial.cognigy.ai
API_KEY  = 4704471a069ce9a4c9a8cea0caf6002fc56715862882c1a788a5da349c94afe3adf032e9ea35a29708c50379c59043d77f81a4de02d02ec686e6e15a1b9efa81
```

All requests use header: `X-API-Key: <API_KEY>`

---

## Python helper

All API calls use this pattern — write to `/tmp/cognigy_op.py` then run it:

```python
import json, urllib.request, urllib.error, sys

BASE = "https://api-trial.cognigy.ai"
KEY  = "4704471a069ce9a4c9a8cea0caf6002fc56715862882c1a788a5da349c94afe3adf032e9ea35a29708c50379c59043d77f81a4de02d02ec686e6e15a1b9efa81"
HEADERS = {"X-API-Key": KEY, "Content-Type": "application/json"}

def api(method, path, body=None):
    url = BASE + path
    data = json.dumps(body).encode() if body else None
    req = urllib.request.Request(url, data=data, headers=HEADERS, method=method)
    try:
        with urllib.request.urlopen(req, timeout=30) as r:
            raw = r.read().decode()
            return json.loads(raw) if raw else {}
    except urllib.error.HTTPError as e:
        print(f"HTTP {e.code}: {e.read().decode()}", file=sys.stderr); sys.exit(1)
```

---

## Step 1 — Select a project (always do this first unless user already specified a projectId)

```python
projects = api("GET", "/v2.0/projects?limit=100")
for p in projects["items"]:
    print(p["_id"], p["name"])
```

Show the list, ask the user which project to work with. Store the chosen `projectId`.

---

## Step 2 — List flows in a project

```python
flows = api("GET", f"/v2.0/flows?projectId={project_id}&limit=100")
for f in flows["items"]:
    print(f["_id"], f["name"])
```

Ask the user which flow to work with. Store `flowId`.

---

## Step 3 — Read and display flow structure

Fetch all nodes with pagination, then display as an indented tree:

```python
nodes, cursor = [], None
while True:
    qs = f"?limit=100" + (f"&next={cursor}" if cursor else "")
    page = api("GET", f"/v2.0/flows/{flow_id}/chart/nodes{qs}")
    nodes += page["items"]
    cursor = page.get("nextCursor")
    if not cursor:
        break

# Build id→node map and print
for n in nodes:
    print(f"[{n['type']}] {n.get('label') or n['type']}  id={n['_id']}"
          + (f"  ⛔ disabled" if n.get("isDisabled") else "")
          + (f"  🔵 entry" if n.get("isEntryPoint") else ""))
```

**Understand the structure:** Cognigy stores ordering implicitly — present nodes grouped by logical sequence. Highlight entry points, disabled nodes, and node types. When the user asks "what does this flow do?", walk through the nodes in order and summarize each node's purpose based on its type and config.

---

## Step 4 — Fetch node descriptors (do this before creating or editing any node)

Descriptors define what `config` fields every node type accepts:

```python
descriptors = api("GET", f"/v2.0/flows/{flow_id}/chart/descriptors")
# descriptors["items"] → list of descriptor objects
# Each has: type, extension, defaultLabel, fields (list of {key, type, defaultValue})
for d in descriptors["items"]:
    print(d["type"], [f["key"] for f in d.get("fields", [])])
```

Use descriptors to:
- Know what `config` keys a node type accepts before creating/editing it
- Show the user what's configurable on a node type they ask about
- Validate that fields you're setting actually exist on the node type

---

## Step 5 — Get a single node

```python
node = api("GET", f"/v2.0/flows/{flow_id}/chart/nodes/{node_id}")
print(json.dumps(node, indent=2))
```

---

## Step 6 — Create a node

**Required fields:**
- `type`: node type string (from descriptors)
- `extension`: extension identifier from descriptor (e.g. `@cognigy/basic-nodes`)
- `target`: `_id` of the node to attach to
- `mode`: one of `append`, `prepend`, `appendChild`, `prependChild`, `insertAfter`, `insertBefore`, `insertChildAt`

**Config:** build `config` from the descriptor's `fields` list. Use `defaultValue` as base, then apply user's intent.

```python
payload = {
    "type": "say",
    "extension": "@cognigy/basic-nodes",
    "label": "Welcome message",
    "target": "TARGET_NODE_ID",
    "mode": "append",
    "config": {
        # populated from descriptor fields + user intent
    }
}
result = api("POST", f"/v2.0/flows/{flow_id}/chart/nodes", payload)
print("Created:", result["_id"])
```

### Code node sandbox restrictions

The Cognigy Code node runs in a heavily restricted JS sandbox. **Not available:** `Buffer`, `btoa`, `atob`, `require`, `crypto`, `fetch`, `XMLHttpRequest`, or any Node.js built-ins. Write pure JS only — no globals beyond standard ECMAScript (Math, String, Array, Object, JSON, etc.).

### Config patterns for the 5 key node types

Fetch the actual shapes from descriptors at runtime — these are starting points:

**Say** — outputs a message to the user
```json
{
  "say": {
    "type": "text",
    "text": "Hello!"
  }
}
```

**HTTP Request** — calls an external URL
```json
{
  "method": "GET",
  "url": "https://example.com/api",
  "headers": [],
  "bodyType": "json",
  "body": {},
  "responseVar": "httpResult"
}
```

**Code** — runs arbitrary JS
```json
{
  "script": "actions.output('Hello from code node', {});"
}
```

**Question** — asks the user something and waits for a reply
```json
{
  "questionType": "text",
  "text": "What is your name?",
  "repromptOptions": { "enabled": false },
  "validation": { "enabled": false }
}
```

**AddToContext** — stores a value in the conversation context
```json
{
  "key": "myKey",
  "value": "{{input.text}}"
}
```

> Always verify the exact field names against the descriptor before creating. Run Step 4 first.

---

## Step 7 — Update a node

PATCH with only the fields you want to change. Always read the node first (Step 5) to avoid overwriting config fields you don't intend to touch.

```python
payload = {
    "label": "New label",
    "config": { ...merged config... }
}
result = api("PATCH", f"/v2.0/flows/{flow_id}/chart/nodes/{node_id}", payload)
# 204 = success, no body
```

---

## GoTo node config

GoTo uses `referenceId` (UUID format), NOT `_id`. Get them via:
- Flow referenceId: `GET /v2.0/flows/{flowId}` → `.referenceId`
- Node referenceId: from the chart nodes array → `.referenceId`

```python
{
    "flowNode": {"flow": "<flow-referenceId>", "node": "<node-referenceId>"},
    "absorbContext": False,
    "executionMode": "continue",   # or "once"
    "injectedText": "",
    "injectedData": "{}",
    "parseIntents": False,
    "parseKeyphrases": False
}
```

## If node behaviour

Creating an `if` node automatically creates its `then` and `else` children — do NOT try to create them manually (400 error). Fetch the chart after creating the If to get the auto-created children IDs from `relations[].children`.

## Step 8 — Move a node

```python
payload = {
    "target": "NEW_PARENT_NODE_ID",
    "mode": "append"   # same mode options as create
}
api("PATCH", f"/v2.0/flows/{flow_id}/chart/nodes/{node_id}/move", payload)
```

---

## Step 9 — Delete a node

Confirm with the user before deleting. Deleting a parent node may remove its children.

```python
api("DELETE", f"/v2.0/flows/{flow_id}/chart/nodes/{node_id}")
```

---

## Workflow rules

1. **Always fetch descriptors before creating or editing** — never guess `config` shapes.
2. **Always read a node before updating** — merge changes into existing config, don't replace wholesale.
3. **Show the flow structure after any mutation** — re-run Step 3 so the user sees the current state.
4. **Paginate node fetches** — flows can have hundreds of nodes; always follow `nextCursor`.
5. **Confirm destructive operations** — ask before delete or move.
6. **When explaining a flow:** walk nodes in order, describe each type in plain language, call out entry points, disabled nodes, and any HTTP/Code nodes (side effects).
