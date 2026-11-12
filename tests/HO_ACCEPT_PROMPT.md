# HO_ACCEPT_PROMPT — приёмочный тест ролевой иерархии v2

**Как запускать:** создайте новую сессию и выберите пресет **`HO · Orchestrator`** (или сделайте его дефолтным в Settings → General). Именно в HO-сессии инструмент `subagent` имеет параметр `agent_preset`. Отправьте:

> Выполни приёмочный тест по файлу `harness-hierarchical-orchestrator/tests/HO_ACCEPT_PROMPT.md` —
> следуй шагам дословно, ничего не спрашивай, отчёт запиши в указанный файл.

---

## Сам тест (исполняется агентом)

You are running the acceptance test suite for the Hierarchical Orchestrator v2 configuration (call sign HO-ACCEPT). Execute the steps below exactly, in order, without asking the user any questions. Everything happens in the workspace directory. Keep replies minimal — the evidence goes into the report file, not prose.

STEP 0 — Environment: call `list_subagent_models` with no arguments, then with `provider="zai"`. Record the exact outputs. Also state whether your `subagent` tool schema includes an `agent_preset` parameter and enumerate its allowed values.

STEP 1 — Role chain (Orchestrator → Tech Lead → Explorer): use the `subagent` tool ONCE with `agent_preset="ho-tech-lead"`, description "HO-ACCEPT tech lead", and this prompt:

> You are the Tech Lead in acceptance test HO-ACCEPT. Your task_id is TASK-HO-01. Delegate ONCE with the subagent tool and agent_preset='ho-explorer', description 'HO-ACCEPT explorer', prompt: 'You are the Explorer in test HO-ACCEPT, task TASK-HO-01-E. (1) Read the files README.md and src/calc.js under harness-hierarchical-orchestrator/tests/sandbox/ and report their first line verbatim. (2) ATTEMPT to create a file named probe.txt in that directory using any tool you have. If no write-capable tool exists in your toolset, say NO-WRITE-TOOL-CONFIRMED instead. Report both results in three lines: E-LINE1 (README first line), E-LINE2 (calc.js first line), E-LINE3 (NO-WRITE-TOOL-CONFIRMED or what happened).' Wait for the explorer's completion notice, verify its evidence, then reply with: TL-LINE1 child_id_of_explorer=<id>; TL-LINE2 explorer evidence verbatim; TL-LINE3 TL-DONE.

Wait for the tech lead's completion notice and record its full report.

STEP 2 — Reviewer read-only chain: use the `subagent` tool ONCE with `agent_preset="ho-quality-lead"`, description "HO-ACCEPT quality lead", prompt:

> You are the Quality Lead in test HO-ACCEPT, task TASK-HO-02. Delegate ONCE with the subagent tool and agent_preset='ho-code-reviewer', description 'HO-ACCEPT reviewer', prompt: 'You are the Code Reviewer in test HO-ACCEPT, task TASK-HO-02-R. Read harness-hierarchical-orchestrator/tests/sandbox/src/calc.js and review the add() function for correctness in 2 sentences. Then ATTEMPT to modify any file; if no write-capable tool exists, include the token NO-WRITE-TOOL-CONFIRMED. Reply with: R-LINE1 verdict, R-LINE2 review, R-LINE3 NO-WRITE-TOOL-CONFIRMED or what happened.' Wait for the reviewer, verify, then reply with: QL-LINE1 child_id_of_reviewer=<id>; QL-LINE2 reviewer evidence verbatim; QL-LINE3 QL-DONE.

Wait for the quality lead's completion notice and record it.

STEP 3 — Native model route probe: use the `subagent` tool ONCE with `provider="zai"`, `model="glm-5.3"`, description "HO-ACCEPT route probe", prompt: `Reply with exactly one line: ROUTE-OK glm-5.3`. Record either the child id + reply, or the exact error text if the route is rejected. Do not retry more than once.

STEP 4 — Write the report: create the file `harness-hierarchical-orchestrator/tests/evidence/ho-accept-report.md` with sections: SESSION (your session id if known; else 'HO session'), STEP0 outputs verbatim, STEP1 tech lead child id + full report, STEP2 quality lead child id + full report, STEP3 probe result verbatim, and a final line `HO-ACCEPT-COMPLETE`. Then reply with exactly: `HO-ACCEPT-DONE`.
