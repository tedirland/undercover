# Week 1 Day 2 Review Guide: Models vs Agents on Insurance Claims

Source: `labs/week1day2.ipynb`. Cells are referred to by what they do, not by number.

## How to use this guide

1. **Pass 1 (5 min):** read "The map" and say it out loud without looking.
2. **Pass 2 (15 min):** for each concept, read the analogy, then cover the answer under "Recall" and try to produce it.
3. **Pass 3 (as needed):** read "Where the analogy breaks". The break points are where real understanding sits. An analogy you cannot break is one you don't understand yet.

Analogies here were checked three ways (see "Analogy audit" near the end): does it map piece by piece, where does it fail, and does it predict something true about the real system.

---

## The map

The lab asks one question: **for the same 25 insurance claims, which approach is most accurate, cheapest and fastest?**

```
 25 claims + policy wording + handlers' real decisions (the answer key)
                          |
     +--------------------+----------------------+
     |                    |                      |
 PART 2: one LLM call   PART 3: Jev          PART 4: Agent
 (Structured Outputs)   (decision model,     (LLM + tools in a loop)
 vary model + effort     no text output)
     |                    |                      |
     +--------------------+----------------------+
                          |
        score each: right / wrong, cost in $, seconds
                          |
          one comparison table (accuracy, cost, speed)
```

Three dials to compare everything on: **accuracy, cost, speed**.

---

## Concepts

### 1. Messages: system prompt vs user prompt

**Plain version.** A chat model is called with a list of `{"role", "content"}` dicts. The `system` message sets the job and the rules. The `user` message is the request. The model is **stateless**: it remembers nothing between calls.

**Analogy: the temp worker with a rulebook.** Each call brings in a temp worker who has never seen your company.
- System message = the job description plus the policy rulebook, stapled to their desk.
- User message = today's case folder.
- In the notebook: `SYSTEM_PROMPT` (instructions + whole policy wording) is the same for all 25 claims. `make_user_prompt(claim)` changes every time.

**Where it breaks.** A real temp worker remembers yesterday's cases. The model does not. Every call, you hand over the entire rulebook again, and you pay for reading it again.

**Recall.** Why is the policy wording in the system prompt and the policy record in the user prompt?

<details><summary>Answer</summary>
The wording is constant across all claims (the rulebook). The policy record is specific to each claim (the case folder), so it travels with the claim.
</details>

---

### 2. OpenRouter and the base_url swap

**Plain version.** `OpenAI(base_url="https://openrouter.ai/api/v1", ...)` reuses the OpenAI client library, but sends the calls to OpenRouter. OpenRouter forwards them to many providers' models (OpenAI, Anthropic, Google) and bills you in one place. To switch models you only change the `model` string, e.g. `"anthropic/claude-haiku-5.5"`.

**Analogy: a telephone switchboard that speaks one dialing format.**
- The OpenAI client = the way you dial (a fixed format).
- OpenRouter = the switchboard. Same dialing, many destinations.
- `model="provider/name"` = the extension you ask for.
- `extra_body={"provider": {"require_parameters": True}}` = "only connect me to an operator who can handle everything I'm asking for; otherwise refuse." It prevents silently dropping a feature, such as structured outputs, that the model doesn't support.

**Where it breaks.** A switchboard only connects. OpenRouter also adds billing and the `usage.cost` field. That field is an OpenRouter extension, which is why Pylance complains about `response.usage.cost`: the OpenAI library's types don't know about it.

**Recall.** Why can't the OpenAI client call Jev?

<details><summary>Answer</summary>
Jev is served on OpenRouter's separate Decisions endpoint, not the chat completions endpoint. The OpenAI client only knows chat completions, so Part 3 uses OpenRouter's own Python client (`from openrouter import OpenRouter`).
</details>

---

### 3. Structured Outputs with Pydantic

**Plain version.** Instead of free text, the model must reply in JSON that fits a schema. You describe the schema as a Pydantic class (`ClaimDecision`), pass it as `response_format`, and call `.parse()`. The result arrives as an object in `response.choices[0].message.parsed`.

**Analogy: a printed form instead of a letter.**
- `ClaimDecision` = the blank form.
- `Literal["approve", "decline", "refer"]` = three tick-boxes, nothing else allowed.
- `Field(description=...)` = the small hint text under a box. The model reads it.
- `payout: float` = a box that only accepts a number.

