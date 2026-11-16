---
name: code-provenance
description: The single source of truth for how the studio labels the origin of every source file it produces — ai-generated, human-assisted, human-written, or legacy — the in-file header format, when each label applies, and how the manifest, telemetry, and gate work
type: prompt
when_to_use: Whenever an agent writes, edits, or reviews a source file in a studio project, and whenever provenance is reported (app-status, dashboard) or gated (code-reviewer, release)
whenToUse: Whenever an agent writes, edits, or reviews a source file in a studio project, and whenever provenance is reported or gated
---

# Code provenance

Every source file the studio produces carries a **provenance tag** that records who authored it.
This lets the team — and the user — see at a glance how much of the app was written by AI, shaped
together, written by a human, or carried in from elsewhere. The tag lives in the file; everything
else (the manifest, the dashboard, the gate) is rolled up from it.

## The four classes

| Class | Meaning | Applies when |
| --- | --- | --- |
| `ai-generated` | Written start to finish by a studio agent, with no human editing of the content. | The agent created the file, or rewrote it wholesale, and a human only approved — never edited — the result. |
| `human-assisted` | AI and a human both shaped the content. | The agent edited a file a human wrote, **or** a human edited a file the agent wrote. Any genuine mix. This is the correct label whenever authorship is shared. |
| `human-written` | Authored entirely by a human, with no AI-generated content. | A person wrote the file by hand. An agent must never set this on its own output. |
| `legacy` | Pre-existing, imported, vendored, or tool-scaffolded code brought in as-is. | Xcode/SwiftPM template files, third-party sources copied in, code migrated from another project. Origin is not this studio run. |

Rule of thumb for an agent: **what you create is `ai-generated`; what you edit that someone else
authored becomes `human-assisted`; code you merely pull in unchanged is `legacy`.** Never label
your own work `human-written`.

## The in-file tag (source of truth)

Put a single comment line in the **first 10 lines** of the file:

```swift
// Provenance: ai-generated
```

- One of the four class tokens exactly, lowercase, after `Provenance:`.
- Swift, JS/TS, Kotlin, C/ObjC use `//`; shell/Ruby/Python use `#`. The tag goes below a shebang if present.
- It is the source of truth. The manifest and dashboard are regenerated from it and never override it.

The studio ships `scripts/provenance.mjs` to read, roll up, stamp, and gate these tags — prefer it
over editing the manifest by hand.

```
node scripts/provenance.mjs scan            # print the current breakdown (no writes)
node scripts/provenance.mjs write           # rebuild docs/80-provenance.{md,json} from the tags
node scripts/provenance.mjs set <file> ai-generated   # stamp/replace one file's tag, then refresh the manifest
node scripts/provenance.mjs check           # exit non-zero if any source file is untagged (gate/CI)
```

Run the script from the project root; it skips build output, dependencies, `docs/`, `assets/`, and
`design/`.

## The manifest (`docs/80-provenance.md` + `.json`)

`scripts/provenance.mjs write` regenerates both:
- `docs/80-provenance.md` — a human-readable summary table (files and lines per class) plus a
  per-file list. This is the doc the standup and humans read.
- `docs/80-provenance.json` — the machine-readable rollup the dashboard reads.

They are generated artifacts: never hand-edit them; change a file's tag and rerun `write`.

## Telemetry

When an agent sets or changes a file's class, emit one `provenance` event (see Telemetry in the
`studio-pipeline` skill) in the same step:

```
printf '%s\n' "{\"ts\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"event\":\"provenance\",\"file\":\"Sources/Onboarding/OnboardingView.swift\",\"class\":\"ai-generated\",\"agent\":\"ios-developer\"}" >> docs/90-telemetry.jsonl
```

## The gate

An untagged source file is a review finding, not a hard stop for the developer — but the
`code-reviewer` runs `node scripts/provenance.mjs check` and will not approve a ticket whose changed
source files lack a valid tag. Keep tagging part of writing the file, not a cleanup pass.
