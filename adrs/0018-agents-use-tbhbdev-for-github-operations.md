---
type: Decision Record
title: Agents use tbhbdev for GitHub operations
description: Agents use dedicated tbhbdev commands for GitHub reads and writes, and shared checks validate content before posting.
tags: [agents, github, tooling]
status: stable
decided: "2026-11-11"
---

# Agents use tbhbdev for GitHub operations

## Context

Agents, meaning automated contributors, need consistent rules for how they identify themselves and what they post to GitHub. They need a supported way to read pull requests, proposed changes and check results without a separate client. The rule covers GitHub calls initiated by agents and excludes calls initiated by people.

`tbhbdev` is the project's command-line tool for agent operations. `gh` is GitHub's command-line tool. The GitHub application programming interface (API) lets programs access GitHub directly. The harness is the software that runs the agent. The command guard checks which commands the agent may run. CUE is a language for defining data constraints.

## Decision

Agents use a dedicated `tbhbdev` command for each supported GitHub read or write. Harness rules block direct `gh` and GitHub API calls. The GitHub command guard denies writes outside the supported commands for pull requests, issues and project boards.

Commands that write to GitHub use `tbhbdev.content`, the shared module that builds posts from the applicable template. It checks the exact outgoing text for disclosures of private information, violations of writing rules and direct quotations of the operator (the person overseeing agent work). CUE schemas specify and validate the required structure of the content and its metadata. A failed check stops the write and reports the finding for correction.

## Consequences

Each new agent operation needs a supported command. Changes to the command guard go through a reviewed pull request with tests for the affected commands. Existing callers of the interim `tbhbdev alpha` subcommands use a compatibility alias until they move to commands directly under `tbhbdev`.
