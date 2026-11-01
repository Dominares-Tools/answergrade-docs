# Changelog

## 1.1.3 (2026-10-31)

README only. No code change.

- The README now says plainly what a judge's clear `false` does with a category the rubric does not know: the case is a PASS (a clear verdict is taken as given), not "could not grade". The behaviour is unchanged since 1.1.0; only the explanation is new.

## 1.1.2 (2026-10-05)

Tests only. No code change.

- Buyer test suite restored; no code change. More of the README-checking tests now ship and run in your copy. Version number only otherwise.

## 1.1.1

Fixes to `kb-lint` and `reported-errors` from a review of 1.1.0, and a list of how they differ
from the code they were extracted from.

- `reported-errors` counts people, not messages. Several messages with the same `session_id`,
  contact email or phone, or address are one reporter, and the report says "N messages from one
  visitor" instead of "N people ... reported it separately". New `ReportedError.people` and
  `reporter_key()`. Chat logs with one record per message no longer inflate the count.
- `reported-errors` takes a message's subject from the topic table whether or not any cited source
  is given. Before, with no `--cited` or `--store`, every complaint landed in one group. The text
  about the bot quoting a source now appears only when a cited source matches the subject.
- `reported-errors` reads the `user` key, so a `{"user": ..., "bot": ...}` log works with both
  `questions --from-logs` and `reported-errors`. 1.1.0 read it in one and silently found nothing
  in the other.
- `kb-lint` says why a duplicate was removed from the facts ("kept row is protected", "is newer",
  "same date", "no dates, the one listed first is kept"). 1.1.0 always said "the newer copy is kept".
- `kb-lint --publish-tag` strips `[bracketed internal text]` from the title as well as the body.
- `kb-lint` describes a `Test`-titled row with a short body as having a short body (with its
  length), not as having "no real content".
- README: the limits now say the no-topic-word phrases match any message containing them.
- Deliberate changes from the code kb-lint and reported-errors were extracted from: the default
  topic table gained `pricing` and `webinar`/`opening`, `office` became `location`, and the
  topics and words that only fitted the original's one audience were dropped; the factual words and the no-topic-word phrases
  gained price terms and "wrong price"; and in `kb-lint` a protected row is entered in the
  duplicate map, so an unprotected copy of it is removed (the original skipped such rows).

## 1.1.0

Two additions to the knowledge-base half, and a README correction.

- `answergrade kb-lint` (and `answergrade.lint`): plans the cleanup of a knowledge store from
  JSONL, a JSON list or a folder of files. Finds exact duplicates (the newest is kept), test rows,
  raw email-blast markup, tracking-wrapped and unsubscribe links, control and zero-width
  characters, transcripts, and pairs of active cards with overlapping titles and different bodies
  (a title match, not a reading of the bodies). It plans removals and rewrites and reports the rest;
  it never deletes or modifies your store. `--protect-tag` rows are left alone, `--publish-tag`
  rows get `[bracketed internal text]` stripped, `--cleaned-out` writes the survivors to a new
  file. Exit 1 when there is anything to do.
- `answergrade reported-errors` (and `answergrade.reported`): reads conversations for visitors saying
  something you publish is wrong, groups them by subject, and matches each subject to the
  most-cited sources from `Citations`, so a bad fact quoted to everyone ranks first. A weak phrase
  ("is wrong") counts only within a clause of a word naming something published; a mail robot is
  never named as the reporter. Reads the same chat logs as `answergrade questions`.
- README: the opening is now a one-sentence description and a three-line example, and every
  command runs from the repository folder.
- README correction: a judge verdict with `"flag": false` and a category of the model's own
  invention (`{"flag": false, "category": "banana"}`) is graded as a pass, not as "could not
  grade". The 1.0.x text said an invented category was always "could not grade". The code is
  unchanged; the text now says what the code does, and a test pins it. An invented category is
  "could not grade" only when the flag is missing or true.
- Changelog: the 1.0.1 entry no longer names the publisher.

## 1.0.2

Documentation and packaging only. The code is unchanged.

- The README now opens with a one-line description and a three-line example you can paste,
  and the licence and access notes moved to the end.
- The package version (`pip show answergrade`, `answergrade.__version__`) now matches the release tag.
  Release 1.0.1 reported 1.0.0.
- The README no longer says the judge "will not invent" problems or "cannot invent" them. What the code
  covers is narrower: a reply that is not JSON, a failed or cut-off judge call, and a verdict that
  contradicts itself are "could not grade", never a flag. A verdict that is well formed but wrong still
  counts as a flag, so read the flagged answers; `evidence_verified` tells you whether the judge's quote
  appears in your sources.

## 1.0.1

Licence and contact details only. The code is unchanged.

- Licence text updated (version 1.3).
- One public contact address in the licence and the README.

## 1.0.0

First release.

- `answergrade eval`: recall@1, recall@k and MRR for any scorer with a
  `score_chunks(chunks, query, limit)` signature; test cases from JSONL; save a result as a
  baseline and compare a change with it, exiting non-zero on a regression beyond `--max-drop`.
  Expected ids that name no chunk are reported instead of counting as misses.
- `answergrade coverage`: how many chunks really have a usable vector (present, numeric,
  non-zero, the common length), with the state `off`, `no_key`, `no_cache`, `broken`, `partial` or
  `working`.
- `grade()`: a conservative, JSON-only AI judge for one answer against its sources, with optional
  facts and earlier turns. Flags `contradiction`, `fabrication` and `fabricated_action`; an
  unparseable, failed or cut-off judge reply is "could not grade", never a flag.
- `answergrade probe` and `run_probe()`: ask a bot (HTTP endpoint or Python callable), grade each
  answer against the sources it returned or a retrieval step, and write a text, JSON or HTML
  report with fix cards.
- `answergrade questions`: build a probe question set from chat logs (greetings, quoted email
  history, task prompts and duplicates removed) or generate one from your sources.
- `Citations` and `Gaps` (`answergrade kb`): which sources are cited, when they were last hit,
  60 days of history, sources that have gone quiet, and unanswered questions recorded with emails,
  phone numbers, amounts and long digit runs scrubbed. Stored in a small pluggable store
  (`JsonFileStore`, `MemoryStore`, or your own).
- Judges: `openai_compatible()` for any chat-completions endpoint and `command()` for any program;
  the `answergrade demo` stand-in help bot and rule-based judge for trying it with no API key.
- `answergrade judge-check`: ten answers with known grades, run through the judge you chose.
- Judge endpoints that reject a default field can be handled with `--judge-max-tokens`,
  `--judge-max-tokens-param`, `--judge-no-temperature`, `--judge-no-json-mode` and
  `--judge-param`; a reasoning model's `<think>` block is ignored, and an ambiguous verdict is
  "could not grade", never a flag.
- Bots that cite by id, file name, URL or title are graded against the chunks they cited, looked
  up in `--sources`; a bare name is never shown to the judge as evidence.
- Scorers may return chunk texts, ids or dicts.
- Standard library only at run time; `httpx` is an optional extra.
