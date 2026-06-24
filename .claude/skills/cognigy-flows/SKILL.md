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

## API response format (HAL)

The API returns HAL-format responses. Data is **never** under `items` — always under `_embedded`:

```python
# Projects
projects = api("GET", "/v2.0/projects?limit=100")
all_projects = projects.get("_embedded", {}).get("projects", [])

# Flows
flows = api("GET", f"/v2.0/flows?projectId={project_id}&limit=100")
all_flows = flows.get("_embedded", {}).get("flows", [])
```

Object IDs are extracted from `_links.self.href` (not a top-level `_id`):
```python
href = obj.get("_links", {}).get("self", {}).get("href", "")
obj_id = href.rstrip("/").split("/")[-1]
```

Flow/project names are nested under `properties`:
```python
name = obj.get("properties", obj).get("name", "")
```

For single-object responses (GET by ID, chart, node), fields are at the top level with a real `_id`.

---

## Step 1 — Select a project

```python
projects = api("GET", "/v2.0/projects?limit=100")
all_projects = projects.get("_embedded", {}).get("projects", [])
for p in all_projects:
    pid = p.get("_links",{}).get("self",{}).get("href","").split("/")[-1]
    print(pid, p["name"])
```

---

## Step 2 — List flows in a project

```python
flows = api("GET", f"/v2.0/flows?projectId={project_id}&limit=100")
all_flows = flows.get("_embedded", {}).get("flows", [])
for f in all_flows:
    fid = f.get("_links",{}).get("self",{}).get("href","").split("/")[-1]
    name = f.get("properties", f).get("name", fid)
    print(fid, name)
```

---

## Step 3 — Read flow structure

**Use the chart endpoint** — the paginated nodes endpoint returns `total` but empty items in practice. The chart returns everything at once including the relation graph:

```python
chart = api("GET", f"/v2.0/flows/{flow_id}/chart")
# chart["nodes"]     → list of node objects (each has _id, type, label, referenceId, config, ...)
# chart["relations"] → list of {node, next, children} — the connection graph
```

Build and display as a tree:
```python
nodes_by_id = {n["_id"]: n for n in chart["nodes"]}
rels_by_node = {r["node"]: r for r in chart["relations"]}

# Find roots: nodes nobody points to as "next"
all_nexts = {r["next"] for r in chart["relations"] if r["next"]}
roots = set(nodes_by_id) - all_nexts

def print_node(nid, indent=0):
    n = nodes_by_id.get(nid)
    if not n: return
    preview = n.get("preview", "")
    if isinstance(preview, dict):
        ptext = preview.get("text") or preview.get("condition", "")
        if isinstance(ptext, list): ptext = ptext[0] if ptext else ""
    else:
        ptext = str(preview)
    entry = " 🔵" if n.get("isEntryPoint") else ""
    print("  " * indent + f"[{n['type']}] {n.get('label','')}{entry}  →  {str(ptext)[:80]}")
    rel = rels_by_node.get(nid, {})
    for child_id in rel.get("children", []):
        print_node(child_id, indent + 1)
    if rel.get("next"):
        print_node(rel["next"], indent)

for root in roots:
    print_node(root)
```

---

## Step 4 — Fetch node descriptors

**Note: the descriptors endpoint currently returns empty in this environment.** Skip it and instead read an existing node of the same type to learn its config shape:

```python
# Find an existing node of the type you want to create/edit
existing = next(n for n in chart["nodes"] if n["type"] == "question")
full = api("GET", f"/v2.0/flows/{flow_id}/chart/nodes/{existing['_id']}")
print(json.dumps(full.get("config", {}), indent=2))
```

---

## Step 5 — Get a single node

```python
node = api("GET", f"/v2.0/flows/{flow_id}/chart/nodes/{node_id}")
print(json.dumps(node, indent=2))
```

---

## Step 6 — Create a node

**Required fields:** `type`, `extension`, `target` (_id of existing node), `mode`

Valid modes: `append`, `prepend`, `appendChild`, `prependChild`, `insertAfter`, `insertChildAt`
**Do NOT use `insertBefore`** — causes 500 error. Use `append` on the preceding node instead.

