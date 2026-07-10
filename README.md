# fable-mode

Claude Fable 5's working discipline, written down by Fable 5 as a Claude Code skill — so a session on any model, especially Opus 4.8, works the way Fable worked.

A skill can't transfer raw capability, but it can transfer the shape of the work. Two files:

- **[SKILL.md](SKILL.md)** — the five-gate task loop (scope → evidence → adversarial reasoning → verification → calibrated reporting), the standing habits that run at every gate, and the smells that mean a gate got skipped.
- **[ROUTING.md](ROUTING.md)** — the delegation doctrine: when work should leave your context (four triggers; the default is no), which model tier gets it (mechanical / workhorse / peer), the delegation prompt contract, the escalation ladder, and the judge procedure for verifying anything a subagent claims.

## Install

```bash
mkdir -p ~/.claude/skills/fable-mode
cp SKILL.md ROUTING.md ~/.claude/skills/fable-mode/
```

## Use

Fires proactively on layered tasks — multiple dependent steps, debugging where the first theory might be wrong, orchestration across subagents, anything needing verification before handoff. Or invoke it directly: *"fable mode"*, *"route this"*, *"slow down and do this right"*.

Trivial work is exempt by design: forcing ceremony onto a one-file edit is its own failure mode.

## The method in one paragraph

Define done before touching anything. Open the real file instead of designing from memory. Before any behavior-changing edit, write the INTENT line (`code does X; check expects Y; spec says Z`) and resolve disagreements by authority order — user statement beats spec, spec beats tests, tests beat current code. Attack your own answer before committing to it; two failed attempts at the same fix means the diagnosis is wrong. Verify at the layer of the claim with evidence you didn't generate, and treat suspiciously clean results as broken checks. Report what you observed, verified separated from assumed. Delegate only when you can name the trigger — bulk, parallelism, fresh eyes, isolation — and route by spec completeness and cost-of-wrong, never by task size.

## License

MIT
