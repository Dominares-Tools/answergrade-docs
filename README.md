# answergrade — Grade chatbot answers against their own sources

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/answergrade**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

answergrade measures recall@k and MRR for your search and fails a change that makes retrieval worse. A cautious AI judge grades each answer against the sources your bot used, and the report says what to fix. It also plans a knowledge-base cleanup and lists what visitors say is wrong.

## Get it

Buy answergrade at https://dominares.org/tools/answergrade: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python library and the answergrade command, with a stand-in help bot to try it on.

Requirements: The Python standard library. The [http] extra adds httpx.

This page documents version 1.1.3. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

**Grade your chatbot's answers against its own sources, measure retrieval before you ship a change, and find what is wrong in the knowledge base it answers from.**

```bash
answergrade eval --chunks examples/chunks.jsonl --cases examples/cases.jsonl
```

That measures retrieval over a small made-up knowledge base that ships in `examples/`, and prints:

```
Retrieval eval: built-in keyword (12 cases, k=5)
  recall@1: 0.917
  recall@5: 0.917
  MRR:      0.917
  ...
  missed (no expected chunk in the top 5): 1
    - 'Can I get my money back?': wanted refunds.md; got nothing
```

Run it from the root of your copy of the repository. No API key, no network.

### What it does

A bot that answers from your documents can get worse without anything breaking. Someone shortens
the chunks, swaps the embedding model, rewrites the prompt, and every request still returns HTTP
200 with a fluent answer. Nothing tells you that "Can I get my money back?" no longer finds the
refund policy, or that the bot now says "we don't offer a student discount" when a help article
describes one.

answergrade gives you three kinds of evidence, all offline and all from the command line:

- **Retrieval numbers, before and after.** Write down the questions your bot must answer and the
  source each answer lives in. `answergrade eval` reports recall@k and MRR for your search, and
  compares a change with a saved baseline. It exits non-zero when the change makes retrieval
  worse, so it can gate a pull request.
- **Answers graded against the sources the bot used.** `answergrade probe` asks your bot a list of
  questions, hands each answer and the sources behind it to a cautious AI judge, and writes a
  report with a fix card for everything that needs a person: an answer that contradicts a
  source, an invented detail, a "done, I've cancelled it" from a bot that cannot cancel anything,
  a question no source covers.
- **A knowledge base you can trust.** `answergrade kb-lint` reads the cards your bot answers from
  and plans a cleanup: exact duplicates, test rows, raw email-blast HTML, tracking and
  unsubscribe links, invisible characters, and pairs of active cards that may contradict each
  other. It reports; it never deletes. `answergrade reported-errors` reads your conversations for
  visitors saying something you published is wrong, and puts that next to the sources your bot
  cites most, so a bad fact quoted to everyone ranks first.

