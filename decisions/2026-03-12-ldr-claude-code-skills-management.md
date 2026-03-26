# 2026-03-12 Claude Code Skills Management

## Context

The team has begun exploring [Claude Code skills](https://docs.anthropic.com/en/docs/claude-code/skills) as a way to encode multi-step workflows into guided, executable processes. So far, a single skill (`/hubverse-release`) has been created for releasing hubverse R packages. It currently lives in one team member's personal directory and is not available to anyone else.

The [dashboard release and deployment RFC](./2026-03-10-RFC-dashboard-release-deployment.md) proposes five additional skills (`/dashboard-local-build`, `/dashboard-release`, `/dashboard-stage`, `/dashboard-debug`, `/dashboard-config-migrate`) to address knowledge concentration and manual deployment complexity. Several of these skills would need to be invoked from different repositories and coordinate work across multiple repos.

As more skills are created, we need a way to store, version control, and share them so that team members can discover and use skills created by others, changes are reviewable, and cross-repo skills have a sensible home.

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
|-------|----------|----------------|
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

### Distribution: symlinked personal skills

Team members will clone the shared repository and create symlinks from their personal skills directory to individual skills in the repo:

```bash
# Clone the shared repo (one-time)
git clone git@github.com:hubverse-org/hubverse-claude-skills.git ~/hubverse-claude-skills

# Symlink individual skills into personal skills directory
ln -s ~/hubverse-claude-skills/skills/hubverse-release ~/.claude/skills/hubverse-release
ln -s ~/hubverse-claude-skills/skills/dashboard-release ~/.claude/skills/dashboard-release
# ... etc.
```

We chose this approach because it introduces no new concepts beyond what the team already knows — skills appear as personal skills and are invoked directly (e.g., `/hubverse-release`). It coexists naturally with unrelated personal skills in `~/.claude/skills/`, and updates are straightforward: `git pull` in the shared repo updates all linked skills in place.

The trade-offs are acceptable at the current team size: each team member must manually create symlinks for new skills, remember to pull updates, and broken symlinks from renamed or removed skills will need manual cleanup.

Should the number of skills or team members grow to the point where this manual maintenance becomes burdensome, the repository structure is designed to support migration to a Claude Code plugin with minimal changes (adding a `.claude-plugin/plugin.json` manifest and replacing symlinks with plugin configuration). See the [migration path](#migration-path-to-plugin) section below.

### When to use which scope

| Situation | Where to put the skill |
|-----------|----------------------|
| Skill is specific to a single repo | `.claude/skills/` in that repo (committed to git) |
| Skill is used across repos or by the broader team | `hubverse-claude-skills` shared repo |
| Skill is personal, experimental, or unrelated to hubverse | `~/.claude/skills/` locally |

### Review and version control

Changes to shared skills will follow the same process as code changes: pull requests with review. Changes to existing skills that alter their behaviour should be reviewed by someone who uses that skill. We will not adopt formal semantic versioning for the repository initially — git history and PR review are sufficient at this stage.

### Documentation

The repository README will serve as the primary documentation, including: a catalogue of available skills with brief descriptions, and installation instructions for getting the skills onto your machine. Projects that depend on specific shared skills (e.g., the dashboard repos) should note this in their `CLAUDE.md` or contributing guide so that new contributors know to install them.

The [hubverse developer guide](https://docs.hubverse.io/en/latest/developer/index.html) should be updated to introduce Claude Code skills as part of the development workflow and direct contributors to the shared skills repository for the full catalogue and setup instructions.

### Migration path to plugin

If the symlink approach becomes unwieldy, the shared repository can be converted to a Claude Code plugin with minimal changes:

1. Add a `.claude-plugin/plugin.json` manifest to the repo (a small JSON file with name, description, and version)
2. Review skill directory names and remove redundant prefixes now covered by the plugin namespace (e.g., rename `hubverse-release/` to `release/` so the command becomes `/hubverse:release` rather than `/hubverse:hubverse-release`)
3. Team members remove their symlinks from `~/.claude/skills/`
4. Team members configure the repo as a plugin instead

The `skills/` directory and `SKILL.md` files remain identical in both approaches. No skills need rewriting. The main adjustment for users is that skill invocation gains a namespace prefix (e.g., `/hubverse-release` becomes `/hubverse:release`).

### Other Options Considered

1. **Personal skills only (`~/.claude/skills/`), no shared repo.** Each team member maintains their own copy of skills. Not chosen because there is no version control, no review process, and skills diverge between team members over time. Onboarding requires manually copying files. This is the current state for `/hubverse-release` and it does not scale even to a small team.

2. **Project-level skills duplicated in each repo.** Place skills in `.claude/skills/` within each repository that needs them. Not chosen as the primary approach because many skills (e.g., `/hubverse-release`, `/dashboard-release`) are used across multiple repos. Duplicating skills across repos creates maintenance burden and version drift. However, project-level skills remain appropriate for genuinely single-repo workflows.

3. **Project-level skills in one "home" repo.** Place cross-project skills in `.claude/skills/` within one designated repo (e.g., the control room for dashboard skills). Not chosen because skills are only auto-discovered when working in that specific repo, it conflates the skill's lifecycle with the host repo's, and it does not handle skills that span project boundaries (like `/hubverse-release` which applies to any R package repo).

4. **Plugin from the start.** Package skills as a plugin immediately. Not chosen because it introduces additional concepts (plugin manifests, namespacing, plugin loading configuration) that add friction for a team just getting started with Claude Code skills. The symlink approach lets us start simply while preserving a clear migration path.

### Skill testing: a known gap

There is currently no official test framework, test runner, or dry-run mode for Claude Code skills. Skills are markdown instructions interpreted by an LLM, not deterministic code, which makes traditional automated testing fundamentally difficult. The available options are limited:

- **Manual testing**: Invoke the skill interactively and verify it behaves as expected. This is currently the primary method.
- **Code review**: Since skills are markdown files, careful review of the instructions can catch logical errors, missing steps, dangerous operations, or ambiguous wording. This is our most important quality gate.
- **Hooks**: Claude Code's `PreToolUse` and `PostToolUse` hooks can validate or block actions at runtime (e.g., preventing a skill from pushing to a protected branch), but this tests guardrails rather than the skill's logic.
- **Headless mode**: The `claude -p` flag runs Claude non-interactively, but it does not support invoking skills by slash command name, limiting its usefulness for CI-based skill testing.

Until better tooling exists, we will mitigate this through:

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
- The symlink approach requires no new concepts beyond cloning a repo and creating links
- Personal skills unrelated to hubverse are unaffected
- A clear migration path to a plugin exists if the approach needs to scale

Negative:

- An additional repository to maintain
- Team members must clone the shared repo and create symlinks for each skill they want
- When new skills are added, team members must manually add new symlinks
- No automated way to test skills — the team must rely on code review and manual testing

Neutral:

- Project-specific skills remain in their respective repos unchanged
- Personal experimental skills can still live in `~/.claude/skills/`
- The decision does not affect how skills are authored — only where shared skills are stored and how they are distributed
- Skills proposed in the dashboard RFC still need to be written — this decision covers where they will live once created

## Projects

 - [The hubverse dashboard](../project-posters/hub-dashboard/hub-dashboard.md)
 - [Eval Metrics Expansion](../project-posters/eval-metrics-expansion/eval-metrics-expansion.md)