```python
def create(label, ntype, target, mode, config=None):
    payload = {"type": ntype, "extension": "@cognigy/basic-nodes",
               "label": label, "target": target, "mode": mode}
    if config: payload["config"] = config
    r = api("POST", f"/v2.0/flows/{flow_id}/chart/nodes", payload)
    return r.get("_id") or r.get("_links",{}).get("self",{}).get("href","").split("/")[-1]
```

### If node behaviour

Creating an `if` node **automatically creates its `then` and `else` children** — do NOT create them manually (400 error). After creating the If, fetch the chart and read `relations[if_id].children` to get their IDs:

```python
if_id = create("My check", "if", target_id, "append", if_config("context.x === 1"))
chart = api("GET", f"/v2.0/flows/{flow_id}/chart")
rels = {r["node"]: r for r in chart["relations"]}
then_id = rels[if_id]["children"][0]
else_id = rels[if_id]["children"][1]
```

### If node config shape

```python
def if_config(condition_str):
    return {"condition": {
        "type": "condition",
        "rule": {"left": "1", "operand": "eq", "right": "1"},
        "condition": condition_str
    }}
```

### Config shapes for key node types (verified against live API)

**Say** — `text` is always an array:
```json
{
  "say": {"type": "text", "text": ["Hello, {{context.name}}!"]},
  "handoverOutput": "userAndAgent",
  "generativeAI_rephraseOutputMode": "none",
  "generativeAI_amountOfLastUserInputs": 5,
  "generativeAI_customInputs": "",
  "generativeAI_temperature": 0.7
}
```

**Question** — `type` is the question type, text is in `say.text` (array):
```json
{
  "type": "text",
  "say": {"type": "text", "text": ["What is your name?"]},
  "storeResultInContext": true,
  "contextKey": "userName",
  "validationMessage": "Not sure I understood that."
}
```
For yes/no questions use `"type": "yesNo"`. Answer is in `input.slots.yesNo[0].keyphrase` (`'yes'` or `'no'`).

**Code** — pure JS, key is `code` (not `script`):
```json
{"code": "context.result = context.value * 2;"}
```

**GoTo** — uses `referenceId` (UUID), NOT `_id`. Get them:
- Flow referenceId: `GET /v2.0/flows/{flowId}` → `.referenceId`
- Node referenceId: from `chart["nodes"][i]["referenceId"]`

```json
{
  "flowNode": {"flow": "<flow-referenceId>", "node": "<node-referenceId>"},
  "absorbContext": false,
  "executionMode": "continue",
  "injectedText": "",
  "injectedData": "{}",
  "parseIntents": false,
  "parseKeyphrases": false
}
```

### Code node sandbox restrictions

**Not available:** `Buffer`, `btoa`, `atob`, `require`, `crypto`, `fetch`, or any Node.js built-ins. Write pure ECMAScript only. If you need base64 or hashing, implement in pure JS.

---

## Step 7 — Update a node

Always read the node first, then merge your changes into the full config to avoid wiping fields:

```python
node = api("GET", f"/v2.0/flows/{flow_id}/chart/nodes/{node_id}")
config = node.get("config", {})
config["say"]["text"] = ["Updated message"]  # mutate only what you need
api("PATCH", f"/v2.0/flows/{flow_id}/chart/nodes/{node_id}", {"config": config})
# Returns 204 on success (no body)
```

---

## Step 8 — Move a node

```python
api("PATCH", f"/v2.0/flows/{flow_id}/chart/nodes/{node_id}/move", {
    "target": "NEW_TARGET_NODE_ID",
    "mode": "append"
})
```

---

## Step 9 — Delete a node

Confirm with the user first. Deleting a parent node may remove its children.

```python
api("DELETE", f"/v2.0/flows/{flow_id}/chart/nodes/{node_id}")
```

---

## Workflow rules

1. **Use the chart endpoint** (`GET /chart`) for reading structure — not the paginated nodes endpoint.
2. **Always read a node before updating** — merge changes, never replace the whole config.
3. **Never use `insertBefore`** — use `append` on the preceding node.
4. **If nodes auto-create Then/Else** — fetch chart after creation to get their IDs.
5. **GoTo needs referenceId (UUID), not _id** — fetch flow object and chart nodes to get them.
6. **Say and Question `text` fields are always arrays**, not strings.
7. **Code node is pure ECMAScript** — no Node.js globals at all.
8. **Confirm destructive operations** — ask before delete or move.
9. **After any mutation**, re-fetch and display the chart so the user sees current state.
