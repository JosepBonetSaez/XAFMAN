# 🏃 XAFMAN

**Conversational assistant for the Servei d'Activitat Física (SAF) at the Universitat Autònoma de Barcelona (UAB)**

Built for **UAB The Hack! 2026** · 7–8 November 2026 · Escola d'Enginyeria UAB

![Status](https://img.shields.io/badge/status-hackathon%20prototype-blue)
![Python](https://img.shields.io/badge/python-3.11%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)
[![XAFMAN demo](https://img.youtube.com/vi/bL_MfDehktQ/hqdefault.jpg)](https://youtu.be/bL_MfDehktQ)

---

## ⚡ TL;DR

**XAFMAN** answers natural-language questions (in Catalan) about the SAF: opening hours, capacity, real-time crowding, low-traffic slots, guided activities. Every number comes from the operational dataset (April 1 – October 31, 2026). The LLM never computes — it only decides *which of 10 deterministic tools to call* and optionally *how to phrase the answer*. A **number guard** rejects any rephrasing that introduces a digit not present in the tool's output.

If XAFMAN doesn't know, it says **"No ho sé"** (I don't know) and explains why. It never guesses.

---

## 🎯 Hackathon Context

XAFMAN was built for **Line 1, Question 3** of [UAB The Hack! 2026](https://uabthehack.cat), a 24-hour hackathon held on **7–8 November 2026** at the Escola d'Enginyeria UAB, celebrating the **40th anniversary** of the Servei d'Activitat Física.

**Scope of the challenge:**
> Build a conversational assistant that answers user queries about schedules, facility capacities, real-time crowding, and alternative low-traffic slots based strictly on the real operational data of the SAF (April 1 – October 31, 2026). No hallucinations permitted.

**What made this hard:**
- 194,567 reservation rows + 48,591 guided-activity registrations + 41 spaces — easy to overfit, easy to hallucinate.
- The dataset contains **five deliberate traps** (see [Domain Traps](#-domain-traps)) that a naive system will misreport.
- The judging criteria prioritised **determinism**, **traceability**, and **graceful failure** over flashy UX.

### A note on how XAFMAN was built

**XAFMAN was coded in less than a day** (roughly 18 hours from first commit to demo), with heavy use of AI assistance for boilerplate generation, refactoring, and debugging. The **architecture**, the **five domain traps**, the **anti-hallucination strategy**, and the **guard mechanism** are human decisions informed by the dataset; the AI accelerated implementation, not design.

The decision to use AI as a pair-programmer rather than as an end-user product was deliberate: in a hackathon, the fastest path to a *credible* prototype is to spend your human hours on the parts the judges will actually scrutinise — the architecture diagram, the traceability panel, and the "No ho sé" behaviour — and let AI generate the 60% of code that is mechanical (pandas filters, Streamlit layout, dataclass definitions, JSON-schema conversion).

---

## 🧠 Thought Process

Before writing a line of code, we asked ourselves five questions. Every design choice in XAFMAN traces back to one of them.

### Q1. What does the user *actually* want to ask?

We read the challenge statement and the sample data, then wrote down ~40 realistic questions a UAB student would ask:

- *"Quina ocupació té el fitness els dimecres a les 18:00?"*
- *"A quina hora hi ha menys gent al fitness?"*
- *"El 12 de maig, quines franges estan lliures al tennis?"*
- *"Compara les reserves del fitness a l'agost i a l'octubre"*
- *"Quines classes de zumba hi ha el 12 de maig?"* ← **trap**

These 40 questions became the specification. Every tool we wrote exists because at least three of those 40 needed it. Every tool we *didn't* write (there were ten more in early drafts) was cut because fewer than two questions needed it.

### Q2. What are the failure modes we must eliminate?

We identified four:

1. **Fabricated numbers.** The most common LLM failure: inventing a plausible-looking capacity, a peak hour, a percentage. Fatal for this challenge.
2. **Confidently wrong tool calls.** The LLM picks `comparar_mesos` when the user wanted `perfil_ocupacio`. Produces a technically correct answer to the wrong question.
3. **Wrong units.** Reporting "87 people" when the metric is *pista*. Reporting "10:00 is empty" when it's a shift change.
4. **Silent failure on out-of-scope queries.** The LLM produces a plausible answer for a date outside 2026 or for a space that doesn't exist.

Every mechanism in XAFMAN exists to kill one of these four.

### Q3. How do we eliminate fabricated numbers?

**Answer: the LLM never touches the data.**

- All ten tools are pure pandas functions. They receive a `SafData` object and typed arguments, and return a `Resultat` with a text template, source citations, the raw scalar values, and a `DataFrame`.
- The LLM's role is reduced to two calls: *(a)* pick which tool to run given the question, *(b)* optionally rephrase the tool's output for readability.
- If the LLM rephrases, a **number guard** (`Motor._guard`) extracts *every* number from the rewritten text, compares it to the set of numbers the tool produced, and rejects the rewrite if any digit is new.

The guard is the single most important mechanism in XAFMAN. It converts *"we hope the LLM is honest"* into *"the LLM cannot lie without being caught."*

### Q4. What happens when the LLM fails?

Three layers of graceful degradation:

1. **Primary path:** ornith (the UAB DCC's 35B MoE model) does native function calling. Returns `{tool, args}` validated against JSON-Schema.
2. **Fallback path:** a deterministic regex planner (`planifica_regles()`) covers ~100% of the demo questions. Activated when the LLM is unavailable *or* returns invalid args.
3. **Final refusal:** if both fail, XAFMAN replies *"No ho sé"* with the reason.

This means: **XAFMAN works without an API key.** The LLM is a comfort layer, not a correctness requirement.

### Q5. How do we prove none of this to the judge?

**Traceability, exposed in the UI.** Every response opens an expander showing:

- Which tool ran
- With which arguments (raw JSON)
- Which planning path was used (`llm` or `regles`)
- Whether the reformulation was accepted, rejected, or not attempted
- The cited sources (table name + row count + space code)
- The raw table behind the answer

The judges can audit any response in five seconds. That's the demo.

---

## 🏗️ Architecture

### Layered view

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     USER INTERFACE (Streamlit / CLI)                     │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │ natural language (Catalan)
                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                            MOTOR (orchestrator)                          │
│                     respon() → 4-step pipeline                          │
└─────────────┬────────────────────────────────────┬───────────────────────┘
              │                                    │
              ▼                                    ▼
┌────────────────────────────┐      ┌──────────────────────────────────────┐
│      PLANIFICADOR          │      │      REFORMULACIÓ (optional)         │
│                            │      │                                      │
│  1. LLM function calling   │      │  LLM rephrases the tool's text       │
│     (ornith, native tools) │      │  + GUARD rejects new numbers         │
│  2. Fallback: regex rules  │      │                                      │
│  3. If both fail → None    │      │                                      │
└─────────────┬──────────────┘      └──────────────────────────────────────┘
              │  (tool_name, args) validated
              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    DETERMINISTIC LAYER (tools.py)                        │
│  ── 10 tools, 0 LLM, 0 hallucination surface ──                          │
│  pandas + numpy. Each returns Resultat(text, fonts, dades, taula).       │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                        DATA LAYER (data.py)                              │
│  Loads saf.xlsx, aligns column names, computes derived columns           │
│  (_unitats, _bloc, _fora_horari, _es_pista), indexes spaces by alias.    │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │
                                  ▼
                        ┌──────────────────┐
                        │   data/saf.xlsx  │
                        │   (7 tables)     │
                        └──────────────────┘
```

### Request lifecycle — worked example

**Query:** *"A quina hora obre el SAF els diumenges?"*

```
t=0ms     User submits question
t=50ms    Motor.respon() starts
t=60ms    planifica() → llm.chat_tools(...)
t=800ms   ornith returns tool_calls = [{name: "horari_obertura", args: {"dia": "dg"}}]
t=820ms   valida_args() → ✓ (dia ∈ enum ["dl","dt","dc","dj","dv","ds","dg"])
t=830ms   executa("horari_obertura", {"dia": "dg"}, data)
          → t_horari_obertura filters C.HORARI, returns Resultat with text
             "Horari d'obertura del SAF: diumenge: 09:00–14:00."
t=840ms   reformula() → second ornith call (enable_thinking=False)
t=1500ms  ornith returns "El SAF obre els diumenges de 09:00 a 14:00."
t=1510ms  _guard() → extract {9, 0, 14} from rewritten text
                       vs allowed {9, 0, 14, 1} from Resultat.xifres()
                       → new = ∅ → accept rewrite
t=1520ms  Resposta returned (reformulat=True, via="llm")
t=1600ms  Streamlit renders bubble + traceability expander
```

**Total: ~1.6s**, dominated by the two ornith calls. Without LLM, the same pipeline runs in **<10ms**.

---

## 📂 File-by-file breakdown

```
.
├── README.md                     ← this file
├── requirements.txt              ← dependencies
├── .env.example                  ← environment template
├── .gitignore                    ← excludes .env and data/*.xlsx
├── app.py                        ← Streamlit UI
├── saf/                          ← the XAFMAN package
│   ├── __init__.py
│   ├── config.py                 ← all domain constants
│   ├── utils.py                  ← text normalization, Catalan formatting
│   ├── data.py                   ← load + clean saf.xlsx into SafData
│   ├── tools.py                  ← 10 deterministic tools + registry
│   ├── llm.py                    ← ornith client (OpenAI-compatible)
│   ├── router.py                 ← planner: LLM function calling + regex fallback
│   ├── motor.py                  ← orchestrator + number guard
│   └── cli.py                    ← command-line entrypoint
└── scripts/
    └── make_sample_data.py       ← generates synthetic saf.xlsx with all 5 traps
```

### `app.py` — Streamlit interface (~140 lines)

The chat UI for XAFMAN. Key responsibilities:

- **Sidebar:** model status, dataset stats, reference date picker, 8 example questions (one click = one query), conversation reset.
- **Chat history:** persisted in `st.session_state.missatges`.
- **Traceability expander:** rendered for every assistant message, showing `tool`, `via`, `reformulat`, `rebutjat`, raw JSON args, cited fonts, and the raw DataFrame.
- **Caching:** `@st.cache_resource` on `_dades()` and `_motor()` so the dataset and LLM client load once.

Design choice: **example questions are buttons, not free-text prompts.** In a 3-minute demo, clicking is faster and safer than typing. Three of the eight examples are designed to exercise the three demo points (basic tool, LLM reformulation, domain trap).

### `saf/config.py` — Constants (~70 lines)

Every magic number lives here. Nothing numeric in the tools.

```python
DATA_INICI = date(2026, 4, 1)
DATA_FI    = date(2026, 10, 31)

HORARI = {
    "dl": (time(7, 0), time(21, 0)),
    ...
    "dg": (time(9, 0), time(14, 0)),
}

FORATS_FITNESS      = {"10:00", "15:00"}   # trap 1: shift changes, not low demand
DURADA_TORN_FITNESS = 75                    # fitness shifts are 75 min
LLINDAR_BLOC_MINUTS = 240                   # trap 3: blocks >4h are club/school
PREFIX_FITNESS      = "FITNESS"

ALIASOS = {                                 # "gimnàs", "musculació" → FITNESS
    "fitness": ["FITNESS"],
    "piscina": ["PISCINA"],
    ...
}
```

Also holds the LLM configuration:

```python
UTH_KEY      = os.getenv("UTH_KEY", "")
UTH_BASE_URL = os.getenv("UTH_BASE_URL", "https://dcc-llm-gw.uab.cat/uabthehack/v1")
UTH_MODEL    = os.getenv("UTH_MODEL", "ornith")
```

### `saf/utils.py` — Pure utilities (~110 lines)

No dependency on data or domain. Three groups:

- **Text normalization:** `normalitza()` strips accents, lowercases, removes punctuation. Everything user-facing is normalized before regex matching.
- **Catalan formatting:** `fmt_num(1234.5)` → `"1.234,5"`, `fmt_pct()` → `"71,9 %"`, `fmt_valor()` preserves integers without forcing decimals.
- **Parsing:** `hora_en_minuts("18:30")` → `1110`, `xifres_en_text()` extracts all numbers from a string (handles Catalan and English decimal/thousands conventions), `afegeix_enters()` extends a set with integer truncations (so `12.4` in the text is allowed if `12` is in the allowed set).

The number extraction is deliberately conservative: `"87,3"` and `"87"` are both extracted from `"mitjana 87,3 persones"`, so a reformulation that says `"al voltant de 87"` is accepted, but one that says `"gairebé 100"` is rejected.

### `saf/data.py` — Loading and cleaning (~250 lines)

Loads `saf.xlsx` into a `SafData` dataclass with 7 DataFrames and a space index.

**Column alignment:** `_align()` normalizes incoming column names and matches them against canonical names. If the real Excel has `"Data d'us de la reserva"` (no accent) it still matches `"Data d'ús de la reserva"`. Missing columns are filled with NaN.

**Derived columns** (computed once at load time, cached):

| Column | Formula | Purpose |
|---|---|---|
| `_data` | `pd.to_datetime(...)` | Normalized date |
| `_inici`, `_fi` | minutes since midnight | Fast interval math |
| `_unitats` | `0.5` if `pista AND mitja_pista`, else `1.0` | **Trap 2** |
| `_bloc` | `flag OR Durada > 240` | **Trap 3** |
| `_fora_horari` | flag | Out-of-hours |
| `_es_pista` | metric contains "pista" | **Trap 2** |
| `_dirigida` | flag | Guided activity |

**Space resolution — `SafData.resoldre(text)`:** turns `"piscina"` into a list of space codes, using a 3-tier cascade:

1. **Exact match** against the alias index (built from names, subspaces, and `ALIASOS`).
2. **Substring match** — the longest alias contained in the text wins; nested aliases are absorbed (`"sala 1"` doesn't match when the text is `"sala 12"`).
3. **Fuzzy match** via `difflib.get_close_matches(cutoff=0.84)`.

Returns `(codis, estat)` where `estat ∈ {"ok", "multiple", "unknown"}`. Tools use this to decide whether to answer or refuse.

### `saf/tools.py` — The deterministic layer (~500 lines)

**Ten pure functions.** Each returns a `Resultat`:

```python
@dataclass
class Resultat:
    tool:   str                     # which tool
    args:   dict                    # with which args
    ok:     bool                    # False → "No ho sé"
    text:   str                     # deterministic Catalan template
    fonts:  list[str]               # traceability: table + row count + space code
    dades:  dict                    # scalars the guard will allow
    taula:  pd.DataFrame | None     # raw table for the UI expander

    def xifres(self) -> set[float]:
        """Exhaustive set of numbers appearing in text + dades."""
```

`Resultat.xifres()` is the linchpin of the anti-hallucination design. It collects every number that appears in `text` (via `xifres_en_text()`) *plus* every numeric value in `dades`. This becomes the **allowed set** for the reformulation guard downstream.

| Tool | Purpose | Trap covered |
|---|---|---|
| `horari_obertura` | Opening hours per weekday | — |
| `info_espai` | Capacity, metric, max simultaneous observed | — |
| `ocupacio_instant` | Occupancy at a specific date+time | Fitness 10:00/15:00 warning |
| `ocupacio_tipica` | Mean/median/min/max across all matching weekdays | Fitness 10:00/15:00 warning |
| `perfil_ocupacio` | Average occupancy per hour slot over a period | Fitness turns annotated |
| `franges_baixes` | Ranked low-traffic slots | **Excludes 10:00/15:00** for fitness |
| `disponibilitat` | Free slots on a date | — |
| `comptatge_reserves` | Total + blocks + out-of-hours + guided split | Blocks >4h separated |
| `comparar_mesos` | Volume ratio between two months | **Warns about August slump** |
| `activitats_dirigides` | Aggregated enrollment per activity | Returns `_no_ho_se` if user asks per-day |

**Registry:** the `TOOLS` dict at the bottom maps `name → (function, schema)`. This same dict is used by:

- The validator (`valida_args()`)
- The OpenAI tools converter (`tools_openai()`)
- The executor (`executa()`)

Adding a new tool means adding **one entry** and everything else picks it up.

### `saf/llm.py` — LLM transport (~120 lines)

A single class `DCCLLM(BaseLLM)` wraps the OpenAI SDK pointed at the UAB DCC endpoint.

```python
class DCCLLM(BaseLLM):
    name = "ornith (DCC UAB)"

    def available(self) -> bool:
        return bool(self.key)   # UTH_KEY present?

    def chat_tools(self, messages, tools, disable_thinking=True):
        """Native function calling. disable_thinking speeds up planning."""
        ...

    def complete(self, prompt, max_tokens=2048, json_mode=False, disable_thinking=False):
        """Free-text generation for reformulation."""
        ...
```

Key detail: `disable_thinking=True` sets `extra_body={"chat_template_kwargs": {"enable_thinking": False}}`. Per the DCC model card, this disables the reasoning scratchpad, which is **essential for reformulation** (reasoning tokens would waste the small budget we allow).

`NullLLM` is returned by `get_llm()` when `UTH_KEY` is missing or `SAF_LLM=none`. In that case, `llm.available()` is `False` everywhere, and the router + motor skip the LLM paths entirely. **XAFMAN still works.**

`BaseLLM.extreu_json()` is a balanced-brace extractor that pulls the first complete JSON object from a string. Used only in a legacy path — with native function calling, we don't parse JSON from text anymore.

### `saf/router.py` — The planner (~300 lines)

Two planners, same signature.

**Path A — LLM native function calling:**

```python
resp = llm.chat_tools(
    messages=[{"role": "system", "content": _SYSTEM_PROMPT},
              {"role": "user",   "content": pregunta}],
    tools=tools_openai(),
    disable_thinking=True,
)
tool_calls = resp.choices[0].message.tool_calls
name  = tool_calls[0].function.name
args  = json.loads(tool_calls[0].function.arguments)
ok, _ = valida_args(name, args)
if ok:
    return name, args, "llm"
```

The system prompt is short and explicit:

> *"Ets el planificador d'eines de l'assistent «XAFMAN». Crida exactament una eina amb arguments vàlids. Dates en format YYYY-MM-DD dins 2026-04-01..2026-10-31. Hores en HH:MM. Dies: dl, dt, dc, dj, dv, ds, dg. Si la pregunta no es pot respondre amb cap eina, no cridis cap eina."*

**Path B — Deterministic regex planner (`planifica_regles()`):**

A cascade of `re.search()` patterns. Order matters:

1. `horari_obertura` — if `horari|quan obr|quina hora obr|...` and no space mentioned
2. `activitats_dirigides` — checked **before** reserves (they share vocabulary), and returns `_no_ho_se` if a specific date is present (**trap 6**)
3. `comparar_mesos` — if two month names + space
4. `franges_baixes` — high priority (`menys gent|buida|tranquil|...`)
5. `disponibilitat`, `ocupacio_tipica`, `ocupacio_instant`, `perfil_ocupacio`, `comptatge_reserves`, `info_espai`

Helper functions shared with Path A: `_periode_de()`, `_dia_de()`, `_hora_de()`, `_espai_de()`. This ensures date/hour parsing is consistent across paths.

**Reformulation (`reformula()`):**

```python
_PROMPT_REFORMULA = """
Reescriu la resposta de manera breu (2-4 frases), en català, to amable i directe.

Regles ABSOLUTES:
- NO afegeixis cap número que no aparegui al text original.
- NO canviïs cap xifra, cap codi d'espai ni cap data.
- NO inventis informació. Si el text diu "No ho sé", mantén exactament aquesta idea.

Text original:
{text}
"""
```

The prompt says "no new numbers" — but we don't *trust* the prompt. That's what the guard is for.

### `saf/motor.py` — The orchestrator (~80 lines)

`Motor.respon()` is the only public API. Four sequential steps:

```python
def respon(self, pregunta: str) -> Resposta:
    # 1. Plan
    tool, args, via = planifica(pregunta, self.d, self.llm, self.avui)
    if tool is None:
        return Resposta("No ho sé. ...", tool=None, via="cap", ok=False)

    # 2. Execute (deterministic)
    res = executa(tool, args, self.d)
    if not res.ok:
        return Resposta(res.text, tool, args, via, res.fonts, res.taula, ok=False)

    # 3. Reformulate + guard
    base = res.text
    reformulat, rebutjat = False, False
    if self.llm.available():
        cand = reformula(base, self.llm)
        if cand and cand != base and self._guard(cand, res.xifres()):
            base = cand; reformulat = True
        elif cand and cand != base:
            rebutjat = True

    # 4. Return
    return Resposta(base, tool, args, via, res.fonts, res.taula,
                    reformulat=reformulat, rebutjat=rebutjat, ok=True)
```

**The guard — the single most important method in XAFMAN:**

```python
@staticmethod
def _guard(text_llm: str, permesos: set[float]) -> bool:
    permesos = afegeix_enters(permesos)           # allow int truncation (12.4 → 12)
    noves = xifres_en_text(text_llm) - permesos   # numbers in rewrite not in source
    return not noves                               # False if any new number
```

**Concrete example of the guard rejecting:**

| Stage | Text | Numbers |
|---|---|---|
| Tool output (allowed set) | `"…té 87 persones (71,9 % de 121)…"` | `{87, 71.9, 121}` |
| LLM rewrite | `"…gairebé 100 persones…"` | `{100}` |
| Guard verdict | `100 ∉ {87, 71.9, 121, 87, 71, 121}` | **REJECTED** → show original template |

**Concrete example of the guard accepting:**

| Stage | Text | Numbers |
|---|---|---|
| Tool output | `"…de 09:00 a 14:00…"` | `{9, 0, 14}` |
| LLM rewrite | `"…de 9 a 14h…"` | `{9, 14}` |
| Guard verdict | `{9, 14} ⊆ {9, 0, 14}` | **ACCEPTED** |

Note that `"2 de la tarda"` (a semantically correct but numerically new rewrite of `"14:00"`) is **rejected**. This is deliberate: we trade a small amount of fluency for a hard guarantee that no number in the user-facing text is fabricated.

### `saf/cli.py` — Command-line entrypoint (~25 lines)

```bash
python -m saf.cli "A quina hora obre el SAF els diumenges?"
```

Prints the answer, the trace line (`tool / via / reformulat / rebutjat`), the cited fonts, and the first 12 rows of the table. Useful for automated testing and for the judge who wants to verify without the UI.

### `scripts/make_sample_data.py` — Synthetic data generator (~250 lines)

Generates a synthetic `data/saf.xlsx` with all seven sheets and all five traps reproduced:

- **Fitness:** 11 shifts of 75 minutes, with no reservations at 10:00 and 15:00 (the trap).
- **Padel:** 15% of reservations have `mitja pista = "Sí"`, exercising the 0.5-unit path.
- **Club/school blocks:** 8% of weekend padel/tennis reservations are 4.5–6 hours long.
- **August slump:** month factor of 0.35 for August vs 1.15 for October.
- **Guided activities:** 4% overbooking rate, matching the real dataset.

The generator exists so XAFMAN is **self-contained and runnable without the original Excel.** If the real `saf.xlsx` is available, drop it into `data/` and XAFMAN uses it directly (the `_align` function in `data.py` tolerates column name variations).

---

## 🪤 Domain Traps

The dataset contains five realities that a naive system will misreport. XAFMAN handles each one explicitly.

### Trap 1 — Fitness shift changes are not low demand

The fitness room operates in 75-minute shifts starting at 07, 08, 09, 11, 12, 13, 14, 16, 17, 18, and 19. Hours **10:00 and 15:00 have near-zero reservations** because they are between shifts.

**Handled by:**

- `config.FORATS_FITNESS = {"10:00", "15:00"}`
- `t_franges_baixes` **excludes** these slots from the ranking
- `t_ocupacio_instant` and `t_ocupacio_tipica` **warn** in the text: *"és un forat de canvi de torn (10:00 i 15:00), no és baixa demanda"*
- `t_perfil_ocupacio` reports the mean occupancy in these slots but annotates it

### Trap 2 — People vs. court

Fitness and pool are counted in **people** (peak fitness slot: 121). Padel, tennis, squash are counted in **courts**. A half-court reservation counts as **0.5 units**.

**Handled by:**

- `data._unitats` = 0.5 when `_es_pista AND _mitja_pista`, else 1.0
- `SafData.metrica()` returns `"persones"` or `"pista"` per space set
- Every tool's text uses the correct noun: `"persones"`, `"pistes"`, `"pistes ocupades"`
- `t_info_espai` reports the metric explicitly

### Trap 3 — Club/school blocks >4h

1,371 reservations last more than 4 hours — these are institutional bookings (clubs, schools). Counting them as individual reservations inflates the per-user count by a factor of 2×.

**Handled by:**

- `data._bloc = flag OR (Durada > 240)`
- `t_comptatge_reserves` splits the count: *"X reserves, of which Y are club/school blocks, Z are out of hours, W are guided activities. Individual reservations (excluding blocks): N."*
- `t_ocupacio_instant` warns if a block is active at the queried instant

### Trap 4 — August slump

August reservation volume drops to roughly **one-third** of October. Any month-over-month comparison that doesn't account for this is misleading.

**Handled by:**

- `t_comparar_mesos` always appends: *"Atenció a l'efecte estacional: l'agost cau aproximadament a un terç del volum d'octubre, així que la comparació no reflecteix demanda real."*
- Triggered automatically whenever month 8 is one of the two compared.

### Trap 5 — Guided activities have no class date

The `Activitats dirigides` table has a `Data d'inscripció` (registration timestamp) but **no class date**. A naive system will filter on `Data d'inscripció` and answer as if it were the class date, producing a completely wrong answer.

**Handled by:**

- The router detects *"quines classes hi ha el [date]"* patterns and returns `_no_ho_se` with an explanation:
  > *"No ho sé. La taula «Activitats dirigides» no conté la data en què es fa cada classe (només la data d'inscripció), així que no puc respondre quines classes concretes hi ha un dia determinat. Sí que et puc donar agregats per activitat, sala o hora."*
- `t_activitats_dirigides` only supports filtering by `nom`, `espai`, `tipus`, `hora`, and `nomes_amb_places` — never by date.

This is the trap that most distinguishes XAFMAN: **a system that says "No ho sé" is more trustworthy than one that doesn't.**

---

## 🚀 Quick Start

### 1. Environment

```bash
python -m venv .venv
source .venv/bin/activate           # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Data

```bash
# Option A: generate synthetic data with all five traps
python scripts/make_sample_data.py

# Option B: use the real saf.xlsx
cp /path/to/saf.xlsx data/saf.xlsx
# (optional) export SAF_XLSX=/path/to/saf.xlsx
```

### 3. LLM key (optional but recommended)

```bash
export UTH_KEY=uth-...              # your team's key from the UAB platform
```

Without this, XAFMAN runs in **rules mode** — same answers for the demo questions, no reformulation.

### 4. Run

```bash
streamlit run app.py
```

Open http://localhost:8501.

### 5. CLI

```bash
python -m saf.cli "A quina hora obre el SAF els diumenges?"
```

---

## 🎬 Demo — 90 seconds

The Streamlit UI has 8 example questions in the sidebar. Each is one click.

| # | Question | Demonstrates |
|---|---|---|
| 1 | *Quin horari té el SAF el diumenge?* | Basic tool, deterministic response, traceability table |
| 2 | *Quina ocupació té el fitness els dimecres a les 18:00?* | LLM reformulation + guard acceptance (or rejection if the LLM tries to add a number) |
| 3 | *A quina hora hi ha menys gent al fitness?* | **Trap 1**: 10:00 and 15:00 are excluded from the ranking |
| 4 | *El 12 de maig, quines franges estan lliures al tennis?* | Interval coverage + hour-slot table |
| 5 | *Quantes reserves hi ha hagut a la piscina l'octubre?* | **Trap 3**: blocks counted separately |
| 6 | *Compara les reserves del fitness a l'agost i a l'octubre* | **Trap 4**: seasonal warning |
| 7 | *Quines activitats dirigides tenen menys ocupació?* | Aggregation, overbooking count |
| 8 | *Quines classes de zumba hi ha el 12 de maig?* | **Trap 5**: explicit "No ho sé" |

**Recommended live flow (3 min):**

1. Click question 1. Show the traceability expander. *"Here's the tool, the args, the cited source, the raw table. Audit anything in five seconds."*
2. Click question 2. *"This time the LLM rephrased the answer. But the guard checked every number first."* Point at `Reformulació: sí`.
3. Click question 8. *"This is what a naive chatbot would hallucinate. XAFMAN says 'No ho sé' and explains why."*

**Demo safety:**

- Pre-start Streamlit before going on stage. Don't close it between rehearsals.
- Don't type — click the example buttons.
- If the LLM is down, XAFMAN falls back to rules; the demo still works.
- Screenshots of all three responses in `demo/` as a last resort.

---

## 🧪 Testing

XAFMAN is designed to be testable:

```bash
# Deterministic tools can be tested without LLM
python -m saf.cli "A quina hora obre el SAF els diumenges?"
python -m saf.cli "Quina ocupació té el fitness els dimecres a les 18:00?"
python -m saf.cli "Quines classes de zumba hi ha el 12 de maig?"

# Force rules mode
SAF_LLM=none python -m saf.cli "Quina ocupació té el fitness els dimecres a les 18:00?"
```

Each invocation prints the trace line so you can verify `via=llm` or `via=regles`.

**The guard can be tested in isolation:**

```python
from saf.motor import Motor
allowed = {87.0, 71.9, 121.0}
assert Motor._guard("Hi ha 87 persones (71,9 % de 121)", allowed) is True
assert Motor._guard("Hi ha gairebé 100 persones", allowed) is False
assert Motor._guard("Hi ha 87 persones", allowed) is True  # subset is fine
```

---

## 💡 Design Decisions and Trade-offs

| Decision | Alternative | Rationale |
|---|---|---|
| **Native function calling** over prompt-parsed JSON | Reference implementations use text prompts | Ornith supports it natively; robust, no parse errors, schema enforcement built in |
| **Regex fallback planner** | Rely 100% on the LLM | Resilience; ~60 lines cover all demo questions; demo cannot fail from API issues |
| **10 narrow tools** over one mega-tool | Could have `query(dataset, filters)` | Narrow tools are testable, validatable, and force the LLM to pick one behaviour at a time |
| **`Resultat` with `fonts` + `taula`** | Return only text | Traceability is a judging criterion; the expander is the money-shot of the demo |
| **Guard with integer truncation** | Exact match only | Allows `12.4 → 12` (natural rounding) while still blocking fabricated digits |
| **No RAG / vector store** | Add embeddings | The domain is structured, not textual. Retrieval adds latency and a hallucination surface for zero benefit |
| **No agentic loop** | Multi-tool chaining | One tool call per query. Multi-step increases latency and breaks the clean traceability story |
| **Postgres / SQLite not used** | A real DB | 194k rows fit in memory; pandas is simpler and `resoldre()` handles alias resolution |
| **Streamlit over FastAPI + React** | Full-stack web app | Hackathon constraint: one file deploys the whole UI |

---

## ⚠️ Known Limitations

Being honest about what doesn't work:

1. **The rule planner misses `"Diumenge el SAF obre?"`** — the pattern `quina hora obr` catches `"A quina hora obre el SAF"` but not the bare `"obre"`. If `UTH_KEY` is missing and the user asks exactly that, they get "No ho sé". Fix: add `|obre|obrir|obren|obreu` to the pattern in `router.py`. The LLM path handles it correctly.
2. **`t_horari_obertura` prints two consecutive `:`** (`"…SAF: diumenge: 09:00–14:00"`). Cosmetic. Fix: change the template to `"{dia}: de {obretura} a {tancament}"`.
3. **The font label `"Horari (7 files)"` is imprecise** when filtering by a single day — should be `"Horari (1 fila)"`. The number is correct (it's the total table row count), but the label is misleading for single-day queries.
4. **The guard rejects `"14:00 → 2 de la tarda"`** — a semantically correct paraphrase. We accept this loss of fluency in exchange for the hard guarantee. Extending `afegeix_enters` with equivalence tables (14 ↔ 2 pm) would weaken the guarantee; we chose not to.
5. **The synthetic data generator** produces ~50k rows, not the full 194k. It reproduces all five traps but not the full distribution. For the real demo, use the real `saf.xlsx`.
6. **No streaming.** Responses are 1.5s with LLM, 10ms without. Streaming would complicate the guard (we'd have to buffer to extract all numbers anyway).

---

## 🛠️ Configuration

| Environment variable | Default | Description |
|---|---|---|
| `UTH_KEY` | *(none)* | Team's API key for the UAB DCC. **Required for LLM features.** |
| `UTH_BASE_URL` | `https://dcc-llm-gw.uab.cat/uabthehack/v1` | Ornith endpoint |
| `UTH_MODEL` | `ornith` | Model name |
| `SAF_LLM` | `auto` | `auto` / `none` / `regles` |
| `SAF_XLSX` | `./data/saf.xlsx` | Path to the Excel dataset |

Copy `.env.example` to `.env` and fill in `UTH_KEY`, or export the variables in your shell.

---

## 🤝 How AI was used

Transparency for the judges:

- **Human decisions:** architecture (4-layer pipeline), anti-hallucination strategy (guard + `Resultat.xifres()`), the 5 domain traps, tool selection and naming, the reformulation prompt, the demo flow.
- **AI-accelerated:** pandas filter code (~80% of `tools.py`), Streamlit layout, the JSON-schema converter (`tools_openai()`), the synthetic data generator, and this README's first draft.
- **Human review:** every AI-generated function was read, tested, and often rewritten. The guard was written by hand after an LLM-generated version failed the "100 persones" test case.

**Total build time:** ~18 hours across 7–8 November 2026.

---

## 📜 License

MIT — see `LICENSE` (to be added).

---

## 🙏 Acknowledgements

- **UAB The Hack! 2026** organisers and the Escola d'Enginyeria UAB
- **Departament de Ciències de la Computació (DCC)** for providing the ornith model on 8 GPUs
- **Servei d'Activitat Física (SAF)** for the 40-year-old operational dataset that made XAFMAN possible

---

*Built in Barcelona, 7–8 November 2026.*

