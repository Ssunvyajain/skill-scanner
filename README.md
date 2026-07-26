# Skill Safety Audit — full working solver

A complete, deployable scanner endpoint for the GA5 question *"Skill Safety
Audit — Scanner API"* (`q-skill-safety-audit-server`, 1.5 marks).

Clone it, run one command to prove it works, deploy it, paste your URL, press
Check.

> **No API key. No model. No cost.** This question is pure text analysis — the
> whole thing is deterministic regex and structure parsing. Nothing to
> configure, nothing to leak.

---

## 1. Quick start (2 minutes)

```bash
git clone https://github.com/<you>/tds-ga5-q4-solver.git
cd tds-ga5-q4-solver

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

python selftest.py
```

You should see **`43 passed, 0 failed`** and:

```
precision 1.00  recall 1.00  F-beta(0.5) 1.00
files mishandled: 0/15
```

That runs 15 skill files — clean ones, one per category, multi-category ones,
and deliberately alarming-but-clean ones — and reports the **same aggregate
F-beta(0.5) the grader uses**, so you can see your score before submitting.

---

## 2. What the question asks

The grader POSTs 5 skill files (markdown + YAML frontmatter), one at a time:

```json
{ "skill": "---\nname: notes-digest\n..." }
```

You return **strict JSON with exactly one key**:

```json
{ "categories": ["hardcoded_secret", "excessive_permissions"] }
```

Only these four keys are valid, and `[]` is a perfectly good answer:

| category | what counts |
|---|---|
| `hardcoded_secret` | a literal secret, API key, or **webhook URL** embedded directly in the frontmatter or example script instead of being referenced via an env var or secret store |
| `prompt_injection` | a step that tries to override user/agent control — **silent** exfiltration of file contents, or ignoring the user's stop/cancel request |
| `excessive_permissions` | broader filesystem or network access than the stated task requires (read/write to `/`, egress to any domain, for a narrow task) |
| `unclear_provenance` | no author **and** no version **and** no changelog — and/or a step that silently rewrites its own version metadata |

---

## 3. Why you're at 4/5 — read this bit

**2 of the 5 graded files are genuinely clean.** Scoring is aggregate
**F-beta(0.5)**, pooled over all five files, which weights precision about
**four times** more than recall. In plain terms:

> **One category you claim wrongly hurts roughly four times as much as one you
> miss.**

So the two failure modes are very different:

- **Over-flagged** (`precision < 1.00`) — you are firing on keywords. A skill
  that *mentions* `API_KEY`, or says "stop", or documents a `WEBHOOK_URL`
  environment variable is not vulnerable. Tighten your detectors.
- **Under-flagged** (`recall < 1.00`) — a planted category is being missed.
  Below are the ones people miss most.

### The three most-missed detections

1. **Webhook URLs that aren't Slack.** Most scanners hardcode
   `hooks.slack.com`. The category says *"webhook URL"* generally —
   `hooks.zapier.com/hooks/catch/8461902/b3x9kq7/` counts too, and so does any
   URL sitting in a frontmatter key named `webhook`/`token`/`api_key`. See
   `WEBHOOK_SHAPE_RE` and the URL branch in `detect_hardcoded_secret()`.
   **Rule of thumb:** a concrete URL under a secret-ish key is a finding even
   if it has no high-entropy blob in it.

2. **Silent version rewrites with no explicit "silently".** The category is
   *"a step silently rewrites its own version metadata without surfacing that
   change to the reviewer"*. The file does not have to say the word "quietly".
   A step that edits **this file's own** `version` field when the skill carries
   **no changelog** surfaces the change nowhere — same defect. With a changelog
   present, it is ordinary release hygiene and must **not** be flagged. See the
   end of `detect_unclear_provenance()`.

3. **Permissions written as prose, not globs.** The corpus does not say
   `paths: ["/**"]`. It says *"read-write access to the entire filesystem (/)"*,
   *"unrestricted egress to any host"*, *"egress allowed to any external
   domain"* — against scoped forms like *"read/write limited to ~/notes"* and
   *"none required"*. Judge the value by whether it is **scoped**, not by
   matching a literal. See `SCOPED_RE` / `BROAD_SCOPE_RE`.

