# Code Recipes — Ready-to-Paste Snippets

Copy-paste recipes for the 7 use case patterns from `use-cases.md`. All recipes assume `BUDDYPRO_API_KEY` is set as an environment variable.

Code is provided in three flavors per pattern: **bash/curl**, **Python (requests)**, **Node.js (fetch)**. Pick the one matching the user's stack.

---

## Pattern 1 — Single owner call

### bash
```bash
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "Summarize my top 3 content trends"}]}' \
  | jq -r '.choices[0].message.content'
```

### Python
```python
import os, requests

response = requests.post(
    "https://api.buddypro.ai/v1/chat/completions",
    headers={
        "Authorization": f"Bearer {os.environ['BUDDYPRO_API_KEY']}",
        "Content-Type": "application/json",
    },
    json={"messages": [{"role": "user", "content": "Summarize my top 3 content trends"}]},
    timeout=120,
)
response.raise_for_status()
print(response.json()["choices"][0]["message"]["content"])
```

### Node.js (built-in fetch, Node 18+)
```javascript
const response = await fetch("https://api.buddypro.ai/v1/chat/completions", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.BUDDYPRO_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    messages: [{ role: "user", content: "Summarize my top 3 content trends" }],
  }),
});

if (!response.ok) throw new Error(`HTTP ${response.status}`);
const data = await response.json();
console.log(data.choices[0].message.content);
```

---

## Pattern 2 — Multi-tenant Express server

### Node.js (Express)
```javascript
import express from "express";
const app = express();
app.use(express.json());

app.post("/chat", async (req, res) => {
  const { customerId, message } = req.body;
  if (!customerId || !message) {
    return res.status(400).json({ error: "customerId and message required" });
  }

  const response = await fetch("https://api.buddypro.ai/v1/chat/completions", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.BUDDYPRO_API_KEY}`,
      "Content-Type": "application/json",
      "X-Client-Request-Id": `${customerId}-${Date.now()}`,
    },
    body: JSON.stringify({
      user: customerId,                 // ← stable per-customer ID
      messages: [{ role: "user", content: message }],
    }),
  });

  if (!response.ok) {
    const err = await response.json().catch(() => ({}));
    return res.status(response.status).json(err);
  }

  const data = await response.json();
  res.json({ reply: data.choices[0].message.content });
});

app.listen(3000, () => console.log("Listening on :3000"));
```

### Python (FastAPI)
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import os, requests, time

app = FastAPI()

class ChatIn(BaseModel):
    customerId: str
    message: str

@app.post("/chat")
def chat(payload: ChatIn):
    response = requests.post(
        "https://api.buddypro.ai/v1/chat/completions",
        headers={
            "Authorization": f"Bearer {os.environ['BUDDYPRO_API_KEY']}",
            "Content-Type": "application/json",
            "X-Client-Request-Id": f"{payload.customerId}-{int(time.time())}",
        },
        json={
            "user": payload.customerId,
            "messages": [{"role": "user", "content": payload.message}],
        },
        timeout=120,
    )
    if not response.ok:
        raise HTTPException(response.status_code, response.json().get("error"))
    return {"reply": response.json()["choices"][0]["message"]["content"]}
```

