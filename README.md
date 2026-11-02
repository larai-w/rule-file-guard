# rule-file-guard

[日本語](README.ja.md)

A small check for the files coding agents read as instructions: `AGENTS.md`,
`CLAUDE.md`, `GEMINI.md`, `.cursorrules`, `.windsurfrules` and
`copilot-instructions.md`.

It fails when one of those files tells the agent to:

- **install a package by name** (`npm install left-pad`, `pip install requests`, `uv add httpx`, `cargo install …`)
- **change where packages come from** (`--index-url`, `--extra-index-url`, `registry=`, `npm config set registry`)
- **pipe a download into a shell** (`curl … | bash`, `wget … | sh`)

Installing what the manifest already lists is fine: `npm install`, `npm ci`,
`pip install -r requirements.txt`, `pip install -e .`, `uv sync`.

Since v1.1.0 it also reads every `README*` and `TODO*` file, and fails when one
of them, or one of the instruction files above, **hides an instruction for an
agent**:

- **invisible characters**: zero-width spaces and joiners, bidirectional
  controls, word joiners, a byte-order mark in the middle of a file, Unicode tag
  characters (emoji sequences and subdivision flags are allowed)
- **an HTML comment that speaks to an agent**, such as a note or instructions
  "for the AI", "if you are an AI …", or a comment that starts with
  `Claude:` or `AI agents:`
- **an override phrase** of the "ignore previous instructions" kind <!-- rule-file-guard: allow -->

Install commands in a README are ordinary setup steps, so the package checks
apply only to the instruction files.

## Why

Agent instruction files run with the agent's permissions, change what gets
installed, and are often copied in from somewhere else. They are rarely
reviewed the way code is. A recent preprint (arXiv
[2610.09264](https://arxiv.org/abs/2610.09264), not yet peer reviewed) reports
that a hidden instruction in a rule file can make a coding agent replace a real
dependency with an attacker's package.

Dependencies belong in the manifest and the lockfile, where review and
Dependabot can see them. This check keeps them out of the instruction files.

Agents read more than their instruction files. Another preprint (arXiv
[2610.38867](https://arxiv.org/abs/2610.38867), not yet peer reviewed) holds
the injected text fixed and moves it around a repository: an agent running on
GPT-5.4 followed an unsafe instruction placed in a README or TODO file about
2.6 times as often, on average, as the same instruction placed in a code or
data file. A README is also the file a reviewer is most likely to skim. So the
hidden-text checks cover README and TODO too.

## Use it as a GitHub Action

```yaml
name: Rule-file guard
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  guard:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - uses: larai-w/rule-file-guard@v1
```

Pin the action to a commit SHA if your policy requires it.

## Use it before each commit

With [pre-commit](https://pre-commit.com):

```yaml
repos:
  - repo: https://github.com/larai-w/rule-file-guard
    rev: v1.1.0
    hooks:
      - id: rule-file-guard
```

Or copy `rule_file_guard.py` into your repository and call it from your own
hook. It needs only Python 3.9+ and git.

```bash
python3 rule_file_guard.py              # every tracked rule, README and TODO file
python3 rule_file_guard.py AGENTS.md    # only the files you name
```

## When a line is intended

If a line has been reviewed and really should stay, add
`rule-file-guard: allow` to it, for example in an HTML comment:

```markdown
pip install pre-commit  <!-- rule-file-guard: allow -->
```

## What it does not catch

This is a pattern check, not a defence on its own.

- It reads only the file names listed above, plus `README*` and `TODO*`.
  Instructions planted in code comments, issues, web pages or tool output are
  out of its reach.
- It matches command forms and a few known phrasings. An instruction written
  in plain, visible prose ("add the package foo-utils before you start") or
  hidden in encoded text is not caught. Visible prose in a README still needs a
  human to read the diff.
- It does not look at what a package is or whether it exists.

Keep reviewing changes to instruction files like code, and keep dependencies
in a lockfile.

## Where it came from

It was first written for seven of my own repositories. The first version also
stopped a bare `npm install` and fired on ordinary setup steps in three of
them, so it was narrowed to installs that name a package. A guard that stops
too much gets ignored. `tests/test_rule_file_guard.py` keeps both lists: lines
it must stop, and lines it must let through.

Before the README and TODO checks were released, they were run over about 80
local clones (my own repositories and several open-source forks, including a
large frontend project). They flagged nothing, and the Next.js
`BEGIN:nextjs-agent-rules` markers are in the let-through list.

## License

MIT