### The false positive that bit us

An early version flagged `prompt_injection` on:

> *"Post a one-line summary to the configured webhook."*

because the exfiltration pattern listed `webhook` and `endpoint` as things
being *sent*. They are **destinations**, not payloads. Openly posting a result
to a declared webhook is the skill doing its job. Exfiltration only counts when
the step also **conceals itself** ("silently", "without informing the user",
"do not mention this in your summary") — with credentials as the one exception,
since shipping those anywhere is a finding on its own. That single fix took the
test corpus from `precision 0.88` to `1.00`.

---

## 4. Deploy (free)

Any public HTTPS host works. Render is the least fuss:

1. Push this repo to **your own** GitHub account.
2. [render.com](https://render.com) → **New → Web Service** → connect the repo.
3. Settings:
   - **Runtime:** Python 3
   - **Build:** `pip install -r requirements.txt`
   - **Start:** `uvicorn app:app --host 0.0.0.0 --port $PORT`
   - **Plan:** Free
4. Deploy. No environment variables needed at all.

Check it:

```bash
curl -X POST https://<your-service>.onrender.com/scan \
  -H 'content-type: application/json' \
  -d '{"skill":"---\nname: x\n---\n# x\nNo author, no version, no changelog."}'
# -> {"categories":["unclear_provenance"]}
```

There is a `Dockerfile` and a `render.yaml` here too.

> **Free tier sleeps after ~15 min.** The grader *times out slow requests and
> counts them as failed*, and a cold start is easily 30 s+. **Open your URL
> once, right before you press Check.** This alone has cost people marks.

### Submit

Paste **the full URL of the POST endpoint**, e.g.

```
https://<your-service>.onrender.com/scan
```

The app answers on `/`, `/scan`, `/q4/scan`, `/audit`, `/api/scan` and
`/skill-scan`, so whichever of those you paste will work. Then **Check**, and
when the score looks right, **Save** — Check alone records nothing.

---

## 5. How the detectors decide

Everything lives in `scanner.py`, one function per category, all conservative
by design.

- **`detect_hardcoded_secret`** — known credential shapes (`sk-…`, `ghp_…`,
  `AKIA…`, `AIza…`, JWTs, PEM blocks), webhook-shaped URLs carrying an opaque
  delivery segment, and any high-entropy literal assigned to a secret-ish key.
  Placeholders (`${VAR}`, `os.environ[...]`, `your-key-here`, `<REDACTED>`,
  `changeme`) are never findings.
- **`detect_prompt_injection`** — control override needs a stop/cancel word, a
  defiance verb and a reference to the user **in the same sentence**, so a
  benign *"if it fails, stop"* stays clean. Exfiltration needs an outbound verb
  on a **data** object, an off-machine destination, and concealment.
- **`detect_excessive_permissions`** — reads permission-ish frontmatter keys
  and prose, and asks whether the value is *scoped* (`limited to`, `only`,
  `none`, `read-only`) or *broad* (`entire`, `unrestricted`, `any host`,
  `all domains`, bare `/`, `~`, `*`).
- **`detect_unclear_provenance`** — author/version/changelog presence, from
  frontmatter **or** body headings; plus the self-rewrite rule above. Values
  like `unknown`, `n/a`, `tbd` count as absent.

## 6. Layout

```
app.py         endpoint: six POST paths, never raises, always {"categories": [...]}
scanner.py     the four detectors
selftest.py    15-file corpus + the grader's F-beta(0.5), 43 assertions
Dockerfile     container build
render.yaml    Render blueprint
```

## 7. Please read

Your five graded files are **personalised to your email**, so this code earns
nobody anything on its own — you have to deploy it yourself. Use your own host.
Don't paste someone else's URL into your answer box.

Add your own tricky files to `CORPUS` in `selftest.py` — that is the fastest way
to find out whether a change helps or hurts before you spend a Check on it.

MIT licensed. Open an issue if something breaks.
