# cognigy-email

Sends an email to Phil's inbox via the Cognigy HTTP endpoint.

## Step 1: Determine what to send

**Given a brief or topic** (no full email written): compose the email yourself. Match the tone to the content — casual for personal reminders, professional for work items. Pick a clear, scannable subject line. If the subject is ambiguous from the brief, confirm with the user before sending.

**Given full content** (explicit subject + body): use it as-is. Clean up obvious typos only if they'd cause confusion.

## Step 2: Write the HTML body

Keep it clean and email-client-safe:

- Outer wrapper: a single `<div>`
- Paragraphs: `<p>`
- Lists: `<ul>` / `<li>`
- Emphasis: `<strong>` or `<em>`
- No `style=` attributes unless the user asked for formatting

When composing from a brief, sign off as "Claude". Keep it short — this is a notification to Phil from himself, not a formal memo.

**Example** — brief: *"email me that the Q2 report is ready for review"*

```html
<div>
  <p>Hi Phil,</p>
  <p>Just a heads-up: the <strong>Q2 report</strong> is ready for your review.</p>
  <p>– Claude</p>
</div>
```

## Step 3: Send via the Cognigy endpoint

Write a Python script to `/tmp/send_cognigy_email.py` using a bash heredoc, substituting the actual subject and body into the SUBJECT and BODY placeholders. Then run it.

**Template:**

```python
import json, uuid, urllib.request, urllib.error, sys

subject = """SUBJECT"""
body = """BODY"""

payload = {
    "userId": "claude",
    "sessionId": str(uuid.uuid4()),
    "text": "",
    "data": {"subject": subject, "body": body}
}
url = "https://endpoint-trial.cognigy.ai/00a680a0503b4d3676ebbbf0de77294b83cb2f1bc446d165d3605a180f29536f"
req = urllib.request.Request(
    url,
    data=json.dumps(payload).encode("utf-8"),
    headers={"Content-Type": "application/json"},
    method="POST"
)
try:
    with urllib.request.urlopen(req, timeout=30) as resp:
        print(f"Sent! Status {resp.status}: {resp.read().decode()}")
except urllib.error.HTTPError as e:
    print(f"HTTP Error {e.code}: {e.reason}", file=sys.stderr)
    sys.exit(1)
except Exception as e:
    print(f"Failed: {e}", file=sys.stderr)
    sys.exit(1)
```

**Bash commands to write and run:**

```bash
cat > /tmp/send_cognigy_email.py << 'PYEOF'
<filled-in script here>
PYEOF
python3 /tmp/send_cognigy_email.py
```

> **Tip on quoting:** Triple-quoted Python strings (`"""..."""`) handle single quotes, double quotes, and newlines safely. The only thing to avoid is a literal `"""` in the content itself — extremely unlikely in practice.

## Step 4: Confirm

Tell Phil the email was sent and echo the subject line back. If the HTTP call fails (non-200 status or network error), show the error message and offer to retry.