**Production checklist for Pattern 2:**
- ✅ Use a stable customer ID (UUID, hashed email — NEVER username)
- ✅ Add `X-Client-Request-Id` for tracing
- ✅ Implement retry with backoff on 429/5xx
- ✅ Rate-limit incoming requests at your edge (your customers don't see the 30/min budget directly)
- ✅ Log `x-request-id` from response for support tickets

---

## Pattern 3 — Batch quality audit

### Python
```python
import os, json, time, requests
from pathlib import Path

QUESTIONS = Path("test-questions.txt").read_text().splitlines()
RESULTS = []

for i, question in enumerate(QUESTIONS):
    response = requests.post(
        "https://api.buddypro.ai/v1/chat/completions",
        headers={
            "Authorization": f"Bearer {os.environ['BUDDYPRO_API_KEY']}",
            "Content-Type": "application/json",
        },
        json={
            "x_buddy_saveToHistory": False,             # ← stateless
            "messages": [{"role": "user", "content": f"/investigateAnswer:{question}"}],
        },
        timeout=120,
    )

    if response.status_code == 429:
        time.sleep(60)                                  # rate limit, back off
        continue

    response.raise_for_status()
    data = response.json()
    RESULTS.append({
        "question": question,
        "answer": data["choices"][0]["message"]["content"],
        "request_id": response.headers.get("x-request-id"),
    })

    print(f"[{i+1}/{len(QUESTIONS)}] {question[:50]}…")
    time.sleep(2.5)                                     # 24 req/min, safe

Path("audit-results.json").write_text(json.dumps(RESULTS, indent=2, ensure_ascii=False))
print(f"Saved {len(RESULTS)} answers to audit-results.json")
```

### bash
```bash
#!/bin/bash
set -euo pipefail

mapfile -t QUESTIONS < test-questions.txt
echo "[" > audit-results.json
FIRST=1

for q in "${QUESTIONS[@]}"; do
  ans=$(curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
    -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg q "$q" '{
      x_buddy_saveToHistory: false,
      messages: [{role: "user", content: ("/investigateAnswer:" + $q)}]
    }')" | jq '.choices[0].message.content')

  [ "$FIRST" -eq 0 ] && echo "," >> audit-results.json
  jq -n --arg q "$q" --argjson a "$ans" '{question: $q, answer: $a}' >> audit-results.json
  FIRST=0
  sleep 2.5
done

echo "]" >> audit-results.json
```

---

## Pattern 4 — Structured output wrapper

### Python (with retry on parse failure)
```python
import os, json, requests

SYSTEM_PROMPT = """You are a JSON API. Respond ONLY with valid JSON in this format:
{
  "summary": "<one sentence>",
  "keyPoints": ["<point 1>", "<point 2>", "<point 3>"],
  "confidence": <0-1 float>
}
No prose, no markdown, just JSON."""

def ask_structured(question: str, retries: int = 2) -> dict:
    for attempt in range(retries + 1):
        response = requests.post(
            "https://api.buddypro.ai/v1/chat/completions",
            headers={
                "Authorization": f"Bearer {os.environ['BUDDYPRO_API_KEY']}",
                "Content-Type": "application/json",
            },
            json={
                "x_buddy_systemPrompt": SYSTEM_PROMPT,
                "x_buddy_systemPromptMode": "replace",
                "x_buddy_saveToHistory": False,
                "messages": [{"role": "user", "content": question}],
            },
            timeout=120,
        )
        response.raise_for_status()
        content = response.json()["choices"][0]["message"]["content"]

        try:
            # The bot may add code-fences; strip them
            content = content.strip().removeprefix("```json").removeprefix("```").removesuffix("```").strip()
            return json.loads(content)
        except json.JSONDecodeError:
            if attempt == retries:
                raise ValueError(f"Bot didn't return valid JSON after {retries+1} tries: {content[:200]}")

# Usage
result = ask_structured("What is your best framework for pricing?")
print(result["summary"])
for p in result["keyPoints"]:
    print(f"- {p}")
```

---

## Pattern 5 — Voice loop

### Python
```python
import os, base64, requests
from pathlib import Path

def voice_chat(audio_path: str, customer_id: str = None) -> tuple[str, bytes]:
    """Send an audio question, get text transcript + TTS audio reply."""
    audio_b64 = base64.b64encode(Path(audio_path).read_bytes()).decode()

    payload = {
        "modalities": ["text", "audio"],
        "audio": {"format": "mp3"},
        "messages": [{
            "role": "user",
            "content": [{
                "type": "input_audio",
                "input_audio": {"data": audio_b64, "format": "mp3"}
            }]
        }]
    }
    if customer_id:
        payload["user"] = customer_id

    response = requests.post(
        "https://api.buddypro.ai/v1/chat/completions",
        headers={
            "Authorization": f"Bearer {os.environ['BUDDYPRO_API_KEY']}",
            "Content-Type": "application/json",
        },
        json=payload,
        timeout=180,                                    # voice gen takes longer
    )
    response.raise_for_status()
    msg = response.json()["choices"][0]["message"]

    text_reply = msg["content"]
    audio_reply = base64.b64decode(msg["audio"]["data"]) if "audio" in msg else None
    return text_reply, audio_reply

# Usage
text, audio = voice_chat("question.mp3", customer_id="user-42")
print(f"Bot said: {text}")
if audio:
    Path("reply.mp3").write_bytes(audio)
    print("TTS reply saved to reply.mp3")
```

---

## Pattern 6 — Image critique

### Python (URL-based image)
```python
import os, requests

def critique_image(image_url: str, instruction: str, customer_id: str = None) -> str:
    payload = {
        "messages": [{
            "role": "user",
            "content": [
                {"type": "text", "text": instruction},
                {"type": "image_url", "image_url": {"url": image_url}}
            ]
        }]
    }
    if customer_id:
        payload["user"] = customer_id

    response = requests.post(
        "https://api.buddypro.ai/v1/chat/completions",
        headers={
            "Authorization": f"Bearer {os.environ['BUDDYPRO_API_KEY']}",
            "Content-Type": "application/json",
        },
        json=payload,
        timeout=120,
    )
    response.raise_for_status()
    return response.json()["choices"][0]["message"]["content"]

# Usage
feedback = critique_image(
    "https://example.com/landing-mockup.png",
    "Critique this landing page based on your conversion principles",
    customer_id="designer-7",
)
print(feedback)
```

### Python (base64-embedded image, no URL)
```python
import base64, requests
from pathlib import Path

def critique_local_image(image_path: str, instruction: str) -> str:
    img_b64 = base64.b64encode(Path(image_path).read_bytes()).decode()
    mime = "image/png" if image_path.endswith(".png") else "image/jpeg"
    data_url = f"data:{mime};base64,{img_b64}"

    response = requests.post(
        "https://api.buddypro.ai/v1/chat/completions",
        headers={
            "Authorization": f"Bearer {os.environ['BUDDYPRO_API_KEY']}",
            "Content-Type": "application/json",
        },
        json={
            "messages": [{
                "role": "user",
                "content": [
                    {"type": "text", "text": instruction},
                    {"type": "image_url", "image_url": {"url": data_url}}
                ]
            }]
        },
        timeout=120,
    )
    response.raise_for_status()
    return response.json()["choices"][0]["message"]["content"]
```

---

## Pattern 7 — Daily admin script

### bash (cron-friendly)
```bash
#!/bin/bash
# Run via cron: 0 9 * * * /path/to/buddypro-daily-report.sh
set -euo pipefail

source ~/.zshrc                                         # load BUDDYPRO_API_KEY

OUTPUT=$(mktemp)
{
  echo "BuddyPro Daily Report — $(date)"
  echo "================================"
  echo
  echo "## Stats"
  curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
    -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"messages": [{"role": "user", "content": "/stats"}]}' \
    | jq -r '.choices[0].message.content'
  echo
  echo "## Setup health"
  curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
    -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"messages": [{"role": "user", "content": "/checkSetup"}]}' \
    | jq -r '.choices[0].message.content'
  echo
  echo "## Active invites"
  curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
    -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"messages": [{"role": "user", "content": "/listInvites"}]}' \
    | jq -r '.choices[0].message.content'
} > "$OUTPUT"

