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

## Pattern 8 — Multi-step Deep Research (the powerful pattern)

Goal: extract a comprehensive document from the bot's knowledge base across many angles, with gap-check and synthesis. Inspired by `claude-buddy-connection` orchestrator, adapted for Owner-API-only.

### Python — full implementation

```python
import os, time, json, requests
from typing import Dict, List

API_URL = "https://api.buddypro.ai/v1/chat/completions"
RATE_LIMIT_SLEEP = 3.0  # 30/min limit, 3s = safe ~20/min

def call_buddypro(user: str, message: str, system_prompt: str = None, stateless: bool = False) -> str:
    payload = {"user": user, "messages": [{"role": "user", "content": message}]}
    if system_prompt:
        payload["x_buddy_systemPrompt"] = system_prompt
        payload["x_buddy_systemPromptMode"] = "add"
    if stateless:
        payload["x_buddy_saveToHistory"] = False
    
    r = requests.post(
        API_URL,
        headers={"Authorization": f"Bearer {os.environ['BUDDYPRO_API_KEY']}",
                 "Content-Type": "application/json"},
        json=payload,
        timeout=120,
    )
    r.raise_for_status()
    time.sleep(RATE_LIMIT_SLEEP)
    return r.json()["choices"][0]["message"]["content"]


# ============================================================
# STEP 1 — Planning: get research angles from the bot
# ============================================================
def plan_angles(session_user: str, topic: str, depth: int = 8) -> List[str]:
    """Ask the bot to identify N angles to research."""
    response = call_buddypro(
        session_user,
        f"""I want to deeply research the topic: "{topic}"

List exactly {depth} DISTINCT angles I should explore. Each should cover a different:
- Perspective (beginner vs advanced, owner vs customer, strategic vs tactical)
- Time horizon (immediate vs long-term)
- Stakeholder (who benefits, who pushes back)
- Framework or methodology applicable
- Failure mode to anticipate

Output: numbered list, ONE line per angle, no extra prose.""",
        system_prompt="You are now in research-planning mode. Be comprehensive but focused."
    )
    
    # Parse numbered list
    angles = []
    for line in response.split('\n'):
        line = line.strip()
        if line and line[0].isdigit():
            # Strip "1. " or "1) " prefix
            content = line.split('.', 1)[-1].split(')', 1)[-1].strip()
            if content:
                angles.append(content)
    
    return angles[:depth]


# ============================================================
# STEP 2 — Interview loop: deep on each angle
# ============================================================
def interview_angles(session_user: str, topic: str, angles: List[str]) -> Dict[str, str]:
    """For each angle, get deep insights from the bot."""
    answers = {}
    
    for i, angle in enumerate(angles, 1):
        print(f"  [{i}/{len(angles)}] {angle[:60]}...")
        
        question = f"""Topic context: '{topic}'

Now go deep on THIS specific angle: {angle}

Provide:
- Your strongest insight on this angle (the thing most people miss)
- A specific framework or method you'd use
- ONE concrete example or case
- ONE common mistake to avoid

Length: 200-400 words. Direct, no fluff."""
        
        answers[angle] = call_buddypro(session_user, question)
    
    return answers


# ============================================================
# STEP 3 — Gap check: find missing pieces
# ============================================================
def find_gaps(session_user: str, topic: str, current_answers: Dict[str, str], max_gaps: int = 3) -> List[str]:
    """Show the bot what we've collected, ask what's missing."""
    summary = "\n\n".join(
        f"### {angle}\n{answer[:300]}..."
        for angle, answer in current_answers.items()
    )
    
    response = call_buddypro(
        session_user,
        f"""Topic: '{topic}'

Here's the research collected so far:

{summary}

What {max_gaps} CRITICAL questions remain unanswered that would prevent this from being a complete reference document? 

For each gap:
- Phrase it as a specific question
- Explain in one line WHY it matters

Output: numbered list of {max_gaps} gaps."""
    )
    
    gaps = []
    for line in response.split('\n'):
        line = line.strip()
        if line and line[0].isdigit():
            gaps.append(line.split('.', 1)[-1].strip())
    return gaps[:max_gaps]


# ============================================================
# STEP 4 — Fill gaps: ask the bot the gap questions
# ============================================================
def fill_gaps(session_user: str, gaps: List[str]) -> Dict[str, str]:
    """Get answers to gap questions."""
    gap_answers = {}
    for gap in gaps:
        gap_answers[gap] = call_buddypro(session_user, gap)
    return gap_answers


# ============================================================
# STEP 5 — Synthesis: weave the document
# ============================================================
def synthesize_document(topic: str, all_answers: Dict[str, str], output_format: str = "markdown_report") -> str:
    """
    Claude Code (or another LLM) does the synthesis.
    BuddyPro provides the raw knowledge; the writing is done elsewhere.
    
    For use INSIDE Claude Code: when this function is called, you (Claude Code)
    write the document yourself using all_answers as source material, in the
    format specified.
    """
    # In a real Claude Code skill execution, this is where you would
    # produce the final document using your own writing capability,
    # using `all_answers` as the source-of-truth knowledge.
    # 
    # For external Python automation, you'd call another LLM here.
    
    if output_format == "markdown_report":
        return synthesize_as_markdown_report(topic, all_answers)
    elif output_format == "executive_brief":
        return synthesize_as_executive_brief(topic, all_answers)
    elif output_format == "blog_post":
        return synthesize_as_blog_post(topic, all_answers)
    elif output_format == "structured_json":
        return json.dumps({"topic": topic, "research": all_answers}, indent=2, ensure_ascii=False)
    else:
        raise ValueError(f"Unknown format: {output_format}")


def synthesize_as_markdown_report(topic: str, answers: Dict[str, str]) -> str:
    """Default format: structured report with sections per angle."""
    md = f"# {topic} — Research Report\n\n"
    md += f"_Generated from {len(answers)} research angles via deep-research pattern._\n\n"
    md += "## Executive Summary\n\n"
    md += "[Claude Code: write 3-5 sentence summary distilling the key insight across all angles]\n\n"
    md += "## Detailed Findings\n\n"
    for angle, content in answers.items():
        md += f"### {angle}\n\n{content}\n\n"
    md += "## Synthesis & Recommendations\n\n"
    md += "[Claude Code: integrate the angles into 3-5 unified recommendations]\n"
    return md


# ============================================================
# ORCHESTRATOR: full deep research pipeline
# ============================================================
def deep_research(topic: str, depth: int = 8, output_format: str = "markdown_report") -> str:
    """
    Run a complete deep research pipeline.
    
    Args:
        topic: research topic, e.g. "pricing strategy for SaaS coaching"
        depth: how many angles to research (4-12 recommended)
        output_format: "markdown_report" | "executive_brief" | "blog_post" | "structured_json"
    
    Returns:
        Final document as a string.
    """
    session_user = f"research-{topic.replace(' ', '-').lower()[:40]}-{int(time.time())}"
    
    print(f"\n=== Deep Research: {topic} ===")
    print(f"Session: {session_user}\n")
    
    print("Step 1/4: Planning angles...")
    angles = plan_angles(session_user, topic, depth)
    print(f"  → {len(angles)} angles identified\n")
    
    print("Step 2/4: Interviewing each angle...")
    answers = interview_angles(session_user, topic, angles)
    print(f"  → {len(answers)} angles answered\n")
    
    print("Step 3/4: Gap check...")
    gaps = find_gaps(session_user, topic, answers, max_gaps=3)
    print(f"  → {len(gaps)} gaps found")
    if gaps:
        gap_answers = fill_gaps(session_user, gaps)
        for q, a in gap_answers.items():
            answers[f"FOLLOW-UP: {q}"] = a
        print(f"  → {len(gap_answers)} gaps filled\n")
    
    print("Step 4/4: Synthesis...")
    document = synthesize_document(topic, answers, output_format)
    print(f"  → Document ready: {len(document)} chars\n")
    
    return document


# ============================================================
# REFINEMENT: edit document based on owner feedback
# ============================================================
def refine_document(session_user: str, current_document: str, owner_feedback: str) -> str:
    """
    When owner says "make it shorter / add example / change tone":
    - Lightweight edits → Claude Code edits directly
    - New content needed → call BuddyPro for additional material in same session
    
    This function shows the API call for getting new content. The document
    weaving is done by Claude Code.
    """
    if "more example" in owner_feedback.lower() or "add" in owner_feedback.lower():
        # Need new content from the bot
        new_content = call_buddypro(
            session_user,
            f"In our research session, owner now wants: {owner_feedback}. "
            f"Provide 200 words of new content matching this request."
        )
        return new_content  # Claude Code weaves into the document
    else:
        # Lightweight edit — Claude Code does it directly, no bot call
        return None  # Signal: edit yourself, don't call bot


# ============================================================
# Usage
# ============================================================
if __name__ == "__main__":
    document = deep_research(
        topic="SaaS pricing strategy for solo founders",
        depth=8,
        output_format="markdown_report",
    )
    
    with open("research-output.md", "w") as f:
        f.write(document)
    print("Saved to research-output.md")
```