**Order matters.** `rationale` is the *first* field, before `decision`. The model writes left to right and cannot go back, so writing its reasoning first means the reasoning is available when it picks the decision. This is "show your working before the answer" on an exam.

**Where it breaks.** A person can fill a form in any order, skip around and erase. The model fills fields strictly in order, one token at a time. Also, the reasoning-first claim is the notebook author's design rationale, not something the notebook measures. See the experiments section for a way to test it.

**Recall.** What would go wrong if `payout` were typed as `str`?

<details><summary>Answer</summary>
Nothing would stop the model writing "about $200", and `is_right` does arithmetic (`abs(answer["payout"] - label["payout"])`), which would fail on a string. The type is a contract that makes downstream code safe.
</details>

---

### 4. Reasoning effort (low / medium / high)

**Plain version.** `reasoning_effort` controls how much hidden thinking the model does before answering. More effort usually costs more and takes longer, and may help on harder problems.

**Analogy: scratch paper.** Low = a few seconds with a Post-it. High = a full page of working before you answer. The exam question and the student are the same; only the time allowed to think changes.

**Where it breaks.** More scratch paper does not guarantee a better answer. A student can write more and still be wrong. And, unlike a student, you are billed by the page.

**Recall.** What do the results table's three columns let you decide?

<details><summary>Answer</summary>
Whether the extra accuracy from a higher effort or bigger model is worth the extra dollars and seconds. A customer will ask about all three. Cost per claim is easiest to relate to the business: compare it with what a handler's time on a claim costs.
</details>

---

### 5. Evaluation: the answer key, `is_right`, and difficulty

**Plain version.** Handlers already decided all 25 claims. Those decisions are the **labels**. A model's answer is "right" only if the decision matches **and** the payout is within $1 (`is_right`). Claims are tagged `routine`, `hard` or `manual`.

**Analogy: a marked exam.**
- The 25 claims = the exam paper. The models = students.
- Handlers' decisions = the examiner's answer key.
- `is_right` = a strict marking scheme: right answer and right working, both needed.
- `manual` claims = an open-book exam where the book is missing a chapter. They depend on the claims manual or the customer's history, which the model is not given. The notebook warns to expect nearly all of them wrong for now.
- `held_out=True` = the final exam the student never practiced on. It guards against tuning your prompt to the 25 visible claims (overfitting).

**Where it breaks.** The notebook includes a `refer` option for when "the wording does not settle the question". A smart student with a missing chapter could answer "refer" and be correct. Whether the manual claims' labels are `refer` is something to check in the data, not assume.

