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

### `eds-conversion`

Guides an AdobeDocs content repository through a Gatsby-to-Edge Delivery Services conversion by:

- updating project configuration and template files;
- migrating and validating content;
- resolving lint and deprecated-block issues;
- coordinating local review and manual checkpoints; and
- managing stage and production deployment, redirects, and verification.

See [`skills/eds-conversion/SKILL.md`](skills/eds-conversion/SKILL.md) for the complete workflow.

## Installation

### Install with the skills CLI

Install either skill directly from the repository with [`skills`](https://skills.sh/):

```bash
pnpx skills add AdobeDocs/adp-devsite-cursor-rules --skill devsite-release
pnpx skills add AdobeDocs/adp-devsite-cursor-rules --skill eds-conversion
```

### Clone or copy

Clone this repository and configure your agent to discover the `skills/` directory:

```bash
git clone https://github.com/AdobeDocs/adp-devsite-cursor-rules.git
```

Alternatively, copy one or all skill directories into a skills location supported by your agent:

```bash
cp -R adp-devsite-cursor-rules/skills/<skill-name> <agent-skills-directory>/
cp -R adp-devsite-cursor-rules/skills/* <agent-skills-directory>/
```

### Symlink on Linux or macOS

You can symlink all skills into the shared agents directory. Using symlinks means updates to the cloned repository are immediately available to your agents:

```bash
cd adp-devsite-cursor-rules
mkdir -p "$HOME/.agents/skills"
for skill in "$(pwd)"/skills/*; do
  ln -s "$skill" "$HOME/.agents/skills/$(basename "$skill")"
done
```

Skill discovery locations and invocation syntax vary by agent. Consult your agent's documentation to confirm that it supports the `~/.agents/skills` directory or use its preferred skills directory instead.

## Usage

### Run a release

Ask the agent to prepare or run an adp-devsite EDS release. For example:

> Prepare the AdobeDocs/adp-devsite release from stage to main.

The agent should load `devsite-release` based on its description. This workflow creates or updates GitHub branches, files, and pull requests.

### Convert a content repository

Identify the content repository and ask the agent to start or continue its conversion. For example:

> Start the EDS conversion for adobe-assurance-public-apis.

The agent should load `eds-conversion`, track progress through the workflow, and pause at each manual checkpoint for confirmation.

Both skills can make repository or remote-service changes. Review their instructions before running them.

## Requirements

| Skill | Requirements |
|---|---|
| `devsite-release` | Bash, the [GitHub CLI](https://cli.github.com/) (`gh`), network access, GitHub authentication, and permission to access `AdobeDocs/adp-devsite` and `AdobeDocs/dev-docs-reference`. |
| `eds-conversion` | Bash, Git, Node.js and npm, network access, access to the relevant AdobeDocs repositories and deployment services, and a browser for manual verification steps. |

## Repository structure

```text
skills/
├── devsite-release/
│   └── SKILL.md
└── eds-conversion/
    └── SKILL.md
```
