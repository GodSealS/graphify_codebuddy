# graphify reference: commit hook and native CodeSquad integration

Load this when the user asked to install the post-commit hook or wire graphify into CodeSquad.

## For git commit hook

Install a post-commit hook that auto-rebuilds the graph after every commit. No background process needed - triggers once per commit, works with any editor.

```bash
graphify hook install    # install
graphify hook uninstall  # remove
graphify hook status     # check
```

After every `git commit`, the hook detects which code files changed (via `git diff HEAD~1`), re-runs AST extraction on those files, and rebuilds `graph.json` and `GRAPH_REPORT.md`. Doc/image changes are ignored by the hook - run `/graphify --update` manually for those.

If a post-commit hook already exists, graphify appends to it rather than replacing it.

---

## For native CodeSquad integration

Run once to make graphify always-on in CodeSquad sessions. Skill and AGENTS.md always share the same scope:

```bash
graphify codesquad install            # user-scope: ~/.codesquad/
graphify codesquad install --project  # this project: .codesquad/
```

User-scope writes the skill to `~/.codesquad/skills/graphify/` and a `## graphify` section to `~/.codesquad/AGENTS.md`. `--project` writes both under `.codesquad/` instead. The AGENTS.md section instructs CodeSquad to check the graph before answering codebase questions and rebuild it after code changes. No manual `/graphify` needed in future sessions.

```bash
graphify codesquad uninstall            # remove ~/.codesquad/ skill + AGENTS.md
graphify codesquad uninstall --project  # remove .codesquad/ skill + AGENTS.md
```
