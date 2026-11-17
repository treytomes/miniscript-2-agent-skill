# miniscript-2-agent-skill

An agent skill for writing, reviewing, and debugging [MiniScript 2](https://github.com/JoeStrout/miniscript2)
(`.ms`) code.

## The skill file

**`SKILL.md` is the skill.** It is the only file here meant to be loaded into an
agent's context, and it is self-contained: syntax and scoping, lists/maps and
frozen values, functions and prototype classes, error values, the command-line
extras, and the bundled standard-library modules.

This repository is checked out into a skill folder by a git submodule, so
`README.md` and `LICENSE` land in that same folder next to `SKILL.md`. Neither is
part of the skill: they are repository documentation for humans, they carry no
skill instructions or agent directives, and a skill loader should skip them.

## Using it

`micro-expert` consumes this repository as a submodule at
`.agents/skills/miniscript-2`, so the skill lands at the path an agent already
looks in:

```
git clone --recurse-submodules git@github.com:treytomes/micro-expert.git
```

A clone made without `--recurse-submodules` leaves that folder empty; populate it
with `git submodule update --init`, or run `bootstrap.bat` at the project root.

Anywhere else, either add this repository as a submodule of
`.agents/skills/miniscript-2`, or copy `SKILL.md` into that path and keep it in
sync by hand.

## License

MIT — see [LICENSE](LICENSE).