Also in the box: a check that reports how many chunks *really* have a vector (so "semantic search
is on" over an empty cache is caught), a way to build a question set from your chat logs,
counters for which sources your bot cites and which questions it cannot answer (with personal
data scrubbed before anything is stored), and a stand-in help bot so you can watch all of it work
before you point it at yours.

**Who it is for:** teams running a support or documentation bot over their own content (a RAG
app) who want regression evidence before and after changing retrieval, prompts or models, and who
want confidently wrong answers caught without signing up for an evaluation service.

**How it relates to a live checker.** A scheduled live checker asks your production bot a few
must-pass questions and tells you when it is down or lying. answergrade does not repeat that. It
is the offline half: it runs on a laptop or in CI, over many more questions, and tells you whether
a change is safe to ship.

The runtime is the Python standard library only (3.10 or newer). There is no SDK to install and no
account to make. The judge is any function from a prompt to text; two ready-made ones are included
(an OpenAI-compatible HTTP endpoint, and any command-line program).

### Install

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*

Nothing needs the `http` extra; urllib does the same job. To run the test suite from a checkout:
`pip install -e '.[dev]' && python3 -m pytest`.

### 60-second quickstart

The repository ships a small made-up knowledge base ("Northwind Notes", a note-taking app) in
`examples/`, twelve test questions for it, and a stand-in help bot. No API key is
needed.

**1. Measure retrieval.** Twelve questions, each with the chunk that answers it:

```bash
answergrade eval --chunks examples/chunks.jsonl --cases examples/cases.jsonl --save baseline.json
```

```
Retrieval eval: built-in keyword (12 cases, k=5)
  recall@1: 0.917
  recall@5: 0.917
  MRR:      0.917
  ...
  missed (no expected chunk in the top 5): 1
    - 'Can I get my money back?': wanted refunds.md; got nothing
```

Eleven of twelve are found. The miss is a synonym: the refund policy never says "money back".

**2. Try a change that helps.** Give the search a small table of words that mean the same thing:

```bash
answergrade eval --chunks examples/chunks.jsonl --cases examples/cases.jsonl \
    --aliases examples/aliases.json --baseline baseline.json
```

```
Baseline built-in keyword vs candidate built-in keyword + aliases (12 cases, k=5, allowed drop 0.000)
  recall@1   0.917 -> 1.000  (+0.083)
  ...
  now found:  'Can I get my money back?'
  verdict: ok
```

Exit status 0.

**3. Try a change that looks harmless and is not.** The example scorer indexes only the first 60
characters of every chunk, the sort of thing a "smaller chunks" change does:

```bash
answergrade eval --chunks examples/chunks.jsonl --cases examples/cases.jsonl \
    --scorer examples/short_chunks_scorer.py:score_chunks --baseline baseline.json
```

```
  recall@1   0.917 -> 0.583  (-0.333)  REGRESSION
  recall@5   0.917 -> 0.750  (-0.167)  REGRESSION
  MRR        0.917 -> 0.667  (-0.250)  REGRESSION
  now missed: 'How do I download all my notebooks?'
  now missed: 'Is two-factor authentication available?'
  verdict: FAIL
```

Exit status 1, and the two questions the change broke are named.

**4. Check the vectors.** In the example cache, 10 of the 12 chunks have an entry. One entry is
all zeros and one has the wrong length, so 8 are usable. Two chunks have none, and one entry
belongs to a chunk that no longer exists:

```bash
answergrade coverage --chunks examples/chunks.jsonl --vectors examples/vectors.jsonl
```

```
Meaning-based search covers part of the knowledge [partial]
  8 of 12 chunks have a vector (67%). Questions about the rest fall back to keyword matching. Rebuild the missing vectors.
  ...
  missing: 2 (trash#1, integrations#1)
  empty: 1 (security#1)
  wrong dim: 1 (education#1)
  orphans: 1 (pricing-old#1)
```

Exit status 1. The setting says "on"; two thirds of the knowledge is actually covered.

**5. Grade a bot.** Start the stand-in bot, then probe it. `--judge stand-in` is a rule-based
judge for this demo only; it is not a language model (see [Choosing a judge](#choosing-a-judge)
for a real one).

```bash
answergrade demo &                                   # Northwind Notes on 127.0.0.1:8766, answers honestly
answergrade probe --questions examples/questions.jsonl --bot http://127.0.0.1:8766/chat \
    --judge stand-in --facts examples/facts.txt --out report.html
```

```
8 of 8 graded answers passed (100%). 0 flagged, 0 with no answer.
...
What to fix:
  LOOK #7 No source covers “Is there a Linux desktop app?”
```

Exit status 0. The one warning is real: nothing in the knowledge base answers the Linux question.
Now make the bot wrong on purpose:

```bash
kill %1; answergrade demo --mode wrong &
answergrade probe --questions examples/questions.jsonl --bot http://127.0.0.1:8766/chat \
    --judge stand-in --facts examples/facts.txt --out report.html --out run.json
```

```
2 of 8 graded answers passed (25%). 6 flagged, 0 with no answer.
What to fix:
  FIX  #1 The answer contradicts source plans#1
       The answer says $10, which the sources do not. The source says: "Plus costs $8 per month ..."
  FIX  #3 The answer claims an action the bot cannot take
  FIX  #4 The answer contradicts source support#1
  ...
```

Exit status 1. Open `report.html` for the same thing as a page, or `answergrade report run.json`
to read the saved run again. Stop the demo with `kill %1`.

### Use it on your own bot

#### 1. Put your knowledge in chunks

A chunk has an `id`, its `text`, and the `source` it came from (a file name, a URL, an article
id). One JSON object per line:

```json
{"id": "refunds#1", "text": "Refunds are issued within 14 days ...", "source": "refunds.md"}
```

`--chunks` also takes a JSON list, or a **directory** of `.txt`, `.md` and `.html` files, which is
split at blank lines (ids become `refunds.md#1`, `refunds.md#2`, and the source is the file name).
Export the chunks your bot really searches; the closer they are to production, the more the numbers
mean.

#### 2. Write down the questions and where the answers live

```json
{"question": "Can I get my money back?", "expected": ["refunds.md"], "kind": "synonym"}
```

`expected` names chunk ids or sources; a source counts once however many of its chunks rank.
`kind` is optional and groups the report ("lexical", "synonym", "paraphrase"...). A typo in an
expected id is reported as an error instead of quietly counting as a miss. Start with the twenty
questions your customers ask most, add one whenever the bot gets something wrong, and keep the
file in version control.

#### 3. Point eval at your search

The built-in scorer is a stemmed keyword ranker with an optional alias table. It is a floor to
beat, not a claim about your stack. To measure **your** search, give a function or object with
this shape:

```python
def score_chunks(chunks, query, limit):
    """chunks: list of str. Return (words, [(score, chunk_text), ...]) best first, at most `limit`."""
```

```bash
answergrade eval --chunks chunks.jsonl --cases cases.jsonl --scorer my_search.py:score_chunks
```

`--scorer` is `file.py:name` or `package.module:name`; the name can be a function or an object with
a `score_chunks` method. The file may import modules that sit beside it.

What the scorer returns can be whatever your search naturally gives back, alone or as
`(anything, results)`, best first. Each result may be the chunk's **text**, its **id**, a dict
with an `id` or `text` (`{"id": "refunds#1", "score": 0.83}`, the usual vector-database shape), or a
`(score, result)` pair. Text is matched ignoring case, extra spaces, HTML and invisible
characters, so your stored copy of a chunk still matches. If a scorer returns results and none of
them is one of your chunks, `eval` stops with an error naming what it got, instead of reporting a
recall of zero. A scorer that raises also stops the run with exit `2`, not `1`, so a broken
scorer is never read as a regression.

Save a result with `--save baseline.json` before you change anything, commit it, and compare the
change against it with `--baseline baseline.json`. `--max-drop 0.02` allows a two-point dip in any
metric before it counts as a regression. Two runs over different questions are never compared: the
gate fails and says so.

`--save` and `--baseline` may name the same file, which is the natural "check, then roll the
baseline forward" step. The baseline is read first, so the comparison is against the old numbers.
The file is overwritten only when the candidate passes; a regression leaves the old baseline in
place.

A pull-request gate is one step:

```yaml
- run: python3 -m pip install ./answergrade   # wherever you vendor or check out this repository
- run: answergrade eval --chunks data/chunks.jsonl --cases data/cases.jsonl --scorer app.search:scorer --baseline data/baseline.json
```

#### 4. Grade answers

The judge reads one answer and the sources the bot had. Its prompt tells it to flag an answer only when it clearly:

| flag | meaning |
|------|---------|
| `contradiction` | contradicts or denies something a source or fact states ("no weekend support" when a source lists Saturday hours) |
| `fabrication` | states a specific detail (a price, a date, a link, a policy term) that conflicts with the material or is plainly invented |
| `fabricated_action` | claims an action the bot cannot take from a chat ("I've cancelled your subscription") |

The prompt also tells it not to flag tone, length, a polite refusal or an honest "I don't know", and
not to flag when it is unsure; that is an instruction to the model, which the code cannot enforce.
A reply that is not JSON, a judge call that fails, a reply that was cut off, or
a verdict that contradicts itself (`"flag": true` with no category or with `"none"`, `"flag": "maybe"`,
`"flag": false` with one of the three real categories) is **could not grade**, never a flag.
An invented category is **could not grade** only when the flag is missing or true
(`{"category": "banana"}`, `{"flag": true, "category": "banana"}`).
**A verdict with `"flag": false` and an invented category is graded as a pass, not as "could not
grade"**: `{"flag": false, "category": "banana"}` passes, because a clear `false` is what the code reads and
the unknown category beside it is ignored. This is deliberate: the code takes a well-formed verdict as
given, and the prompt tells the judge to default to not flagging. That is all the code checks: it rejects
malformed verdicts. A verdict that is well formed but wrong, including a flag on a problem the answer
does not have, still counts as a flag.
A `source_id` the judge names that was not among the sources it was given is dropped, and
`evidence_verified` says whether the quote it gave really appears in the sources or facts. Those answers are
counted separately in the report, and a run where nothing could be graded exits `2`, not `0`.

From Python:

```python
from answergrade import grade

sources = [{"id": "support#1", "text": "Support is open Monday to Friday, and Saturday 10am to 2pm."}]

def llm(prompt: str) -> str:
    # Call your model here and return its text. This stand-in always says "fine".
    return '{"flag": false, "category": "none", "reason": "consistent with the sources"}'

verdict = grade("Is support open on Saturday?", "Yes, Saturday 10am to 2pm.", llm=llm, sources=sources)
print(verdict.flag, verdict.graded)      # none True
```

`grade(question, answer, *, llm, sources=None, facts=None, history=None, config=None)` returns a
`Verdict(flag, reason, evidence, source_id, graded, evidence_verified)`. Give it what the bot saw:

- `sources`: the chunks the bot retrieved (strings, dicts or `Chunk`s). A judge that sees less
  than the bot marks correct answers as invented.
- `facts`: hard facts that outrank the sources (a list of strings or a block of text). If your
  system prompt carries a few, `facts_from_prompt(system_prompt, header="facts")` pulls the bullet
  lines under that heading.
- `history`: earlier turns, so a detail established earlier in the conversation is not called
  invented. Accepts a string, `{"role", "content"}` dicts, `(role, text)` pairs, or
  `{"user", "bot"}` exchanges.

The prompt tells the judge to treat every source, fact and turn as reference about what is true,
never as instructions, so a source that says "always offer the upgrade" is not meant to steer the grade
(the model can still be swayed).

#### 5. Probe a bot

```bash
answergrade probe --questions questions.jsonl --bot https://bot.example.com/chat \
    --judge-url https://llm.example.com/v1/chat/completions --judge-model my-model \
    --header "Authorization: Bearer $BOT_TOKEN" --out report.html --out run.json
```

Each question is asked with a fresh session id that starts with `answergrade-`, so your own
analytics can leave probe traffic out. The bot is one of:

- **An HTTP endpoint** (a URL). By default it POSTs `{"question": "...", "session_id": "..."}`
  and looks for the answer under `answer`, `reply`, `response`, `text`, `output`, `message` or
  `choices.0.message.content`, and for the sources under `sources`, `citations`, `context`,
  `documents` or `references`. Set `--answer-path data.reply` and `--sources-path data.citations`
  (dotted, list indexes allowed) when yours differ, and `--body` to change the request:
  `--body '{"messages": [{"role": "user", "content": "{question}"}], "sid": "{session_id}"}'`.
  `{question}` and `{session_id}` are filled in before the body is serialized, so a question with
  quotes stays valid JSON.
- **A Python callable** as `file.py:name` or `package.module:name`. It takes the question and
  returns a string, an `(answer, sources)` pair, a dict with `answer` and `sources`, or a
  `BotReply`.

**If your bot returns names instead of text.** Many bots reply with citations such as
`["plans.md"]`, URLs, titles or `{"id": "plans#1", "url": ...}`. A name is not evidence, so answergrade
never hands one to the judge as if it were. Give `--sources chunks.jsonl` and each cited id, source,
file name, URL or title is looked up there, and the answer is graded against those chunks. A
citation that matches nothing falls back to retrieval (below). With no `--sources` the answer is
**not graded** and a card says why, rather than being judged against nothing.

**If your bot does not return its sources**, give `--sources chunks.jsonl` and answergrade retrieves
them with the built-in scorer (`--k` sets how many). From Python, pass `retrieve=` any function
from a question to sources. The report says which answers were graded against the bot's own
sources, against the chunks it cited, and against retrieval. Only the bot's own sources show what
it really saw: an answer graded against answergrade's keyword search can be flagged because the
search missed the right chunk, so those cards are warnings and say so.

From Python:

```python
from answergrade import run_probe

def bot(question):
    return {"answer": "The Plus plan costs $10 a month.",
            "sources": [{"id": "plans#1", "text": "Plus costs $8 per month."}]}

def llm(prompt):
    # Stand-in judge. A real one returns the model's JSON reply.
    return ('{"flag": true, "category": "contradiction", "reason": "Wrong price.", '
            '"source_id": "plans#1", "evidence": "Plus costs $8 per month."}')

run = run_probe(["How much is Plus?"], bot, llm=llm)
print(run.headline())          # 0 of 1 graded answers passed (0%). 1 flagged, 0 with no answer.
print(run.cards()[0]["title"]) # The answer contradicts source plans#1
print(run.exit_code())         # 1
```

**The fix cards** say what to change, not just that something is wrong:

| card | severity | what it means |
|------|----------|---------------|
| The answer contradicts source *X* | fail | the bot ignored the source, or the source is out of date |
| The answer states something no source supports | fail | add a source, or tell the bot to say it does not know |
| The answer states something the retrieved sources do not support | warn | graded against answergrade's search, not the bot's own sources; check by hand |
| No source covers "*question*", and the bot filled the gap | fail | a hole in the knowledge base that the bot papered over |
| The answer claims an action the bot cannot take | fail | a prompt or tool problem, not a knowledge problem |
| The bot did not answer | fail | it is down, or the request shape or answer path is wrong |
| No source covers "*question*" | warn | nothing was retrieved; if the bot also denied it exists, that may be a retrieval miss |
| The bot named sources that answergrade cannot read | warn | it cited names without text; give `--sources` |
| The expected source was not retrieved | warn | (when a question has `expected`) add it to your eval cases |

Exit status: `0` nothing to fix; `1` a flagged answer or a bot that did not answer (with
`--strict`, also any warning); `2` nothing could be graded.

#### Choosing a judge

`probe` and `questions --generate` need a judge. One of:

- `--judge-url URL --judge-model NAME`: any OpenAI-compatible chat-completions endpoint, hosted
  or local (Ollama, vLLM, LM Studio, llama.cpp). Set the key in `ANSWERGRADE_JUDGE_KEY`.
- `--judge-command "PROGRAM ARGS"`: a program that reads the prompt on stdin and prints the reply.
  No shell is involved. Wrap any model CLI or script you already have.
- `--judge stand-in`: the demo's rules. It only knows prices, percentages, day counts and a few
  phrases. Use it to try answergrade, never to grade a real bot.

Some endpoints reject a field the default request carries. Each has a switch:

| the endpoint says | use |
|-------------------|-----|
| `max_tokens` is not supported; use `max_completion_tokens` (OpenAI reasoning models) | `--judge-max-tokens-param max_completion_tokens` |
| `temperature` must be the default (reasoning models) | `--judge-no-temperature` |
| `response_format` is not accepted (LM Studio, some llama.cpp builds) | `--judge-no-json-mode` |
| you want another field in the body, such as `reasoning_effort` | `--judge-param reasoning_effort=low` (repeatable; the value is JSON when it parses) |
| replies are cut off (`finish_reason=length`) | `--judge-max-tokens 1000`; `0` leaves the field out |

A reasoning model's `<think>...</think>` block before the JSON is ignored. The default reply limit
is 400 tokens for a verdict and 2000 for `questions --generate`.

**Check the model before you trust it.** `answergrade judge-check` sends ten answers whose grade is
known (five wrong, five right) through the judge you chose and lists what it got wrong:

```bash
answergrade judge-check --judge-url https://llm.example.com/v1/chat/completions --judge-model my-model
```

```
Judge check: known answers, known grades
  ok   wrong price            expected contradiction      got contradiction
  ...
  10 of 10 as expected. 0 good answers flagged, 0 bad answers passed, 0 not graded.
```

Exit `0` when every case comes out as expected, `1` when one does not, `2` when nothing could be
graded. Ten cases show a model that cannot do the job, not that one can do every job; your own
question set is the longer test.

Or set `ANSWERGRADE_JUDGE_URL`, `ANSWERGRADE_JUDGE_MODEL` and `ANSWERGRADE_JUDGE_KEY` once. From
Python, `answergrade.llm.openai_compatible(url, model, api_key=...)`, `command(argv)` and
`from_env()` return the same `prompt -> text` functions, or write your own. Use a model at
temperature 0; the included HTTP judge does.

A capable model is worth paying for here. The judge is conservative by design, and a weak one
will miss subtle contradictions and may flag answers that are fine, so it will both pass some bad
answers and flag some good ones.
Read the flagged answers and a sample of the passed ones for the first few runs.

#### 6. Build the question set

```bash
answergrade questions --from-logs chat-log.jsonl --out questions.jsonl
answergrade questions --from-logs chat-log.jsonl --generate 20 --sources chunks.jsonl \
    --judge-url ... --judge-model ... --out questions.jsonl
```

The log is JSONL, one message per line: `{"session_id": "...", "role": "user", "text": "..."}`
(the text may also be under `question`, `message`, `content`, `user` or `body`; a `channel` of
`email` is treated as email). answergrade keeps real questions and drops what is not one: greetings and thanks
("hi there", "thank you so much!", "yes please"), quoted email history under a real question (`On ... wrote:`, `> ...`, forwarded header
blocks), task prompts that point at content it will not have ("rewrite this to sound friendlier"),
pasted letters, and duplicates (case-insensitive). A "Hi," in front of a question or a "Thanks!" after
it is trimmed. Ordinary questions that use words like *edit*, *fix* or *translate* ("How do I edit the
email address on my account?") are kept. Probe traffic (session ids starting
`answergrade-`) is left out.

`--generate N` asks the judge model for N new, self-contained questions grounded in your sources,
avoiding the ones you already have. Each request shows the model a spread of your sources (a chunk
from each in turn, not the first few files), and more than 20 questions are asked for in batches
that look at different sources. Generated questions go through the same filter. Real questions
come first in the file, with `source` telling you which is which. `--count` caps the total.

#### 7. Count what the bot cites and cannot answer

Two small counters you call from your bot, one line each. They are cheap enough for every turn and
never raise: a bookkeeping error is logged, not thrown into a chat.

```python
from answergrade import Citations, Gaps

citations = Citations("answergrade-data")   # a directory; or any store, see below
gaps = Gaps("answergrade-data")
session_id, user_message = "s-1", "Do you have a Linux app? Call me on 619-555-0123"

# after answering:
citations.record(["refunds.md"], session_id=session_id)
# when retrieval found nothing:
gaps.record(user_message)
```

`Gaps` scrubs the question **before** it is stored: emails, phone numbers, money amounts (`$49`, `20€`, `¥5000`, `Rs. 500`, `30 dollars`) and runs of
six or more digits (card, account and order numbers, with or without a letter prefix such as
`A12345678`) become `[email]`, `[phone]`, `[amount]` and
`[number]`. A five-digit postal code is left alone. Two questions that differ only in personal data
count as one gap.

```bash
answergrade kb --store answergrade-data --known-sources chunks.jsonl
```

```
Knowledge base
  sources cited: 1, 1 citations in all, 1 in the last 30 days
  most cited:
         1  refunds.md  (last 2026-09-30T06:37:06Z)
  never cited: 10
    billing.md
    cancel.md
    ...
  unanswered questions:
         1  do you have a linux app? call me on [phone]
```

The rollup also names **stale** sources: at least 5 lifetime citations and none in 30 days, which
usually means a page was replaced or retrieval stopped finding it. Per-source history is kept for
60 days, and `citations.sources_for_session(id)` answers "which sources produced this
conversation's answers". If you know the chunk the bot used but not the document it came from,
`sources_containing(chunk_text, {source_id: full_text})` finds it.

The counters live in a store the package owns. `JsonFileStore(directory)` writes one JSON file per
counter, atomically, under a cross-process lock, so several web workers can count into one
directory without losing increments. `MemoryStore()` is for tests. To keep the counters in your own
database, pass any object with two methods:

```python
class MyStore:
    def load(self, name: str) -> dict: ...          # the current document ({} when there is none)
    def transaction(self, name: str): ...           # a context manager yielding the document as a dict;
                                                    # what it holds on a clean exit is saved, all at once
```

#### 8. Lint the knowledge store

A store that people and imports write to for months collects the same dirt, and none of it makes
anything fail: the same card twice, a literal `test` row, a 20 KB email blast stored as a card, a
tracking-wrapped link the bot pastes into replies as "the link", a zero-width character that splits
a word so nothing can match it, two active cards that say different things.

```bash
answergrade kb-lint --rows examples/kb-rows.jsonl --protect-tag owner-note
```

```
Knowledge lint: 10 rows, 8 kept after the plan
  would remove (2):
    test_row           kb-3: title 'test' and a placeholder body
    duplicate          kb-1: same title and body as kb-2; kb-2 is newer, so it is kept
  would rewrite (2):
    tracking_link      kb-10: tracking or unsubscribe links removed
    html               kb-4: markup, 371 characters flattened to 80 of text
  for a person to decide (2):
    transcript         kb-7: reads like a meeting transcript, which ranks on many queries and can carry old guidance
    conflict           kb-5, kb-6: 2 active rows with overlapping titles ('Student discount'; 'Student discount policy') and different bodies; decide which one is right
  Nothing was deleted or changed. This is a plan.
```

Exit status 1 when there is anything to remove, rewrite or review, 0 for a clean store. `--rows` is
JSONL, a JSON list, or a folder of `.txt`, `.md` and `.html` files, read as they are (markup is not
flattened on the way in, because finding it is the point). A row has an `id`, `title`, `body`, and
optionally `tags` (a comma-separated string or a list), `status` and `created_at`; `--id-field`,
`--title-field`, `--body-field`, `--tags-field`, `--status-field` and `--created-field` rename them.

What it does, per row:

- **Removes** (planned): an exact duplicate, comparing title and body after cleaning and ignoring
  case and spacing. A protected copy is kept; otherwise the newest is; with no dates (a folder of
  files has none) the one listed first is. Each finding says which of those decided it. A test row:
  a title of `test`, `testing`, `tests` or `(no subject)` with an empty, placeholder or very short
  body (under 60 characters, and the finding says so, because that body can be real), or a body
  that is only `test`.
- **Rewrites** (planned): markup flattened to text (a body of 20,000 characters or more is called
  an email blast; `--blast-chars` moves that); tracking-wrapped links (the common mail platforms'
  click hosts) and unsubscribe or manage-preferences links deleted, every other link kept unless
  you pass `--strip-urls`; control bytes and zero-width characters removed. A row whose body would
  be empty after all that is reported, not emptied.
- **Reports for a person**: a likely contradiction, and a transcript.
- **Leaves alone**: a row with a `--protect-tag`. It is never removed, rewritten or reported, and it
  wins over an unprotected copy of itself. `--publish-tag TAG` is the reverse for text you publish:
  rows with that tag have `[bracketed internal text]` stripped from the body and the title, and
  look-alike characters (full-width forms, mathematical bold letters, no-break spaces) folded.

`--cleaned-out FILE` writes the rows that survive the plan, as JSONL, to a new file. It refuses to
overwrite the file you read. Your store is never opened for writing; applying the plan is yours.

```python
from answergrade.lint import lint

rows = [
    {"id": "r1", "title": "Refund policy", "body": "Refunds within 30 days.", "created_at": "2026-05-01"},
    {"id": "r2", "title": "Refund policy", "body": "Refunds within 30 days.", "created_at": "2026-06-01"},
    {"id": "r3", "title": "test", "body": "test", "created_at": "2026-06-02"},
]
plan = lint(rows)
print(plan.remove_ids)                       # ['r3', 'r1']
print([r["id"] for r in plan.kept], plan.clean)    # ['r2'] False
```

`lint(rows, fields=Fields(...), protected_tags=(), publish_tags=(), strip_urls=False)` returns a
`LintPlan` with `kept`, `remove_ids`, `rewritten_ids`, `conflict_groups` and `findings`. The pieces
are importable on their own: `scrub_tracking_urls`, `strip_all_urls`, `looks_like_transcript`,
`strip_bracketed`, `fold_for_publish`, `publish_text`.

The contradiction report is a title cluster: two active rows whose titles share at least 60% of
their longer-than-two-letter words (measured against the shorter title) and whose bodies differ. It
does not read what the bodies say, so it cannot tell a real contradiction from two cards that
legitimately differ, and it cannot find two cards that disagree under different titles. Treat it as
a list of places to look.

#### 9. Find the sources your visitors say are wrong

`Citations` tells you which sources the bot cites. Your conversations tell you what people say about
the bot. `reported-errors` puts the two side by side: a visitor writes "the opening hours you list
are wrong" while the hours file sits among the most-cited sources, so the bot keeps handing the bad
hours to everyone who asks.

```bash
answergrade reported-errors --conversations examples/complaints.jsonl --cited plans.md --cited support-hours.md
```

```
Reported errors: 2 subjects
  2 people report a problem with the prices you publish
    They reported it separately. That is among the sources your bot quotes most, so it is probably being repeated to everyone who asks.
    most-cited sources about it: plans.md
    open: s-202 (+1 more)
    said: 'The price of the Plus plan on your pricing page is wrong. I was charged $9, not $8.'
    said: 'your plans page is out of date, Team costs more than that now'
  Someone reports a problem with the hours you publish
    That is among the sources your bot quotes most, so it is probably being repeated to everyone who asks.
    most-cited sources about it: support-hours.md
    open: email:m-9
    said: 'Your support hours are wrong on the help page, you close at 4 on Saturdays.'
```

Exit status 1 when something is reported, 0 when nothing is. `--store DIR` takes the most-cited
sources from the directory your bot's `Citations` writes to (`--top N` of them); `--cited NAME` adds
a source by hand. `--conversations` reads JSONL or a JSON list, one record per message, with the
message under `text`, `message`, `content`, `body`, `user`, `preview`, `snippet`, `subject` or
`question`; the same chat log that `answergrade questions --from-logs` reads works here. Only the visitor's side is read: a
record whose `role` is not a user role, or that is `outbound` or `deleted`, is skipped, and quoted
history is cut from email records. `--only-open FIELD` reads only records where that field is true.

How it decides:

- **A defect is not a disagreement.** Some phrases are taken as asserting a published fact is wrong
  on their own ("mispublished", "listed incorrectly", "out of date", "needs updating", "wrong
  address"); they are plain substring matches (see the limits). Weaker
  ones ("is wrong", "incorrect", "should say") count only within 60 characters of a word that names
  something published: hours, an address, a phone number, a date, a price, a page. "The director is
  wrong about the new policy" is an opinion and is not reported.
- **Grouped by subject, not by reporter.** Two people naming the same wrong thing is one finding
  with a count, and that is the strongest evidence it needs fixing. People are counted, not
  messages: records with the same `session_id` (or `session`), the same contact email or phone, or
  the same address count as one person, and the report then says "N messages from one visitor"
  instead of "reported it separately". Records with none of those are counted one each.
- **Subject first, then reach.** A message gets its subject from the table below whether or not
  you give any cited sources. Without `--cited` or `--store`, nothing is called quoted and the
  report says so.
- **Sized by what the bot quotes.** A subject counts as quoted when the name of a most-cited source
  contains one of its words (`office-hours.md` for hours). Reports about a quoted subject come
  first. That is a match on file names, not a reading of the files; name your sources so the
  subject shows.
- **Subjects** are a small table (hours, location, phone, pricing, events). `--topics FILE` replaces
  it with `{"shipping": ["shipping", "delivery"]}` or
  `{"shipping": {"words": ["delivery"], "phrase": "the delivery times you publish"}}`; the words of
  your subjects then also let a weak phrase count. A complaint about something outside the table is
  still reported, without a subject and without a claim about reach; the header counts those
  separately ("1 group with no subject").
- **A mail robot is never named** as the reporter (`noreply`, `mailer-daemon` and similar);
  the report says "Someone".

```python
from answergrade import Citations, MemoryStore
from answergrade.reported import find_reported_errors

cites = Citations(MemoryStore())
cites.record("office-hours.md")
messages = [{"id": "chat-7", "text": "The opening hours on your site are wrong, you close at 4."}]
for found in find_reported_errors(messages, cites.rollup()["top_cited"]):
    print(found.headline)
    print(found.quoted_sources)
```

```
Someone reports a problem with the hours you publish
(('office-hours.md', 1),)
```

### Configuration reference

#### Commands

| command | purpose | exit `0` | exit `1` | exit `2` |
|---------|---------|----------|----------|----------|
| `eval` | recall@k and MRR; compare with a baseline | no regression | regression beyond `--max-drop` (or runs not comparable) | bad file, unknown expected id, a scorer that raises or returns none of your chunks |
| `coverage` | how many chunks really have a vector | working, or search switched off | partial, broken, no cache, no key | bad file |
| `probe` | ask the bot, grade, report | nothing to fix | flagged answer or bot error | nothing graded, no judge, bad file |
| `report` | render a saved run | as `probe` | as `probe` | not a saved run |
| `questions` | build a question set | wrote the file | | no questions, bad file |
| `kb` | print citations and unanswered questions | printed | | no such directory |
| `kb-lint` | plan a cleanup of a knowledge store | nothing to remove, rewrite or review | something to remove, rewrite or review | bad file, `--cleaned-out` would overwrite the input |
| `reported-errors` | match complaints to the most-cited sources | nothing reported | a published fact is reported wrong | bad file or directory |
| `judge-check` | run ten known answers through your judge | all as expected | a case came out wrong | nothing graded |
| `demo` | serve the stand-in help bot | | | |

`answergrade --version` and `python3 -m answergrade` work too.

#### `eval`

| flag | default | |
|------|---------|--|
| `--chunks FILE` | required | JSONL, a JSON list, or a directory of text files |
| `--cases FILE` | required | JSONL of `{question, expected, kind}` (a `.json` list works too) |
| `--k N` | 5 | how many results count |
| `--scorer SPEC` | built-in keyword | `file.py:name` or `module:name` |
| `--aliases FILE` | | JSON object `{"refund": ["money back", "reimburse"]}` for the built-in scorer |
| `--baseline FILE` | | a result saved earlier with `--save` |
| `--baseline-scorer SPEC` | | a second scorer to compare against instead of a file |
| `--max-drop X` | 0 | the drop in any metric allowed before it is a regression (0.02 is two points) |
| `--save FILE` | | write this result as JSON |
| `--json` | | print JSON |

#### `coverage`

`--chunks FILE`, `--vectors FILE` (JSONL of `{id, vector}` or a JSON object `{id: vector}`),
`--min-coverage 0.90` (below this share the state is *partial*), `--disabled` (semantic search is
off on purpose), `--no-key` (no embedding credentials), `--json`. A vector counts only if it is
a list of finite numbers, not all zeros, and the same length as the rest. States: `off`, `no_key`,
`no_cache`, `broken` (under 5% covered), `partial`, `working`. An empty knowledge base is never
`working`. From Python: `audit_vectors(chunk_ids, vectors)` and `describe_coverage(...)`.

#### `probe`

| flag | default | |
|------|---------|--|
| `--questions FILE` | required | JSONL of `{question, topic, expected}` or a `.txt` with one per line |
| `--bot TARGET` | required | an `http(s)` URL, or `file.py:name` / `module:name` |
| `--header "Name: value"` | | for an HTTP bot; repeatable |
| `--body JSON` | `{"question": "{question}", "session_id": "{session_id}"}` | request template |
| `--answer-path`, `--sources-path` | auto | where to read them in the reply (dotted) |
| `--sources FILE` | | chunks to retrieve from when the bot returns none |
| `--k N` | 5 | sources retrieved per question with `--sources` |
| `--facts FILE` | | text of facts that outrank the sources |
| `--describe TEXT` | a customer-facing help chatbot | how the judge should describe your bot |
| `--workers N` | 4 | questions asked at once |
| `--timeout S` | 60 | seconds per bot answer |
| `--limit N` | | ask only the first N |
| `--label TEXT` | | a name for the run |
| `--out FILE` | | `.html`, `.json` (readable by `report`) or `.txt`; repeatable. The folder must exist; this is checked before the first question is asked |
| `--strict` | | also exit 1 on warnings |
| `--json` | | print JSON |
| `--judge-url`, `--judge-model`, `--judge-command`, `--judge stand-in`, `--judge-timeout 60`, `--judge-max-tokens`, `--judge-max-tokens-param`, `--judge-no-temperature`, `--judge-no-json-mode`, `--judge-param` | | see [Choosing a judge](#choosing-a-judge) |

#### Environment variables

| variable | |
|----------|--|
| `ANSWERGRADE_JUDGE_URL` | judge endpoint when no flag is given |
| `ANSWERGRADE_JUDGE_MODEL` | judge model name |
| `ANSWERGRADE_JUDGE_KEY` | bearer token for the judge endpoint (never passed on the command line) |
| `ANSWERGRADE_HTTP` | set to `urllib` to use urllib even when httpx is installed |

#### `JudgeConfig`

```python
from answergrade import JudgeConfig
config = JudgeConfig(
    bot_description="the help bot for Northwind Notes",   # how the prompt describes your bot
    extra_rules=("Prices are quoted before tax.",),        # your own rules, one line each
    max_source_chars=12000, max_facts_chars=4000, max_history_chars=4000, max_answer_chars=6000,
)
```

Long material is cut to these limits so a huge retrieval cannot push the question and the answer
out of the judge's context. Pass it as `grade(..., config=config)` or `run_probe(...,
judge_config=config)`.

### Honest limits

- **The judge is a language model.** It is told to be conservative, and the code turns a malformed
  or failed reply into "could not grade" instead of a flag. Nothing checks a well-formed verdict
  against the truth, so it can be wrong in both directions: it can miss a wrong answer and it can flag
  a good one (`evidence_verified` only says whether its quote appears in your sources).
  Treat a flag as "a person should look", and read some passes too. The stand-in judge is much
  weaker than a real model.
- **It grades against what the bot retrieved.** If retrieval missed the right source, the judge
  cannot see the truth either; that is what the eval half is for, and why a card says "no source
  covers this" instead of accusing the bot.
- **The built-in scorer is a baseline.** It ranks by stemmed keywords with aliases. It does not
  embed anything. Use your own scorer to measure the search you actually run.
- **Recall depends on your cases.** Twelve questions prove a scorer works, not that it is good.
  Numbers from a case set that resembles your traffic mean something; numbers from a toy set do not.
- **English text.** Stemming and the greeting, email and denial rules assume English.
- **It does not call your embedding provider.** `coverage` reads a vector file you export; it
  cannot rebuild missing vectors.
- **Question generation and grading spend your model quota.** One judge call per graded answer.
- **kb-lint plans; it does not apply.** It never opens your store for writing, and the contradiction
  report is a title match, not a comparison of what two cards say (see section 8). Duplicates are
  exact after cleaning: two cards that say the same thing in different words are two cards.
  The markup flattener keeps text only, so a link inside an HTML anchor goes with its tags; an
  unsubscribe link is removed only where it appears as text.
- **reported-errors reads English phrases and matches file names.** A complaint in another language, or
  worded without any of the phrases, is not found. It cannot tell whether the complaint is true,
  only that someone made it and what the bot cites most. A source whose name does not contain its
  subject's words is never called quoted.
- **reported-errors fires on phrases, wherever they appear.** The phrases that need no topic word
  ("out of date", "outdated", "wrong number", "wrong time" and the rest) match any message that
  contains them, including ones that are not about your content: "my card is out of date", "sorry,
  wrong number, I meant to text my sister" and "is my browser outdated?" are all reported. Read the
  quoted messages before you act on a count.
- **reported-errors prints what people wrote.** The report names reporters (a name, an email address
  or a phone number from the record) and quotes their messages as given. Nothing is stored, and
  nothing is scrubbed (`Gaps` scrubs before it stores; this command does not). Treat the output the
  way you treat the log it came from. An email record's short fields (under 12 characters once
  quoted history is cut) are dropped.
- **The demo bot is a toy.** It exists so the quickstart runs anywhere.
- **Not a live monitor.** It runs when you run it. Pair it with a scheduled check of your live bot.

### Layout

```
answergrade/         the package (standard library only)
  retrieval.py       recall@k, MRR, baselines and the regression gate
  coverage.py        real vector coverage
  judge.py           the grading prompt, verdict parser, facts and denial helpers
  probe.py           bots, the probe run, fix cards
  questions.py       question sourcing from logs and from sources
  kb.py store.py     citation and gap counters, and the store they live in
  lint.py            kb-lint: duplicates, test rows, markup, tracking links, likely contradictions
  reported.py        reported-errors: complaints matched to the most-cited sources
  llm.py             HTTP and command judges
  judgecheck.py      ten known answers to test a judge model with
examples/            the Northwind Notes knowledge base, cases, questions, logs and a scorer
tests/               the test suite (no network; fakes for every model and HTTP call)
```

---

Docs mirror of the private repository Dominares-Tools/answergrade; updated with each release.
