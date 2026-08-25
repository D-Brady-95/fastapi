# Replay packet — 2026-08-25-powershell-path-context

Sensitivity: redacted

## Redactions applied
- Username/hostname redacted where present (if pasted).
- No secrets/tokens included.

## Task / prompt
Run a BOM-removal pass across specific files (sandbox/log_summariser + sessions/T1 + reviews/T1) by rewriting them as UTF-8 without BOM, then verify with pytest and commit.

## Context snapshot
- Hot file: .github/copilot-instructions.md (not present in this repo branch — limitation)
- Warm files: docs/context/stack.md (not present in this repo branch — limitation)
- Skills/rules: (none loaded by tooling; manual terminal-driven run)
- Spec / delta: reviews/T1/review.md drove the requirement (remove BOM blocker)
- Other context:
  - Repo root expected: C:\Users\DarrenBrady\fastapi
  - Affected files list: sandbox/log_summariser/*, sessions/T1/*, reviews/T1/review.md

## Model and client metadata
- Client: PowerShell + Git CLI (human-operated); Copilot not in the loop for the failing command execution.
- Model / version if visible: N/A (no model involved in execution); guidance originated from chat assistant (model identity not exposed in terminal).
- Date / time: 2026-08-25 (exact timestamps not captured — gap)
- Mode: supervised (human ran each command)

## Ordered action log
1. Operator ran BOM-removal loop using relative paths (.\sandbox\..., .\sessions\..., .\reviews\...) assuming working directory was repo root.
2. PowerShell resolved paths relative to C:\Windows\System32, producing DirectoryNotFoundException referencing C:\WINDOWS\system32\... paths.
3. Confusing output occurred: exceptions were thrown but subsequent "Rewrote without BOM: .\..." messages printed, creating ambiguity about whether files were modified.
4. Issue was caught by noticing error paths pointed at System32 rather than repo root.
5. Mitigation applied: re-ran BOM removal using absolute repo-root paths (C:\Users\DarrenBrady\fastapi\...), which succeeded and produced a small diff.
6. Verification gates executed: pytest on both sandbox tests passed after the corrected run.
7. Changes were committed.

## Output or offending portion
(Insert the exact PowerShell exception lines showing System32 path resolution here.)
(Insert the corrected absolute-path run confirmation here.)

## Triage note
- Failure mode: wrong working directory / relative path resolution caused file operations to target System32 paths, leading to exceptions and misleading success logs.
- Trigger: running a multi-line loop after context switches; assuming relative paths were anchored at repo root.
- How it was caught: exception referenced C:\WINDOWS\system32\...; recognized mismatch with intended repo root.
- Correction or mitigation: re-run using absolute paths rooted at repo directory; optionally cd explicitly before each critical batch; log pwd at start of scripts.
- Packet sufficient for triage: yes (assuming the exact exception snippet is included below).

### Happy-path reproduction evidence (captured after-the-fact)
== HAPPY PATH EVIDENCE ==
2026-08-25T15:49:41
repo=C:\Users\DarrenBrady\fastapi

== BOM rewrite output ==
Rewrote without BOM: C:\Users\DarrenBrady\fastapi\sandbox\log_summariser\README.md
Rewrote without BOM: C:\Users\DarrenBrady\fastapi\sandbox\log_summariser\app\main.py
Rewrote without BOM: C:\Users\DarrenBrady\fastapi\sandbox\log_summariser\tests\test_get_summary.py
Rewrote without BOM: C:\Users\DarrenBrady\fastapi\sandbox\log_summariser\tests\test_get_summary_independent.py
Rewrote without BOM: C:\Users\DarrenBrady\fastapi\sessions\T1\isolated-test-paste-packet.txt
Rewrote without BOM: C:\Users\DarrenBrady\fastapi\sessions\T1\session-log.md
Rewrote without BOM: C:\Users\DarrenBrady\fastapi\reviews\T1\review.md

== git diff --stat ==
 reviews/T1/review.md                       | 24 ++++++++++++------------
 sandbox/log_summariser/app/main.py         |  2 +-
 sessions/T1/isolated-test-paste-packet.txt |  4 ++--
 sessions/T1/session-log.md                 |  6 +++---
 4 files changed, 18 insertions(+), 18 deletions(-)

== pytest gates ==
python -m pytest -q sandbox/log_summariser/tests/test_get_summary.py
.                                                                        [100%]
1 passed in 0.33s

python -m pytest -q sandbox/log_summariser/tests/test_get_summary_independent.py
........                                                                 [100%]
8 passed in 0.34s


### Original failure output (gap)
Original failing console output showing System32 path resolution was not captured at the time and is not recoverable from this environment; only a post-hoc happy-path reproduction is included.
