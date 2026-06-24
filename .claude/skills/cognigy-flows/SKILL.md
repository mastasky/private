---
name: cognigy-flows
description: Manage Cognigy.AI Flows via the REST API — read flow structure and create/edit/move/delete nodes (Say, Question, If, Code, GoTo, AI Agent + tools), building both deterministic and AI-agent conversation flows. Use when working with Cognigy.AI, building chatbots or voicebots, or editing Cognigy conversation flows.
---

# cognigy-flows

Manage Cognigy.AI Flows: read structure, understand nodes, create/edit/move/delete nodes.

## Constants

```
BASE_URL = https://api-trial.cognigy.ai/new
```

**Always use the `/new` namespace.** All calls go to `https://api-trial.cognigy.ai/new/v2.0/...`. The `/new` prefix is a strict superset of the plain `/v2.0` API — everything that works without it works with it, and some routes (e.g. creating endpoints) **only** work under `/new`. The helper below bakes `/new` into `BASE`, so every `path` you pass starts at `/v2.0/...` and is automatically served from `/new`.

All requests use header: `X-API-Key: <API_KEY>`

### Getting the API key

The API key is **not** stored in this skill. At the start of a session, obtain it in this order:

1. If `COGNIGY_API_KEY` is set in the environment, use it.
2. Otherwise, **ask the user in chat** for their Cognigy API key before making any API call.

Keep the key in memory for the session only — never write it to `SKILL.md`, commit it, or echo it back in output. When writing the helper script, inject the key from the environment (see below) rather than pasting the literal value.

---

## Python helper

All API calls use this pattern — write to `/tmp/cognigy_op.py` then run it.

Pass the key via the environment, never inline. Run the script as:
`COGNIGY_API_KEY="<key the user gave you>" python3 /tmp/cognigy_op.py`

```python
import json, os, urllib.request, urllib.error, sys

BASE = "https://api-trial.cognigy.ai/new"   # always use the /new namespace
KEY  = os.environ["COGNIGY_API_KEY"]   # ask the user for this; do not hardcode
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

### Creating a flow

`POST /v2.0/flows` with just `projectId` + `name` (201 on success). **Do NOT send `localeId`** — it's rejected ("Field 'localeId' is not allowed"); the flow inherits the project's locales. A fresh flow comes with a `start` and an `end` node already wired — fetch the chart to get the start node's `_id` before adding nodes.

```python
flow = api("POST", "/v2.0/flows", {"projectId": project_id, "name": "My Flow"})
flow_id = flow["_id"]
chart = api("GET", f"/v2.0/flows/{flow_id}/chart")
start_id = next(n["_id"] for n in chart["nodes"] if n["type"] == "start")
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

**CRITICAL — how to attach branch body nodes (verified live).** The `then`/`else` nodes are just connector anchors. The body of each branch chains off the **`next`** of the then/else node, NOT as a child of it. So to put a node in the `then` branch, create it with `mode: "append"` and `target=then_id` (this sets `then.next → yourNode`). Do **NOT** use `appendChild`/`prependChild` on the then/else node — that nests the node *inside* the then/else as a child and the runtime does not execute it as the branch body.

```python
# RIGHT — body hangs off then.next / else.next:
code_id = create("Set flag", "code", then_id, "append", code_cfg)   # then.next → Code
create("Go to agent", "goTo", code_id, "append", goto_cfg)          # Code.next → GoTo
create("Decline",     "say",  else_id, "append", say_cfg)           # else.next → Say

# WRONG — appendChild nests it as a child of then; it won't run as the branch body:
# create("Set flag", "code", then_id, "appendChild", code_cfg)
```

