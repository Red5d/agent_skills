# AGENTS.md — agent_skills

Reference for coding agents working on this repo.

## What this is

A public collection of agent skills (SKILL.md folders) usable across agent
platforms. Everything here is world-readable and intended for general use.

## Repository structure

```
README.md            top-level listing of the skills in this repo — keep in sync
AGENTS.md            this file
LICENSE              MIT
skills/<name>/SKILL.md   one folder per skill
```

## Rules for agents

1. **No personal or sensitive information.** Nothing environment-specific to
   the author's setup belongs here: no hostnames, IPs, usernames, absolute
   personal paths, API keys, tokens, or internal URLs. Parameterize anything
   host-specific (environment variables, config values) and document them as
   user-supplied inputs.
2. **Generalize paths and assumptions.** Use placeholders (`$CLM_FT`,
   `<venv>`, `~/.cache/...`) instead of machine-specific locations, and note
   constraints generically ("a machine with enough RAM for the model") rather
   than referencing one specific machine.
3. **Never commit directly to `main`.** Work on a feature branch
   (`add-<skill-name>` / `update-<skill-name>`) and open a pull request.
   Open PRs only when the repo owner asks — agents may prepare branches and
   recommend changes otherwise.
4. **Keep the README in sync.** Every skill added or renamed must be listed
   in the top-level README.md with a one-line description.
5. **Keep the skill format.** Each skill folder contains a `SKILL.md` with
   YAML frontmatter (`name`, `description`, `version`) and a markdown body.
   `description` should state the trigger: "Use when …" followed by the
   one-line behavior.
6. **Scrub before committing.** Before opening a PR, scan the diff for
   usernames, absolute paths from the authoring machine, domains, keys, and
   story specifics that reveal private context. The test: could a stranger
   use this skill without learning anything private about the author?
