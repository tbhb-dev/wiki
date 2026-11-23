---
type: Decision Record
title: 0040. Service and project management designs live in the public wiki
description: The public wiki is the home for the service-management and project-management designs.
tags: [designs, service-management, projects]
status: stable
decided: "2026-11-22"
---

# 0040. Service and project management designs live in the public wiki

## Context

The service-management and project-management designs explain how work moves across repositories. Readers need one place to find them and follow their changes.

This rule governs publication of the two designs and leaves the location of their implementations open.

## Decision

Maintain the [service-management design](https://github.com/tbhb-dev/wiki/blob/main/designs/itsm-records/README.md) and [project-management design](https://github.com/tbhb-dev/wiki/blob/main/designs/pm/README.md) in this public wiki repository. Identify private components by what they do. Check the designs against the [public-content rules](https://github.com/tbhb-dev/wiki/blob/main/AGENTS.md#standing-rules) before publication.

## Consequences

Both designs follow the wiki's pull request process. The public descriptions explain the workflows for readers outside the organization.
