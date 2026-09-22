# Adobe Devsite Agent Skills

Reusable, agent-agnostic skills for Adobe Devsite workflows. Each skill follows the [Agent Skills specification](https://agentskills.io/specification), so it can be used by any AI agent that supports the standard.

## Available skills

### `devsite-release`

Prepares an AdobeDocs/adp-devsite EDS release by:

- comparing `stage` with `main`;
- collecting merged pull requests, approvers, and Jira tickets;
- generating release notes;
- updating the dev-docs-reference EDS changelog; and
- creating or updating the release pull requests.

See [`skills/devsite-release/SKILL.md`](skills/devsite-release/SKILL.md) for the complete workflow.

## Installation

Clone this repository and configure your agent to discover the `skills/` directory:

```bash
git clone https://github.com/AdobeDocs/adp-devsite-cursor-rules.git
```

Alternatively, copy the individual skill directory into a skills location supported by your agent:

```bash
cp -R adp-devsite-cursor-rules/skills/devsite-release <agent-skills-directory>/
```

On Linux and macOS, you can symlink the skill into the shared agents directory instead. Using a symlink means updates to the cloned repository are immediately available to your agents:

```bash
cd adp-devsite-cursor-rules
mkdir -p "$HOME/.agents/skills"
ln -s "$(pwd)/skills/devsite-release" "$HOME/.agents/skills/devsite-release"
```

Skill discovery locations and invocation syntax vary by agent. Consult your agent's documentation to confirm that it supports the `~/.agents/skills` directory or use its preferred skills directory instead.

## Usage

Ask the agent to prepare or run an adp-devsite EDS release. For example:

> Prepare the AdobeDocs/adp-devsite release from stage to main.

The agent should load `devsite-release` based on its description. The workflow creates or updates GitHub branches, files, and pull requests, so review the skill before running it.

## Requirements

The `devsite-release` skill requires:

- Bash;
- the [GitHub CLI](https://cli.github.com/) (`gh`);
- network access;
- authentication with GitHub; and
- permission to access `AdobeDocs/adp-devsite` and `AdobeDocs/dev-docs-reference`.

## Repository structure

```text
skills/
└── devsite-release/
    └── SKILL.md
```
