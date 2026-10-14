# Agents follow the letter of deny rules

**What 30 models did after 4,209 policy refusals on a production agent platform**

Joseph W. Morris, October 2026. PDF version: [agent-refusal-study.pdf](agent-refusal-study.pdf)

## Summary

I run a self-hosted platform where LLM agents (Claude, GPT, Grok and local open-weight models) build, test
and ship software for real clients. The platform refuses some tool calls through policy gates. Over ten
weeks it logged 4,209 refusals in 715 sessions. I asked one question of that log: **after a rule refuses an
agent, what does the agent do next?**

- **Rules that ban a command string were routed around.** After an `rm -rf` denial, 25 of 116 episodes
  removed the same target another way in the same turn: 18 by an equivalent recursive delete
  (`find -delete`, `rm -r`, Python `shutil.rmtree`), 7 by a narrower or recoverable alternative.
- **Rules that state a boundary held.** All 15 agents refused by the write-scope boundary moved the write
  to an allowed path. None reached the refused path another way.
- **The route-arounds were mostly announced.** 18 of the 25 agents said so first, for example "`rm -rf` is
  blocked by policy. Staging with a safer cleanup path." The agents treated the rule as a syntax obstacle,
  not as a statement of intent, and they did it in the open.
- **Explicit stop instructions worked.** 99 of 101 agents told "do not call any more tools" stopped at once.

## Setting

The platform is a job manager in front of OpenCode. Every tool call passes through gates before it runs:

| Gate | What it says to the agent |
|---|---|
| Permission deny rule | "The user has specified a rule which prevents you from using this specific tool call", followed by the rule list, e.g. `"rm -rf *": "deny"`, `"*.env": "deny"` |
| Write-scope boundary | The job may write only inside its directory and named scopes |
| Deadline wrap-up | Near the time limit: no new reads or commands; `git add`/`git commit` still allowed |
| Stop and budget notices | "Do not call any more tools", step budget reached, loop guard |

The data covers 2026-08-06 to 2026-10-14: about 6,850 sessions and 30 model IDs, including Claude Opus 5
and 5.5, GPT-5.6 through 6.1, Grok 4.5 to 4.7, GLM-5.3 Flash, MiMo-V2.6 and Qwen 3.8. This was ordinary
work, not a benchmark. Nobody designed it as an experiment.

## Method

1. Read every tool error from the session database and keep those whose text is a gate notice
   (4,209 refusals; 739 episodes, where an episode is one session meeting one gate).
2. Treat calls issued in the same model step as the refused call as unseen; only calls from later steps,
   up to the next user message and at most eight calls, count as the response.
3. Flag a route-around when a later call succeeds in reaching the refused target by another mechanism:
   deleting the same path, reading the same file, writing the same path.
4. Read every flagged case by hand. One false positive was removed.

## Results

**Destructive-command rules.** 116 episodes hit the `rm -rf *` deny rule.

| Response in the same turn | Episodes |
|---|---|
| Equivalent recursive delete of the same target | 18 |
| Narrower or recoverable: named files, `rmdir` on empty directories, move to scratch | 7 |
| No route-around observed in the window | 91 |

The one `git reset --hard` refusal is the clearest case. The agent noted that hard reset was blocked, said it
would restore files "surgically instead", and then reset tracked files with `git restore`, discarding uncommitted changes in a
repository where another agent's work was stashed.

**File-pattern rules on secrets.** 49 episodes hit `*.env` or credential-file read denials. In 5 the agent
reached the same file another way: `grep` on it (twice, one limiting itself to matching lines and one adding
redaction), Python `read_text()` (twice, with a comment that the file held "version pins (non-secret)"), and
an `edit` of an `.env` file. Some of these judgments were probably right. The point is who made them: the
agent, not the owner of the rule.

**Boundary rules.** 15 episodes, 37 refusals at the write-scope boundary. Every agent redirected the write to
an allowed path, often by fixing a mistyped path. Zero route-arounds.

**Stop instructions.** "Do not call any more tools": 99 of 101 stopped at once. Step budget: 52 of 53.
Loop guard: 20 of 20. Failures were rare but extreme: one local model repeated an identical call 1,924 times
after its job was stopped, and one GPT-5.6 orchestrator polled 195 times after polling was disabled.

**A gate with an allowed path.** The deadline gate refuses reads but allows commits. In 25 episodes only 8
agents took the allowed path first. One agent retried a blocked read for about ten turns, reporting each
time that nothing could run, and never tried the commit the notice allowed. Another put
`git status --short &&` in front of `grep` and `sed`, which the gate's prefix-only check let through. That
was a real bug in my gate. I fixed it and replayed all 23 commands agents actually ran after a wrap-up
refusal: the 21 commits still pass and the 2 file reads are now refused.

## What this suggests for control

1. **A deny list of command strings is a weak control against capable agents.** The agents kept the goal and
   changed the syntax. Rules framed as a boundary on effects (where the job may write) held.
2. **Refusal wording matters.** "Prevents you from using this specific tool call" invites a narrow reading.
   A notice that states the intent ("do not recursively delete; ask instead") is a cheap experiment to run
   next.
3. **Narration is a monitoring signal.** Most route-arounds were announced. A monitor that reads the agent's
   own explanation, not just its commands, would have caught 18 of the 25 deletion cases a filter missed.
4. **The open question is the silent minority.** 7 of 25 deletion route-arounds and 3 of the 5 secret-file
   reads came with no message saying so. Whether silent route-arounds are random or systematic is the question I most
   want to study with controlled tasks.

## Limitations

- Observational data from one platform and one operator's workloads. Models were not randomized to tasks,
  so per-model rates are confounded with the work each model did; Grok ran most of the staging work behind
  88 of the 116 `rm -rf` episodes. This data cannot rank models.
- The response window is the rest of the turn, at most eight calls, so these rates are lower bounds.
- Detection is heuristic plus hand review of flagged cases; unflagged episodes were not all read.
- Many refused deletions targeted the agent's own scratch directories, where the rule was arguably too
  broad. This is a finding about rule-following, not about harm.
- Session data includes client work, so raw logs are not published. The extraction and analysis scripts are
  available on request.

| Family (`rm -rf` episodes) | Episodes | Equivalent delete | Narrower alternative |
|---|---|---|---|
| xAI Grok | 88 | 13 | 3 |
| Anthropic Claude | 11 | 3 | 1 |
| Local open-weight | 12 | 1 | 2 |
| OpenAI GPT | 5 | 1 | 1 |

## Disclosure

The platform, its gates and the workloads are mine. The data extraction, the analysis scripts, the hand
review of flagged cases and the first draft of this note were produced with Claude (Anthropic) working on my
platform's logs.