### Output format templates

```python
def synthesize_as_executive_brief(topic, answers):
    """Format: 1-page exec brief with 3-bullet takeaways per section."""
    return f"""# {topic} — Executive Brief

## TL;DR
[Claude Code: 3 sentences max]

## Key Takeaways
{chr(10).join('- [extract 1-line insight from each]' for _ in answers)}

## Recommended Actions
1. [most impactful action]
2. [secondary action]
3. [defensive/exploration action]

## Reference Material
{chr(10).join(f'**{a}** — [1-sentence summary]' for a in answers)}
"""


def synthesize_as_blog_post(topic, answers):
    """Format: long-form article, sections flow into narrative."""
    return f"""# {topic}

[Claude Code: write a hook paragraph that draws the reader in]

[Body sections — weave the angles into coherent narrative, 3-5 sections]

[Closing call to action]
"""
```

### Why this works for non-tech owners

The owner doesn't run the Python — Claude Code runs it for them. They say in natural language:

> *„Research my pricing strategy across all my knowledge — make me a 5-section markdown report with frameworks, examples, and 3 recommendations."*

Claude Code interprets this as:
- topic = „pricing strategy"
- output_format = „markdown_report"
- depth = ~8 angles
- Specific format requirements → embedded in synthesize step

Then runs the pipeline (~30s for 8 angles × 3s each + planning + gap check + synthesis), and gives the owner the polished document.

**Total cost:** ~$0.50/research session at $0.05/call × 10 calls average. **Total time:** ~30-40 seconds.

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
