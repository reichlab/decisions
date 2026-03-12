# 2026-03-12 Claude Code Skills Management

## Context

The team has begun exploring [Claude Code skills](https://docs.anthropic.com/en/docs/claude-code/skills) as a way to encode multi-step workflows into guided, executable processes. So far, a single skill (`/hubverse-release`) has been created for releasing hubverse R packages. It currently lives in one team member's personal directory and is not available to anyone else.

The [dashboard release and deployment RFC](./2026-03-10-RFC-dashboard-release-deployment.md) proposes five additional skills (`/dashboard-local-build`, `/dashboard-release`, `/dashboard-stage`, `/dashboard-debug`, `/dashboard-config-migrate`) to address knowledge concentration and manual deployment complexity. Several of these skills would need to be invoked from different repositories and coordinate work across multiple repos.

As the team begins building more skills, we need to decide how to store, version control, and share them so that:

- Team members can discover and use skills created by others
- Skills are version controlled and changes are reviewable
- Skills that work across multiple repositories have a sensible home
- The approach is accessible to a team that is new to Claude Code skills

### Definitions

The following Claude Code concepts are relevant to this decision:

**Skill**: A custom slash command for Claude Code that extends what Claude can do in a project. Skills can serve several purposes: guiding multi-step workflows (e.g., release processes, deployment sequences), providing domain knowledge or coding conventions that Claude applies automatically, supplying code generation templates for consistent output, or injecting live context by running shell commands. Each skill is a directory containing a `SKILL.md` file (described below) and optionally supporting files such as reference material, templates, or scripts. For example, `/hubverse-release` is a skill that guides a developer through the full R package release process. A skill directory might look like:

```
dashboard-release/
├── SKILL.md            # Required: defines the skill
├── reference.md        # Optional: detailed reference material
└── scripts/
    └── helper.sh       # Optional: scripts Claude can execute
```

**SKILL.md**: The file that defines a skill's behaviour. It contains YAML frontmatter between `---` markers followed by markdown instructions that Claude reads and follows step by step when the skill is invoked. The frontmatter fields include:

- `name` — the skill's name (used as the slash command, e.g., `hubverse-release` for `/hubverse-release`)
- `description` — a short description (Claude uses this to decide when the skill is relevant)
- `disable-model-invocation` — if `true`, the skill can only be invoked manually by the user, not automatically by Claude
- `argument-hint` — hint text shown to the user for expected arguments (e.g., `"[major|minor|patch]"`)

**Skill scope**: Skills can be stored at different locations, which determines who can access them:

| Scope | Location | Who has access |
|-------|----------|---------------|
| **Personal** | `~/.claude/skills/<name>/SKILL.md` | Only the user who created it, across all their projects |
| **Project** | `<repo>/.claude/skills/<name>/SKILL.md` | Anyone working in that repo (shared via git) |
| **Plugin** | Packaged in a plugin repository | Anyone who installs the plugin |

**Plugin**: A packaged collection of skills (and optionally other Claude Code extensions) distributed as a git repository with a `.claude-plugin/plugin.json` manifest. Plugins are loaded with `claude --plugin-dir /path/to/plugin` or installed from a marketplace. Skills provided by a plugin are namespaced to avoid conflicts (e.g., `/hubverse:dashboard-release` rather than `/dashboard-release`).

### Aims

 - Establish a standard location for storing and version controlling shared Claude Code skills
 - Enable team members and collaborators to discover and use available skills
 - Support skills that operate across multiple repositories
 - Keep the approach simple and accessible to a team new to Claude Code skills
 - Ensure personal skills (unrelated to hubverse) can coexist without conflict

### Anti-Aims

 - Prescribe the content or design of individual skills
 - Require all Claude Code usage to go through skills
 - Build custom tooling around skill distribution

## Decision

### Shared skills repository

We will create a dedicated repository (e.g., `hubverse-org/hubverse-claude-skills`) to store shared skills under version control. The repository will contain a `skills/` directory with one subdirectory per skill:

```
hubverse-org/hubverse-claude-skills/
├── skills/
│   ├── hubverse-release/
│   │   └── SKILL.md
│   ├── dashboard-release/
│   │   ├── SKILL.md
│   │   └── reference.md
│   ├── dashboard-local-build/
│   │   └── SKILL.md
│   ├── dashboard-stage/
│   │   └── SKILL.md
│   ├── dashboard-debug/
│   │   └── SKILL.md
│   └── dashboard-config-migrate/
│       └── SKILL.md
└── README.md
```

This structure is deliberately compatible with both distribution approaches discussed below — the `skills/` directory layout is the same whether skills are symlinked individually or loaded as a plugin.

### Review and version control

Changes to shared skills will follow the same process as code changes: pull requests with review. Changes to existing skills that alter their behaviour should be reviewed by someone who uses that skill. We will not adopt formal semantic versioning for the repository initially — git history and PR review are sufficient at this stage.

### Documentation

The repository README will serve as the primary documentation, including: a catalogue of available skills with brief descriptions, and installation instructions for getting the skills onto your machine. Projects that depend on specific shared skills (e.g., the dashboard repos) should note this in their `CLAUDE.md` or contributing guide so that new contributors know to install them.

The [hubverse developer guide](https://docs.hubverse.io/en/latest/developer/index.html) should be updated to introduce Claude Code skills as part of the development workflow and direct contributors to the shared skills repository for the full catalogue and setup instructions.

### When to use which scope

| Situation | Where to put the skill |
|-----------|----------------------|
| Skill is specific to a single repo | `.claude/skills/` in that repo (committed to git) |
| Skill is used across repos or by the broader team | `hubverse-claude-skills` shared repo |
| Skill is personal, experimental, or unrelated to hubverse | `~/.claude/skills/` locally |

### Distribution approach: to be decided

How team members actually get shared skills onto their machines is the key open question. There are two viable approaches, and this RFC presents both for team discussion rather than prescribing one.

#### Option A: Symlinked personal skills (simpler setup, manual maintenance)

Team members clone the shared repository and create symlinks from their personal skills directory to individual skills in the repo:

```bash
# Clone the shared repo (one-time)
git clone git@github.com:hubverse-org/hubverse-claude-skills.git ~/hubverse-claude-skills

# Symlink individual skills into personal skills directory
ln -s ~/hubverse-claude-skills/skills/hubverse-release ~/.claude/skills/hubverse-release
ln -s ~/hubverse-claude-skills/skills/dashboard-release ~/.claude/skills/dashboard-release
# ... etc.
```

**Advantages:**

- No new concepts — skills appear as personal skills, invoked as `/hubverse-release` (no namespace prefix)
- Coexists naturally with unrelated personal skills in `~/.claude/skills/` — each skill is an independent symlink, nothing is overwritten
- Updates are automatic — since symlinks point into the cloned repo, running `git pull` updates all linked skills in place with no extra steps
- Team members choose which shared skills to install
- No plugin packaging overhead (no `plugin.json` manifest needed)

**Disadvantages:**

- Each team member must manually create symlinks for each skill they want
- When new skills are added to the shared repo, team members must manually add new symlinks
- Team members must remember to `git pull` the shared repo to get updates
- If a skill is renamed or removed in the shared repo, the symlink breaks silently
- A setup script could reduce the manual work but adds something to maintain

#### Option B: Plugin (more setup overhead, easier ongoing maintenance)

The shared repository is structured as a Claude Code plugin by adding a `.claude-plugin/plugin.json` manifest. Team members load it as a plugin:

```bash
# Clone the shared repo (one-time)
git clone git@github.com:hubverse-org/hubverse-claude-skills.git ~/hubverse-claude-skills

# Load as a plugin (per session)
claude --plugin-dir ~/hubverse-claude-skills

# Or configure for persistent loading in Claude Code settings
```

**Advantages:**

- All skills in the repo are available at once — no per-skill setup
- New skills added to the repo are automatically available after `git pull`
- Removed or renamed skills are handled cleanly
- The plugin manifest provides a standard place for metadata (description, version)
- Opens a path to publishing on the Anthropic plugin marketplace in future if desired

**Disadvantages:**

- Skills are namespaced (e.g., `/hubverse:hubverse-release` instead of `/hubverse-release`), which is more to type — though namespacing could be used to reduce prefixing in skill names (e.g., `hubverse-release` could become just `/hubverse:release`)
- Requires understanding the plugin concept, which is an additional layer for a team new to Claude Code
- Requires either passing `--plugin-dir` each session or configuring persistent plugin loading
- Adds a `plugin.json` manifest file to maintain (though it is minimal)
- Personal skills in `~/.claude/skills/` coexist without issue, but the two sets are invoked differently (personal skills without namespace, plugin skills with namespace)

#### Migration path from A to B

The shared repository structure is designed so that moving from Option A to Option B requires minimal changes:

1. Add a `.claude-plugin/plugin.json` manifest to the repo (a small JSON file with name, description, and version)
2. Review skill directory names and remove redundant prefixes that are now covered by the plugin namespace (e.g., rename `hubverse-release/` to `release/` so the command becomes `/hubverse:release` rather than `/hubverse:hubverse-release`)
3. Team members remove their symlinks from `~/.claude/skills/`
3. Team members configure the repo as a plugin instead

The `skills/` directory and `SKILL.md` files remain identical in both approaches. No skills need rewriting. The main adjustment for users is that skill invocation gains the namespace prefix (e.g., `/hubverse-release` becomes `/hubverse:hubverse-release`).

### Other Options Considered

1. **Personal skills only (`~/.claude/skills/`), no shared repo.** Each team member maintains their own copy of skills. Not chosen because there is no version control, no review process, and skills diverge between team members over time. Onboarding requires manually copying files. This is the current state for `/hubverse-release` and it does not scale even to a small team.

2. **Project-level skills duplicated in each repo.** Place skills in `.claude/skills/` within each repository that needs them. Not chosen as the primary approach because many skills (e.g., `/hubverse-release`, `/dashboard-release`) are used across multiple repos. Duplicating skills across repos creates maintenance burden and version drift. However, project-level skills remain appropriate for genuinely single-repo workflows.

3. **Project-level skills in one "home" repo.** Place cross-project skills in `.claude/skills/` within one designated repo (e.g., the control room for dashboard skills). Not chosen because skills are only auto-discovered when working in that specific repo, it conflates the skill's lifecycle with the host repo's, and it does not handle skills that span project boundaries (like `/hubverse-release` which applies to any R package repo).

4. **Publish to the Anthropic plugin marketplace.** Package skills as a plugin and publish for maximum discoverability. Not chosen as the initial approach because the overhead is not justified for a small team just getting started, and some skills encode internal processes. Could be revisited for skills with wider applicability once the team is more experienced.

### Skill testing: a known gap

There is currently no official test framework, test runner, or dry-run mode for Claude Code skills. Skills are markdown instructions interpreted by an LLM, not deterministic code, which makes traditional automated testing fundamentally difficult. The available options are limited:

- **Manual testing**: Invoke the skill interactively and verify it behaves as expected. This is currently the primary method.
- **Code review**: Since skills are markdown files, careful review of the instructions can catch logical errors, missing steps, dangerous operations, or ambiguous wording. This is our most important quality gate.
- **Hooks**: Claude Code's `PreToolUse` and `PostToolUse` hooks can validate or block actions at runtime (e.g., preventing a skill from pushing to a protected branch), but this tests guardrails rather than the skill's logic.
- **Headless mode**: The `claude -p` flag runs Claude non-interactively, but it does not support invoking skills by slash command name, limiting its usefulness for CI-based skill testing.

This is a significant gap. Skills that automate multi-step processes with real side effects (creating branches, pushing code, making releases) carry risk if the instructions are wrong or ambiguous. Until better tooling exists, we will mitigate this through:

1. **Thorough code review**: All new skills and changes to existing skills require careful review, treating skill instructions with the same scrutiny as production code. Reviewers should walk through the instructions mentally (or actually invoke the skill) to verify correctness.
2. **Incremental development**: Start with simpler skills and build complexity gradually as the team gains experience.
3. **Confirmation gates**: Skills that perform irreversible actions (tagging, pushing, releasing) should include explicit pause points that ask the user to confirm before proceeding.
4. **Lessons learned**: As the team uses skills in practice, we will document failure modes and incorporate what we learn into skill design guidelines. The shared skills repository is a natural place for this documentation.

## Status

Proposed

## Consequences

Positive:

- Shared skills are version controlled and changes are reviewable
- Team members can discover available skills through the repository README
- Cross-project skills have a natural home that is not tied to any single repo
- The repository structure supports either distribution approach and migration between them
- Personal skills unrelated to hubverse are unaffected regardless of distribution choice

Negative:

- An additional repository to maintain
- Team members must clone the shared repo and perform setup (symlinks or plugin configuration)
- The distribution question requires a follow-up decision from the team
- No automated way to test skills — the team must rely on code review and manual testing, which requires discipline and will not catch all issues

Neutral:

- Project-specific skills remain in their respective repos unchanged
- Personal experimental skills can still live in `~/.claude/skills/`
- The decision does not affect how skills are authored — only where shared skills are stored and how they are distributed
- Skills proposed in the dashboard RFC still need to be written — this decision covers where they will live once created

## Projects

 - [The hubverse dashboard](../project-posters/hub-dashboard/hub-dashboard.md)
 - [Eval Metrics Expansion](../project-posters/eval-metrics-expansion/eval-metrics-expansion.md)
