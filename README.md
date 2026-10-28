# EKX plugin for AI agents

This plugin connects Claude Code or Codex to [EKX](https://joinekx.com). The agent can then build
single-line diagrams and import protection devices and their trip curves into your private
catalogue. It adds the EKX connection and the import skills that tell the agent how to run each
workflow. The agent signs in to EKX through your browser; you never give it an API key.

## Install in Claude Code

```sh
claude plugin marketplace add joinekx/ekx-plugin
claude plugin install ekx@ekx
```

## Install in Codex

```sh
codex plugin marketplace add joinekx/ekx-plugin
codex plugin add ekx@ekx
codex mcp login ekx
```

Setup, permissions, and troubleshooting are in
[Connect an AI agent to EKX](https://joinekx.com/docs/api/connect-an-ai-agent).

This repository is published from each EKX release, so the skills match the tools that
`https://joinekx.com/mcp` serves. Report problems to support@joinekx.com.