**Mnemonic: the three wrong answers.** *Pay, Pique, Pause.*
- Wrong approval: **Pay** (money the policy doesn't owe).
- Wrong decline: **Pique** (a customer let down).
- Wrong referral: **Pause** (a handler's time wasted).

These are not equally costly, but `right` counts them all the same. That is a limit of the scoring.

**Recall.** Why does the results pivot sum `right` by difficulty instead of showing one overall score?

<details><summary>Answer</summary>
A single score hides where a model fails. Splitting by difficulty shows it can handle routine claims but not manual ones, which tells you whether the failure is the model or missing information.
</details>

---

### 6. Jev: a decision model, not a text model

**Plain version.** Jev (from TypeSafe, served on OpenRouter) doesn't write text. You send it `state` (the facts) and a typed question. It returns the chosen option, a probability per option and a **confidence**. It is very fast. It cannot write a rationale or work out a payout, so it is scored on the decision only.

Its question types are Choice, Noul, Score and Box. The notebook uses Choice with criteria for approve / decline / refer.

**Analogy: a triage nurse with a gut-check dial, not a doctor writing a report.** Tier-1 support in a call center.
- Jev = Tier 1: quick sorting, no explanation.
- The confidence value = how sure Tier 1 is.
- Threshold of 0.9 in the notebook = "if Tier 1 is at least 90% sure, accept; otherwise escalate to Tier 2 (a stronger model)."

**Where it breaks.** A human Tier-1 agent usually knows when they're unsure. Jev's confidence is only useful if it is **calibrated**: when it says 0.95, it should be right far more often than when it says 0.6. The cell that groups by `confident` and counts `right` is the test of exactly this. If the confident group isn't right more often, the cascade idea is worthless.

**Gotcha.** The notebook pins `typesafe/jev-1.13`. The OpenRouter docs page lists `typesafe/jev-1.14` and a moving alias `~typesafe/jev-latest`. Pinned versions drift; check which is current before comparing results over time.

**Recall.** Why is Jev left out of the final comparison table?

<details><summary>Answer</summary>
It only produces a decision. It has no payout, so it cannot be scored by `is_right`, and it has no rationale. It isn't doing the same job as the other rows.
</details>

---

### 7. Agents: an LLM, with tools, in a loop

**Plain version.** Parts 2 and 3 are one call with everything prepacked. An agent is a loop: the model asks for a tool, the tool runs, the result goes back, and it repeats until the goal is done. LangChain's `create_agent` runs the loop. The two tools are `get_policy` (look up the record) and `record_decision` (save the decision).

**Analogy: a boss in a closed room passing notes through a slot.**
- The model = the boss, who can only read and write notes.
- LangChain = the clerk who runs errands and brings back results.
- The tools list (names, docstrings, type hints) = the menu of errands pinned on the wall. The boss reads the menu and never sees how errands are done.
- `get_policy` = "fetch me folder POL-xxx."
- `record_decision` = "write this in the ledger." The `recorded` dict is the ledger.
- One claim = three note exchanges: ask for the policy, record the decision, announce they're finished.

**Why this one, not a "handler walks to the cabinet" picture.** That version fails the check below: it suggests the model itself performs the action. The model only emits a request like `get_policy("POL-123")`. Your code executes it. Remember the slot.

**Where it breaks.** A real boss would not forget the whole job between notes. The model sees the entire message history again on each call, which is why three calls cost more than one: the policy wording is re-read each time. An agent can also finish without ever writing in the ledger, so the notebook counts "nothing recorded" as wrong.

**Recall.** In what way is the agent's input different from the single-call approach?

<details><summary>Answer</summary>
`make_claim_prompt` sends only the claim. The policy record isn't included; the agent has to fetch it with `get_policy`. Once it has, it has the same information as the single call, which is what makes the scores comparable.
</details>

---

## Code idioms worth recognizing

| Snippet | What it does |
|---|---|
| `f"{cost:.4f}"` | f-string; format a float to 4 decimals |
| `json.dumps(x, indent=2)` | dict to pretty-printed JSON text |
| `{**answer, "right": ...}` | copy a dict and add or override keys |
| `sum(r["right"] for r in results)` | `True` counts as 1, so this counts the correct answers |
| `{d["claim_id"]: d for d in decisions}` | dict comprehension; build a lookup table keyed by id |
| `df.pivot_table(index=..., columns=..., aggfunc="sum")` | cross-tab of counts |
| `df.groupby([...])[[...]].sum()` | totals per group |
| `@tool` | turns a function plus its docstring and type hints into a tool the model can request |

---

## Gotchas

- **New OpenRouter accounts** may hit "20 requests per minute". Add `time.sleep(3)` after each LLM call.
- **`luna_medium` is reused.** In the sequence of runs, `luna_medium` is first the Luna run and then reassigned to the Haiku run. The Luna medium results are overwritten, and later cells (the scored table, the final `experiments` list) use the Haiku results under that name. Give each run its own variable name.
- **Pylance errors** on `decide` (`messages`, `.parsed` being possibly `None`, `usage.cost`) come from the OpenAI types not knowing about OpenRouter's extensions. They are type-checker warnings, not runtime failures. `.parsed` being `None` is a real possibility if the model refuses.
- **Variance.** Models vary from run to run. Don't treat one run's score as fixed.

---

## Experiments to try (hypotheses, not results)

1. Swap the order of `rationale` and `decision` in `ClaimDecision`. Does accuracy change? Tests the "reason first" claim.
2. Look at the handler labels for `manual` claims. How many are `refer`? Does a model that refers more score better there?
3. Sweep the Jev confidence threshold (0.7, 0.8, 0.9, 0.95). Where does the right-rate for the confident group stop improving?
4. The assignment: with the agent, find the cheapest model that scores at least 15 of 25 consistently. "Consistently" means several runs.
5. Compare cost per claim with a handler's time cost. This is what turns the table into a business case.

---

## Analogy audit

Each analogy was checked three times: **(1)** does every part map to something real, **(2)** where does it fail, **(3)** does it predict something the notebook actually shows. Revisions are noted.

| Analogy | 1. Maps? | 2. Fails where | 3. Predicts correctly? | Verdict |
|---|---|---|---|---|
| Temp worker + rulebook | System = constant, user = per-claim: matches the code | Real workers remember; models don't | Predicts reusing one system prompt across 25 claims: yes | Kept. First draft ("restaurant house rules") didn't convey statelessness, so it was replaced. |
| Switchboard | `base_url`, `model`, `require_parameters` all map | Switchboard doesn't bill; OpenRouter does | Predicts switching model by string only: yes | Kept with caveat |
| Printed form | Class, Literal, Field, float all map | Humans can fill out of order | Predicts field order matters: plausible, untested here | Kept; flagged as untested |
| Scratch paper | Effort = time to think | More paper does not mean right | Predicts cost and seconds rise with effort | Kept |
| Marked exam | Claims, labels, `is_right`, held-out map | A student can answer "refer" | Predicts manual claims fail: the notebook says so | Kept; the `refer` caveat needs data |
| Tier-1 triage | Jev, confidence, threshold map | Depends on calibration | The `confident` groupby tests it | Kept; the notebook runs the test |
| Closed room + clerk | Model, runtime, tools, ledger map | Model re-reads history each call | Predicts 3 calls cost more than 1: matches "three model calls" | Replaced the first draft ("handler at a desk with a filing cabinet"), which failed check 2 because it suggested the model runs the tools itself. |

---

## Flagged terms for later research

Add an entry whenever you say "flag this". Each gets an explanation, a link back to the notebook, and where to learn more.

### Confusion matrix

**Plain version.** A grid that shows, for every possible true answer, what the model actually said. Rows are the truth, columns are the model's answer. Cells on the diagonal are correct. Every off-diagonal cell is a specific kind of mistake. It answers "how did it get things wrong?" where a single score only says "how often".

For this lab it's 3 x 3 (approve / decline / refer), with the handler as the truth:

```
                    model said ->
                  approve  decline  refer
handler  approve     ok      Pique   Pause
said     decline     Pay      ok     Pause
  |      refer       Pay     Pique    ok
  v
```

Note the mnemonic lines up with the **column**: whatever the model wrongly said decides the kind of mistake. Wrongly *approve* = Pay, wrongly *decline* = Pique, wrongly *refer* = Pause. That is the notebook's "kind of mistake matters" point turned into a table.

**Analogy: a mail room with three bins.** Each letter has a correct bin (rows). The sorter drops it into one bin (columns). The diagonal is letters in the right bin, and each off-diagonal cell is a particular mix-up, such as "letters for decline put in approve".

Checked three ways:
1. *Maps?* Yes: correct bin = row, chosen bin = column, diagonal = correct.
2. *Breaks where?* A mail sorter's mistakes all seem equal. In the matrix, the cells are not equal here: paying a claim wrongly (Pay) is not the same as a wasted referral (Pause). Also, the familiar terms (true/false positive) belong to *yes/no* problems. With three options they only apply to one option at a time, such as "approve vs everything else".
3. *Predicts something true about the code?* Yes. The diagonal total is the count of correct *decisions*. `is_right` also requires the payout to be within $1, so the notebook's `right` count can be **lower** than the diagonal total, never higher. Jev is scored on the decision alone, so its `right` count equals its diagonal total.

**Try it** (checked on toy data, not yet on your results). The notebook's result rows have `decision`, and `labels[claim_id]["decision"]` has the truth:

```python
truth = pd.Series([labels[r["claim_id"]]["decision"] for r in luna_low], name="handler")
said = pd.Series([r["decision"] for r in luna_low], name="model")
pd.crosstab(truth, said)
```

**Related terms to flag next, if you want:** precision, recall, F1, false positive vs false negative, class imbalance.

**Follow-up resources** (search by these names; I haven't opened them in this session):
- scikit-learn user guide, "Model evaluation", the `confusion_matrix` section. It also has a plotting helper.
- Google's Machine Learning Crash Course, the classification module on accuracy, precision and recall.
- Wikipedia, "Confusion matrix", for the naming conventions across fields.
