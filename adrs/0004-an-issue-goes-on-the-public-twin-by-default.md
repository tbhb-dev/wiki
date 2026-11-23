---
type: Decision Record
title: An issue goes on the public twin by default
description: For a repository with a public and a private twin, an issue goes on the public one unless its text needs private details, and a scan decides before it's created.
tags: [issues, public, private, scanning]
status: stable
decided: "2026-11-22"
---

# An issue goes on the public twin by default

## Context

Several repositories come in pairs: a public repository and a private twin whose name adds a `-private` suffix, such as the tools repository and its private twin. An issue about the shared work could go on either. Public by default keeps work visible, but an issue that leaks a host name or an internal path into a public repository is an incident.

## Decision

An issue goes on the public repository unless its text needs private information: host names, tailnet details, IP addresses, internal paths, account names or anything credential-shaped. Such an issue goes on the private twin. The project-management tool finds the twin by the `<name>-private` naming convention, reading the organization's repositories at run time rather than from a committed list. The content kernel scans the issue's exact text before creating it. On a finding, it refuses the public repository and offers the private twin. Nobody rewords text to get it past the scan.

Moving an existing issue between twins uses GitHub's issue transfer, after the same scan.

## Options considered

- Put every issue for a pair on the private twin: rejected because public work would become invisible for no reason.
- Rewrite private details out of the text automatically: rejected because rewording to pass a scan hides the leak instead of putting the issue where it belongs.

## Consequences

Most issues land in public repositories. The scan is the same single path the kernel uses for every write, with the private configuration generated for it. A repository's name alone doesn't send an issue to the private twin, because a later rule allows private repository names in public content. A miss by the scan is a leak into a public repository, which is handled as an incident and fixed at its cause.