Confirm in the chart: `relations[then_id].next` should point at your first body node, and `relations[then_id].children` should be empty. Chain further nodes with `append` on the previous body node. (An If branch falls through to the If's own `next` after the branch body runs, unless a blocking node — Question, AI Agent — ends the turn inside the branch.)

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

**Say** — `text` is always an array. **Always set `preventTranscript: true`** — every Say node must include this to prevent its output from polluting the transcript. (The field is `preventTranscript`, NOT `excludeFromTranscript` — the latter is rejected with HTTP 400 "Invalid field". Note: `preventTranscript: true` also suppresses the Say's REST output when the Say sits at the **end** of a branch with no blocking node after it — if you need the user to actually see a terminal message like a decline, set `preventTranscript: false` on that Say.)
```json
{
  "say": {"type": "text", "text": ["Hello, {{context.name}}!"]},
  "preventTranscript": true,
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
**Reading a Question's answer in a later node:** the answer is stored in **`input.result`** (the runtime field), regardless of `storeResultInContext`/`contextKey` settings. The type of `input.result` depends on the question type:
- `"type": "text"` → string (the user's text)
- `"type": "yesNo"` → **boolean `true` / `false`** (NOT the strings `"yes"`/`"no"`)

So an If node checking a yesNo answer must use `input.result === true`, not `input.result === "yes"` and not `input.slots.yesNo[...]`. Verified live against the trial endpoint. When in doubt, probe the real value: temporarily echo `{{input.result}}` in a downstream Say node.

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

**AI Agent** (`aiAgentJob`) — an LLM-driven conversational agent. Extension is `@cognigy/basic-nodes` like the others. The persona is defined **inline** via `name` / `description` / `instructions`; two reference fields link shared project resources:
- `llmProviderReferenceId` — a Large Language Model resource. List them: `GET /v2.0/largelanguagemodels?projectId={projectId}` (returns `referenceId` + `name` + model). gpt-4o works well: `86e64e87-9c22-4013-98a9-219d8ec978d0` in this trial.
- `aiAgent` — an AI Agent resource (persona) referenceId. **Prefer creating a dedicated persona per agent** rather than borrowing one — it's cheap and keeps voice/identity/safety config separate.

**Creating a persona (AI Agent resource):** `POST /v2.0/aiagents` needs only `projectId` + `name` (201). The resource holds identity/delivery config — `image`, `speakingStyle`, `voiceConfigs` / `enableVoiceConfigs`, `safetySettings`, `enableAutoLanguageDetection`, `contactProfilesOption` — **not** the prompt: `description`/`instructions` stay inline on the node. Use the returned `referenceId` as the node's `aiAgent`.

**Always set the persona `image`.** Default to `"default-avatar:0"`. Accepted both at create and via PATCH. Captured the node's `_id` and `referenceId` from the create response — you need the **mongo `_id`** for any later GET/PATCH/DELETE.

```python
agent = api("POST", "/v2.0/aiagents",
            {"projectId": project_id, "name": "Mr. Sweet", "image": "default-avatar:0"})
agent_ref, agent_id = agent["referenceId"], agent["_id"]   # ref → config["aiAgent"]; _id → updates
# update later:  api("PATCH", f"/v2.0/aiagents/{agent_id}", {"image": "default-avatar:0"})
# delete:        api("DELETE", f"/v2.0/aiagents/{agent_id}")
```

Caveats — **save the `_id` at creation time**: the `GET /v2.0/aiagents` **list** endpoint 500s in this trial (under `/new` too — it's a trial bug, not a namespace issue), the single-GET/PATCH/DELETE require the mongo `_id`, and there is **no referenceId→`_id` lookup**. If you only have a persona's `referenceId` (e.g. copied from a node), you cannot edit or delete that persona via the API — you'd have to recreate it (names must be unique, so a duplicate name 409s) or fix it in the UI. So: capture `_id` when you create, and reuse one persona across flows rather than orphaning duplicates.

```json
{
  "aiAgent": "<aiAgent-referenceId copied from an existing node>",
  "llmProviderReferenceId": "<llm referenceId>",
  "name": "Candy Salesman",
  "description": "Short summary of who the agent is.",
  "instructions": "- Bullet-point behaviour rules.\n- Stay on topic; decline off-topic asks.",
  "toolChoice": "auto", "memoryType": "inherit",
  "knowledgeSearchBehavior": "never",
  "apiVersion": "1.0", "timeoutInMs": 8000, "maxTokens": 4000, "temperature": 0.8,
  "errorHandling": "continue", "storeLocation": "stream", "streamStoreCopyInInput": true,
  "inputKey": "aiAgentOutput", "contextKey": "aiAgentOutput", "outputImmediately": true,
  "streamStopTokens": [".", "!", "?", "\\n"]
}
```

**Auto-created children (like If→then/else):** creating an `aiAgentJob` auto-spawns an `aiAgentJobDefault` ("Default", the path taken when the agent just replies) and one `aiAgentJobTool` ("Tool") child. Do not create them manually. For a **pure conversational agent with no tools, delete the empty Tool node** (`DELETE .../chart/nodes/{toolId}`) — an unconfigured tool is dead weight. Set `knowledgeSearchBehavior: "always"` to attach knowledge-store search.

`storeLocation: "stream"` + `outputImmediately: true` streams the reply straight to the user (no Say node needed). The full result is also stored at `input.aiAgentOutput` / `context.aiAgentOutput`.

### Giving an AI Agent a tool

A tool lets the agent call into the flow (look something up, hit an API, run logic) and use the result in its reply. The structure (verified live):

```
aiAgentJob ──children──> [ aiAgentJobDefault (Default) , aiAgentJobTool (your tool) ]
aiAgentJobTool ──next──> <your handler nodes> ──next──> aiAgentToolAnswer
```

1. **Create the tool node** as a child of the agent — `type: "aiAgentJobTool"`, extension `@cognigy/basic-nodes`, `target`=the aiAgentJob `_id`, `mode: "appendChild"`. Config:
   ```json
   {
     "toolId": "get_candy_price",
     "description": "Look up the price in USD of a specific candy by its name.",
     "useParameters": true,
     "parameters": "{\"type\":\"object\",\"properties\":{\"candy\":{\"type\":\"string\",\"description\":\"The candy name to price.\"}},\"required\":[\"candy\"],\"additionalProperties\":false}",
     "condition": ""
   }
   ```
   `parameters` is a **JSON-schema string** (stringify it). `toolId` is the function name the model calls. `condition` (optional CognigyScript) gates whether the tool is offered.
2. **Build the handler branch** by appending nodes to the tool node (`target`=tool `_id`, `mode: "append"`): do the work (Code/HTTP/etc.), then end the branch with an **`aiAgentToolAnswer`** node ("Resolve Tool Action").
3. **Read the call arguments at `input.aiAgent.toolArgs.<paramName>`** — e.g. `input.aiAgent.toolArgs.candy`. (Discovered live; not in the docs.)
4. **Return the result** via `aiAgentToolAnswer` config `{"answer": "...", "debugToolAnswer": false}`. The `answer` string supports simple CognigyScript interpolation (`{{context.x}}`) and is fed **back to the LLM**, which then phrases the user-facing reply — so make it factual ("A bag of {{context.candyName}} costs ${{context.candyPrice}}."), not a finished sentence. **Handlebars block helpers (`{{#if}}`, `{{#each}}`) do NOT render here** — a `{{#if context.x.success}}...{{else}}...{{/if}}` answer silently produces nothing, the LLM gets an empty result and reports failure. For any conditional answer, build the whole string in the preceding Code node (`context.toolMessage = success ? "..." : "..."`) and set the answer to a plain `{{context.toolMessage}}`.

**Forcing tool use:** `toolChoice: "auto"` lets the model decide — and it will happily answer from its own knowledge instead of calling the tool. If the tool must run (e.g. real prices, not hallucinated ones), say so explicitly in `instructions`: *"For ANY price question you MUST call get_candy_price; never guess a price."*

**Debugging tool runs:** the REST endpoint response does **not** include `context`, so you can't read intermediate values from it. To verify a tool fired, either set a sentinel `answer` (a fake value the bot will parrot) or temporarily append a Say node *after* the agent in the main flow echoing `{{context.x}}` — run one turn, read it, then delete the Say.

### Voice bots — Set Session Config node

For **voice bots**, place a **Set Session Config** node (`type: "setSessionConfig"`, extension **`@cognigy/voicegateway2`**) **in front of the AI Agent node**. It configures the Voice Gateway (speech-to-text, text-to-speech, barge-in, timeouts, DTMF) for the session before the agent speaks. Create it like any node (`append` on the start node, then append the AI Agent after it). Build voice flows as: `start → setSessionConfig → aiAgentJob`.

Key config options (full set verified live; all optional — defaults shown are sane):
- **TTS (how the bot speaks):** `ttsVendor` (`"none"` = inherit endpoint default, else e.g. `"google"`/`"elevenlabs"`/`"azure"`), `ttsVoice`, `ttsLanguage`, `ttsModel`, `ttsDisableCache`.
- **STT (how it hears):** `sttVendor`, `sttLanguage`, `sttModel`, `sttHints` (array of bias phrases), `sttDisablePunctuation`, `googleModel` (e.g. `"latest_short"`). Azure/Google multi-language recognition via `recognizeLanguagesAzure`/`recognizeLanguagesGoogle` + `stt*Lang1..3`.
- **Barge-in (caller interrupts the bot) — turn this on for voice bots:** `bargeInOnSpeech: true` lets the caller cut in while the bot is talking; `bargeInMinWordCount: 2` requires ≥2 words so coughs / "uh" / background noise don't falsely interrupt; `bargeInOnDtmf: true` also interrupts on a keypad press. Without barge-in a voice bot feels robotic — the caller has to wait out every prompt.
- **Endpointing (when a turn ends):** `deepgramEndpointing` + `deepgramEndpointingValue` (ms, e.g. `250`), `sttVadEnabled`, `sttVadMode`, `sttVadVoiceMs`.
- **No-input handling:** `userNoInputTimeoutEnable`, `userNoInputTimeout` (ms), `userNoInputRetries`, `userNoInputMode` (`"event"`/`"play"`), `userNoInputSpeech`/`userNoInputUrl`; flow-level equivalents `flowNoInput*`.
- **DTMF (keypad):** `dtmfEnable`, `dtmfMaxDigits`/`dtmfMinDigits`, `dtmfInterDigitTimeout`, `dtmfSubmitDigit` (e.g. `"#"`).
- **Ambience/overlay:** `atmosphereUrl`/`atmosphereVolume`/`atmosphereLoop`, `silenceOverlayURL`/`silenceOverlayDelay`.
- **Raw passthrough:** `sessionParams` (JSON string) for vendor params not exposed as fields.

Recommended voice-friendly config (inherit the endpoint's voice, enable barge-in + no-input recovery):
```json
{"ttsVendor": "none", "sttVendor": "none",
 "bargeInOnSpeech": true, "bargeInMinWordCount": 2, "bargeInOnDtmf": true,
 "userNoInputTimeoutEnable": true, "userNoInputTimeout": 10000, "userNoInputRetries": 1,
 "userNoInputMode": "event", "deepgramEndpointing": true, "deepgramEndpointingValue": 250,
 "sessionParams": "{}"}
```
Verify placement after creating it: in the chart `relations`, each node has a `next` pointer — confirm the chain reads `start → setSessionConfig → aiAgentJob`. `append` on the start node inserts it right after start, ahead of whatever already followed.
`"none"` for `sttVendor`/`ttsVendor` means "use whatever the voice endpoint is already configured with" — safe default when you don't need to override the voice.

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

### Disabling / enabling a node

`isDisabled` is a **top-level node field**, not part of `config`. A disabled node stays in the flow (and in the chart) but is skipped at runtime. Toggle it with a PATCH at the node root:

```python
api("PATCH", f"/v2.0/flows/{flow_id}/chart/nodes/{node_id}", {"isDisabled": True})
```

This is the mechanism behind the "disable external calls" scaffolding pattern below. The Step 10 lint flags disabled nodes as INFO so they aren't forgotten.

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

## Step 10 — Lint / health-check a flow

Scan every node for the failure classes that bite in production. Fetch the chart once, then the full config per node, and report findings grouped by severity.

```python
import re
FORBIDDEN = ["Buffer", "btoa", "atob", "require(", "crypto", "fetch(", "XMLHttpRequest"]

def lint(flow_id):
    chart = api("GET", f"/v2.0/flows/{flow_id}/chart")
    ref_ids = {n.get("referenceId") for n in chart["nodes"]}
    issues = []
    for n in chart["nodes"]:
        t, label = n.get("type"), n.get("label") or n.get("type")
        cfg = api("GET", f"/v2.0/flows/{flow_id}/chart/nodes/{n['_id']}").get("config", {})
        # ERROR: node flagged broken by Cognigy itself
        if cfg.get("hasError") is True:
            issues.append(("ERROR", label, "node config has hasError=true"))
        # ERROR: Code node uses a global the sandbox doesn't provide
        if t == "code":
            for bad in FORBIDDEN:
                if bad in cfg.get("code", ""):
                    issues.append(("ERROR", label, f"Code uses forbidden global: {bad.rstrip('(')}"))
        # ERROR: GoTo points at a node that no longer exists in this flow
        if t == "goTo":
            tgt = cfg.get("flowNode", {})
            if tgt.get("flow") == chart.get("referenceId") and tgt.get("node") not in ref_ids:
                issues.append(("ERROR", label, "GoTo target node not found in this flow"))
        # WARN: Say/Question with no text
        if t in ("say", "question"):
            txt = cfg.get("say", {}).get("text", [])
            if not txt or not any((s or "").strip() for s in txt):
                issues.append(("WARN", label, f"{t} has empty text"))
        # WARN: Say node missing preventTranscript
        if t == "say" and "preventTranscript" not in cfg:
            issues.append(("WARN", label, "Say node missing preventTranscript"))
        # ERROR: Question or GoTo in a flow that contains an AI Agent
        if t in ("question", "goTo"):
            issues.append(("ERROR", label, f"{t} node is not allowed in AI Agent flows"))
        # INFO: disabled node still sitting in the flow
        if n.get("isDisabled"):
            issues.append(("INFO", label, "node is disabled"))
    return issues
```

Checks (verified live): `hasError` flag · forbidden Code globals · dangling same-flow GoTo targets · empty Say/Question text · disabled nodes. Extend the list as new failure modes are discovered — keep each check grounded in something reliably present in `config`.

---

## Step 11 — Search across flows in a project

"Where is X used?" before changing a shared variable, URL, or sub-flow.

**Cost matters.** Fetching full config for every node in every flow is thousands of requests and will time out on a large project (~42 flows timed out at 2 min). Two tiers:

- **Fast (default):** search each flow's chart JSON — covers `label`, `comment`, `preview` (which holds Say/Question text and If conditions). **One request per flow.** Use for finding text, labels, conditions, node types.
- **Deep (scope by node type):** only when you must match inside full `config` (e.g. Code contents, HTTP URLs). Search **only** nodes of the given type — never all nodes — or it's both slow and noisy with false positives.

```python
def search_project(project_id, pattern, node_type=None):
    rx = re.compile(pattern, re.IGNORECASE)
    flows = api("GET", f"/v2.0/flows?projectId={project_id}&limit=100")
    hits = []
    for f in flows.get("_embedded", {}).get("flows", []):
        fid = f.get("_links",{}).get("self",{}).get("href","").rstrip("/").split("/")[-1]
        fname = f.get("properties", f).get("name", fid)
        chart = api("GET", f"/v2.0/flows/{fid}/chart")
        if not chart or "nodes" not in chart: continue
        for n in chart["nodes"]:
            if node_type:                       # DEEP: only the target type, full config
                if n.get("type") != node_type: continue
                cfg = api("GET", f"/v2.0/flows/{fid}/chart/nodes/{n['_id']}")
                blob = json.dumps((cfg or {}).get("config", {}))
            else:                               # FAST: chart-level fields only
                blob = json.dumps({k: n.get(k) for k in ("label","comment","preview","type")})
            m = rx.search(blob)
            if m:
                ctx = blob[max(0, m.start()-25):m.end()+25]
                hits.append((fname, n.get("type"), n.get("label") or n.get("type"), ctx))
    return hits
```

> Pitfall (hit live): in deep mode, do NOT also chart-match other node types — base64 image data in `aiAgentJob` nodes matched `/sha1/i` by coincidence while the real Code-node match was buried. Restricting deep search to the target type gives exactly the right hit.

---

## Verifying a flow by talking to it (end-to-end test)

If the flow is connected to a REST/webhook endpoint (e.g. `https://endpoint-trial.cognigy.ai/<token>`), you can drive a real conversation to verify your changes actually work — not just that the API accepted them. This is the strongest verification.

### Creating a REST endpoint for a flow (if one doesn't exist)

**Critical: endpoint creation only works under the `/new` namespace.** Since the helper's `BASE` already includes `/new`, call `api("POST", "/v2.0/endpoints", ...)` — it resolves to `/new/v2.0/endpoints`. Hitting the host *without* `/new` (plain `https://api-trial.cognigy.ai/v2.0/endpoints`) returns HTTP 500. This is the clearest case of why the whole skill defaults to `/new`.

The payload needs project/locale/flow identifiers, and **`localeId` must be the locale's `referenceId` UUID** — not the flow's `localeReference` (a 24-char mongo id). Resolving these:

- `projectId` / `entrypoint` = the flow's `projectReference` (from `GET /v2.0/flows/{flowId}`).
- `flowId` = the flow's `referenceId` (UUID).
- `localeId` = the **primary** locale's `referenceId`, found via `GET /v2.0/locales?projectId={projectReference}` and picking the entry with `primary: true`.

```python
flow = api("GET", f"/v2.0/flows/{flow_id}")
proj = flow["projectReference"]
flow_ref = flow["referenceId"]
locales = api("GET", f"/v2.0/locales?projectId={proj}&limit=50")["_embedded"]["locales"]
locale_uuid = next(l for l in locales if l.get("primary"))["referenceId"]

payload = {
    "agentId": "", "channel": "rest", "customIcon": "",
    "entrypoint": proj, "flowId": flow_ref, "localeId": locale_uuid,
    "name": "My Test Endpoint", "projectId": proj, "targetType": "flow",
}
res = api("POST", "/v2.0/endpoints", payload)   # BASE has /new → /new/v2.0/endpoints; 201 on success
token = res["URLToken"]   # use as https://endpoint-trial.cognigy.ai/{token}
```

Common 400s: `localeId should be of format 'uuid'` means you passed the mongo `localeReference` instead of the locale `referenceId`. A 500 means your `BASE` is missing `/new` (you hit plain `/v2.0/endpoints`).

### FIRST: verify the endpoint actually routes to your flow

**Never test blindly** — an endpoint token may point at a different flow, so you could "verify" a flow you never touched. Before sending any messages, confirm the endpoint's attached flow matches the flow you edited:

```python
TOKEN = "<token from the endpoint URL>"   # the path segment after the host

# 1. Find the endpoint whose URLToken matches (paginate; list view omits flow link)
ep_id = None
skip = 0
while ep_id is None:
    r = api("GET", f"/v2.0/endpoints?limit=100&skip={skip}")
    eps = r.get("_embedded", {}).get("endpoints", [])
    for e in eps:
        if e.get("properties", e).get("URLToken") == TOKEN:
            ep_id = e.get("_links",{}).get("self",{}).get("href","").rstrip("/").split("/")[-1]
            break
    if len(eps) < 100: break
    skip += 100
assert ep_id, "No endpoint found for that token"

# 2. Endpoint's flowId is a flow REFERENCEID (UUID), not the 24-char _id.
endpoint_flow_ref = api("GET", f"/v2.0/endpoints/{ep_id}")["flowId"]
flow_ref = api("GET", f"/v2.0/flows/{flow_id}")["referenceId"]

assert endpoint_flow_ref == flow_ref, (
    f"Endpoint routes to {endpoint_flow_ref}, but you edited {flow_ref} — WRONG ENDPOINT")
print("Endpoint confirmed to route to this flow. Safe to test.")
```

If they don't match, STOP and tell the user the endpoint points at a different flow — do not report test results as if they validated the edited flow.

### Then drive the conversation

```python
import json, uuid, urllib.request, time

URL = "https://endpoint-trial.cognigy.ai/<token>"
session = str(uuid.uuid4())   # one session = one conversation thread

def send(text):
    payload = {"userId": "tester", "sessionId": session, "text": text, "data": {}}
    req = urllib.request.Request(URL, data=json.dumps(payload).encode(),
                                  headers={"Content-Type": "application/json"}, method="POST")
    with urllib.request.urlopen(req, timeout=30) as r:
        resp = json.loads(r.read().decode())
    # Plain text is in resp["text"]; quick-reply/rich messages are in resp["data"]["text"][0]
    return resp

send("")        # empty first message triggers the entry flow
send("Encode")  # then send each user turn in sequence
```

Notes:
- Reuse the **same `sessionId`** across turns to keep conversation state; use a **fresh** one per test scenario.
- The bot reply is in `resp["text"]`; for quick-reply/gallery messages `text` is empty and the content is under `resp["data"]["text"]` / `resp["data"]["_cognigy"]`.
- **Test every branch** of any logic you added (yes/no, condition true/false) with separate sessions.
- **Debugging a condition that won't fire:** temporarily echo the runtime value (e.g. `{{input.result}}`) in a downstream Say node, run one turn to read it, then restore. This is how you discover exact field shapes the API doesn't document.

## Pattern: safe external-call scaffolding

When a flow needs to call a real external API (HTTP Request node) that doesn't exist yet, isn't safe to hit, or has no credentials, build it so it's testable end-to-end without making the live call:

1. Create the **HTTP Request node** with the real method/URL/body, then **disable it** (`{"isDisabled": True}`). It documents intent without firing.
2. Immediately after it, add a **Code node placeholder** that writes the response the HTTP node *would* have returned into context, e.g. `context.blockResult = {status: 'blocked', reference: 'MOCK-REF-12345'};`. Downstream nodes read from context and behave identically to the live path.

This keeps the flow runnable and verifiable now; going live later is just enabling the HTTP node and deleting the placeholder. Default to this whenever you scaffold a flow that touches an external system you can't safely call during the build.

## Workflow rules

1. **Use the chart endpoint** (`GET /chart`) for reading structure — not the paginated nodes endpoint.
2. **Always read a node before updating** — merge changes, never replace the whole config.
3. **Never use `insertBefore`** — use `append` on the preceding node.
4. **If nodes auto-create Then/Else** — fetch chart after creation to get their IDs.
5. **GoTo needs referenceId (UUID), not _id** — fetch flow object and chart nodes to get them.
6. **Say and Question `text` fields are always arrays**, not strings.
7. **Code node is pure ECMAScript** — no Node.js globals at all.
8. **Question answers live in `input.result`** — yesNo is a boolean (`true`/`false`), text is a string.
9. **Confirm destructive operations** — ask before delete or move.
10. **After any mutation**, re-fetch and display the chart so the user sees current state.
11. **Verify behaviour, not just acceptance** — if an endpoint is available, drive a real conversation through every branch before declaring done.
12. **Confirm the endpoint routes to your flow before testing** — match the endpoint's `flowId` (a referenceId) to the flow's `referenceId`. Never report results from an endpoint you haven't confirmed.
13. **`isDisabled` is a top-level node field**, not config — PATCH it at the node root.
14. **Scaffold external calls disabled + a placeholder Code node behind them** so the flow is testable without firing live APIs.
15. **Always use the `/new` namespace** — `BASE` ends in `/new`, so paths start at `/v2.0/...`. Endpoint creation (`/v2.0/endpoints`) only works under `/new`; without it you get 500. `localeId` for endpoint creation is the primary locale's `referenceId` UUID.
16. **Create flows with `POST /v2.0/flows` (projectId + name only)** — no `localeId`. They come with start/end nodes already.
17. **AI Agent (`aiAgentJob`) prompt is inline** (`name`/`description`/`instructions`). It auto-creates Default + Tool children — delete the Tool for a tool-free agent.
18. **Give each AI Agent its own persona** — create one with `POST /v2.0/aiagents` (`projectId` + `name`) and use its `referenceId` as the node's `aiAgent`, rather than borrowing another flow's.
19. **For voice bots, prepend a Set Session Config node** (`setSessionConfig`, `@cognigy/voicegateway2`) before the AI Agent: `start → setSessionConfig → aiAgentJob`. It sets STT/TTS, barge-in, endpointing, no-input, and DTMF for the session.
20. **Tools:** add an `aiAgentJobTool` child to the agent, build its handler branch ending in `aiAgentToolAnswer`. Args arrive at `input.aiAgent.toolArgs.<param>`; the answer is fed back to the LLM. Force usage via `instructions` since `toolChoice: "auto"` lets the model skip it.
21. **No Question or GoTo nodes in tool-assisted / AI Agent flows.** These node types are incompatible with the AI Agent conversation model — use Code, HTTP Request, If, and aiAgentToolAnswer instead.
22. **All Say nodes must set `preventTranscript`.** Use `preventTranscript: true` to keep output out of the transcript; set it to `false` on a terminal branch message the user must actually see (a terminal `preventTranscript: true` Say is silent in the REST response). The field is `preventTranscript`, never `excludeFromTranscript` (HTTP 400).
23. **`aiAgentToolAnswer` does NOT render Handlebars block helpers** (`{{#if}}`/`{{#each}}`). Only simple `{{context.x}}` interpolation works; a `{{#if}}` block silently yields nothing, so the LLM receives an answerless result and assumes failure. Build the full answer string in a Code node (e.g. `context.toolMessage = ...`) and set the answer to a plain `{{context.toolMessage}}`.
24. **Cross-flow `goTo` uses `executionMode: "continue"`**, which runs the target flow inline and (for REST endpoints) flows straight into the agent in the same turn. Because a REST endpoint always re-enters its bound flow's start node every turn, gate multi-turn routing with a *router If at the top of the entry flow* that checks a **`profile`** flag (e.g. `profile.aiConsent === true`) — `context` does not reliably persist across REST turns, `profile` does. Set the flag in a Code node (`profile.aiConsent = true;`) on the consenting branch.
