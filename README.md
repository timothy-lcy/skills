# JifChat agent skill

This skill lets an agent generate images and video on the [JifChat](https://jif.dev) infinite canvas.

- [`SKILL.md`](SKILL.md) — how the agent calls the canvas
- [`examples/one-shot.md`](examples/one-shot.md) — the UGC shot to repeat a few times
- [`examples/kate-canvas.json`](examples/kate-canvas.json) — a sanitized full board, reference only. Do not import it as a project.

## Install

Copy `SKILL.md` and `examples/` together into the agent's skills directory. The skill reads the examples next to itself.

### Cursor

```bash
mkdir -p ~/.cursor/skills/jifchat
cp SKILL.md ~/.cursor/skills/jifchat/SKILL.md
cp -R examples ~/.cursor/skills/jifchat/examples
```

For one project only, use `.cursor/skills/jifchat/` in that repo.

### Claude Code

```bash
mkdir -p ~/.claude/skills/jifchat
cp SKILL.md ~/.claude/skills/jifchat/SKILL.md
cp -R examples ~/.claude/skills/jifchat/examples
```

For one project only, use `.claude/skills/jifchat/`.

### Hermes

```bash
mkdir -p ~/.hermes/skills/jifchat
cp SKILL.md ~/.hermes/skills/jifchat/SKILL.md
cp -R examples ~/.hermes/skills/jifchat/examples
```

### Grok

```bash
mkdir -p ~/.grok/skills/jifchat
cp SKILL.md ~/.grok/skills/jifchat/SKILL.md
cp -R examples ~/.grok/skills/jifchat/examples
```

If Grok Bot has no skills directory, give it this repository and tell it to follow `SKILL.md`, with `examples/` kept beside that file.

The agent will ask for a JifChat API key the first time it runs. Keep that key on your machine. Do not commit it.
