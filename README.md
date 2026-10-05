# violet-skills

Agent skills from [VIOLINET Tech](https://github.com/Violinet-tech).
Each one is a folder with a `SKILL.md`, written from a real job and the mistakes
made on the way. They work in Claude Code and any harness that reads the
`SKILL.md` format.

## The skills

| Skill | Use it when |
|---|---|
| [`comfy-mcp-fixes`](https://github.com/Violinet-tech/comfy-mcp-fixes) | A ComfyUI custom node's UI won't show, `restart_comfyui` says `no_background_server`, or a comfy-cli server blocks Desktop from updating. |
| [`comfy-duplicate-model-roots`](https://github.com/Violinet-tech/comfy-duplicate-model-roots) | ComfyUI is slow to start, or a model/LoRA browser shows far fewer files than exist or has no previews. |
| [`seamless-loop-video`](https://github.com/Violinet-tech/seamless-loop-video) | You want a looping ambient video from a still (MiniMax H3 in ComfyUI), or a loop jumps or barely moves. Ships `make-loop.py`, `seamless.py`, `chain.py`. |
| [`ya-google-lee`](https://github.com/Violinet-tech/ya-google-lee) | You want to take an AI Studio / Firebase Studio export and make it run without Google APIs. |
| [`background-contrast`](https://github.com/Violinet-tech/background-contrast) | Text or panels over a photo/video background are washed out or unreadable. |
| [`repo-slim`](https://github.com/Violinet-tech/repo-slim) | A push warns about large files, or `.git` is huge, or build output was committed. |

Each skill also has its own repo (linked above) if you only want one. This repo bundles all six as a single plugin.

## Install

**As a Claude Code plugin (all skills):**
```
/plugin marketplace add Violinet-tech/violet-skills
/plugin install violet-skills@violet-skills
```

**One skill, by hand:** copy its folder into `~/.claude/skills/` (global) or
`.claude/skills/` (one project). The folder name must match the `name:` in its
frontmatter.

```bash
git clone https://github.com/Violinet-tech/violet-skills
cp -r violet-skills/skills/repo-slim ~/.claude/skills/
```

## Contributing

Issues and PRs welcome. Keep a skill generic: no personal paths, keys or
machine addresses.

## License

MIT, see [LICENSE](LICENSE).