mail -s "BuddyPro daily $(date +%Y-%m-%d)" you@example.com < "$OUTPUT"
rm "$OUTPUT"
```

---

## Production helpers

### Retry with exponential backoff (Python)
```python
import time, requests

def call_buddypro(payload: dict, max_retries: int = 4) -> dict:
    for attempt in range(max_retries):
        try:
            r = requests.post(
                "https://api.buddypro.ai/v1/chat/completions",
                headers={
                    "Authorization": f"Bearer {os.environ['BUDDYPRO_API_KEY']}",
                    "Content-Type": "application/json",
                },
                json=payload,
                timeout=120,
            )
            if r.status_code == 429 or r.status_code >= 500:
                wait = 2 ** attempt
                time.sleep(wait)
                continue
            r.raise_for_status()
            return r.json()
        except requests.RequestException:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
    raise RuntimeError("All retries exhausted")
```

### Token rotation (Node.js, multiple keys)
```javascript
const KEYS = process.env.BUDDYPRO_API_KEYS.split(",");
let keyIndex = 0;

function nextKey() {
  const key = KEYS[keyIndex];
  keyIndex = (keyIndex + 1) % KEYS.length;
  return key;
}

// Each call uses a round-robin key — multiplies the 30/min budget
async function callBuddyPro(payload) {
  const response = await fetch("https://api.buddypro.ai/v1/chat/completions", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${nextKey()}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(payload),
  });
  return response.json();
}
```

**Caveat:** each key has its own owner-profile (or test-profile) context. If you use multi-tenant (`user` field), all keys must be generated from the same profile to avoid leakage.

---

## Where to paste these recipes

- **Bash recipes** go into `.sh` scripts; make executable with `chmod +x`.
- **Python recipes** need `pip install requests` (and `fastapi uvicorn` for Pattern 2 server).
- **Node recipes** need Node 18+ for built-in `fetch`. For older Node, install `node-fetch`. Pattern 2 server needs `npm install express`.

For all of them: set `BUDDYPRO_API_KEY` as env var (see `getting-started.md` Stage 1).

*Last updated: 2026-05-07 (v0.2.x)*
