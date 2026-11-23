---
type: Decision Record
title: The service-management and project-management designs live in the public wiki
description: Designs go in the wiki repositories, not product repositories, and the service-management and project-management designs go in the public wiki under the public-repository rules.
tags: [designs, wiki, public]
status: stable
decided: "2026-11-22"
---

# The service-management and project-management designs live in the public wiki

## Context

The service-management and project-management designs started inside their own product repositories. Designs belong with the other designs in the wiki, where readers look for them and where design review applies. Most of both designs can be stated without anything private.

## Decision

Designs go in the wiki repositories, not in product repositories. The service-management and project-management designs live in this public wiki, in `designs/itsm-records/` and `designs/pm/`. The public-repository rules apply to them: no host names, tailnet details, IP addresses, internal paths, account names or credentials. In this repository, private repositories are described by role, such as the private ITSM repository and the central work repository. The outgoing scan and gitleaks pass before every push.

## Options considered

- Designs inside each product repository: rejected because designs belong in one place, the wiki.
- The private wiki: rejected because almost all of both designs is public-safe.

## Consequences

Parts of the designs that need private detail go in a private appendix with the same slug, and the public design says that one exists. A product repository removes any design folder it holds. Design folder names follow the same public-repository rules as their content.
