# Prompts

Reusable operator briefings for autonomous offensive-security agents. Written to be pasted as the **first message** of a run: they set scope, force evidence discipline, and stop the loops that make agents slow.

Live companion: <https://av10v.github.io/CTF-BiotechCorp/#prompt>

## Index

| Prompt | Use it for | Size |
|---|---|---|
| [`prompts/01-autonomous-ad-ctf-agent.md`](prompts/01-autonomous-ad-ctf-agent.md) | Any AD / CTF lab. Lab-agnostic template: placeholders, 10 operating rules, 8 adaptive phases, decision loop, state files, time budgets, pitfalls, report format. | ~730 lines |
| [`prompts/02-biotechcorp-from-zero.md`](prompts/02-biotechcorp-from-zero.md) | A worked example for a 3-host AD lab — `IP1`/`IP2`/`IP3` are variables you fill in per run; from a single handed-over credential to Domain Admin, no solution leakage. | ~180 lines |

## What makes these different from "you are a pentester, go hack"

Three things, in order of impact:

**1. Calibration before exploitation.** Derive a Kerberos key convention once, on a credential whose answer you already know, then reuse it. A wrong string-to-key assumption burns an hour in `PREAUTH_FAILED` and looks exactly like a wrong password.

**2. Permissions before authentication.** The first AD action is `get writable` plus a full certificate-template dump — not a password guess. Multi-hop ACL chains are the norm and one sweep finds them.

**3. Stop conditions.** Objective verified, runtime cap hit, or three re-plans with no new evidence. Then stop and report the exact blocker. Without this an agent drifts for hours; with it, a stalled run still produces a usable partial report.

## Using them

1. Copy the prompt you want.
2. Fill every `[PLACEHOLDER]` — targets, authorized CIDR, initial credentials or `none`, objective, flag format, runtime cap, rate ceiling.
3. Paste as the first message; let it run.
4. Anything you left blank, the agent must treat as unknown and discover — say so explicitly if you want that enforced.

Adapt per lab. The rules and the reporting contract transfer; the phase content is where you tailor.

## Notes

- Prompt 02 is written for one specific (by-then-retired) training range, with `IP1`–`IP3` left as variables to fill per run and dead credentials; the methodology is the point.
- Both prompts assume the agent has a shell and can run the tools itself. If it only has a browser or a chat window, keep the structure but expect the agent to hand you commands to run.
- Nothing here authorises testing anything. Scope is what you fill into the placeholders.
