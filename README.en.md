# dsh-zh-language-policy

简体中文 | [English](README.en.md)

A host-side language contract plugin for DeepSeek Harness: before every model step it re-anchors two contracts at the very top of the system prompt — "English chain-of-thought, Simplified-Chinese user-facing prose" and "final summaries must be self-contained". Pure plugin, no core changes.

## The problem it solves

1. **Language drift**: after reading large English tool outputs, DeepSeek-class models tend to drift into English prose for user-visible text — the user literally reads English paragraphs in the conversation.
2. **Facts lost in summaries**: the UI collapses a turn's intermediate process rows behind the final summary by default, so users mostly read the summary; anything important they saw during the run but that never made it into the summary is effectively lost.

## Mechanism (grounded in source, not guesswork)

The harness assembles the system prompt from ordered sections contributed through `ctx.systemPrompt.section()`; the harness identity sits at order -100 and the deployment persona at 0. This plugin anchors its two contracts at **-200 / -190** — ahead of every other instruction — and the assembly re-runs **before every model step**, so the contracts are re-asserted each turn. That re-assertion is exactly the antidote to "tool results came back in English and the model never switched back".

Both contracts are short positive statements (they set expected behavior instead of fighting the model), written bilingually (the model reads each rule in the language it governs):

1. **Language contract** (order -200): internal reasoning and chain-of-thought in English; every user-visible text (answers, summaries, step narration, explanations) in Simplified Chinese; technical terms keep their original spelling but are wrapped in Chinese sentences; self-check before every visible sentence — when in doubt, emit Chinese.
2. **Summary completeness contract** (order -190): when a turn ends the UI auto-collapses the intermediate process rows, so the final summary must be self-contained — restate the request and what was done, include every important result, number, decision and reason, warning, failure and next step, list created/modified files with full paths in inline code, and restate anything the user saw or could have seen during the process.

The full bilingual contract texts live in `lib/index.js` as `CONTRACT` and `SUMMARY_CONTRACT`.

## Install

```bash
dsh plugin --profile web add -w dsh-zh-language-policy
```

Host plugins load at DSH startup — **fully quit and restart DeepSeek Harness after install**.

## Uninstall

Remove the `zh-language-policy` insert row from `cordis.patch.yml` plus the package directory, then restart.

## Compatibility and version sensitivity

- Tested on DeepSeek Harness `0.1.5-rc.1` (web profile, host side).
- **Note**: the contract `order` anchors (-200 / -190) are relative to harness-internal constants (identity at -100, persona at 0). After a DSH upgrade, if those internal constants move, adjust the order values in `lib/index.js` accordingly, otherwise the contracts may no longer sit at the very top of the prompt. After upgrading DSH, read the official `dsh-system-prompt` section orders first, then restart.

## How it differs from similar community plugins

Some language plugins force the chain-of-thought into Chinese as well; this one deliberately **keeps the chain-of-thought in English** (users never see it) and only forces Chinese for visible prose, while tying summary completeness to the collapse feature — two distinct governance goals.

## License

MIT
