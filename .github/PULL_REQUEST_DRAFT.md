# docs(adrs): add the work-tracking decision records

## Why

The decisions about how tbhb-dev tracks work, from issue priority to service-management records and content writes, exist only as dated decision entries. Distilling them into decision records gives each lasting rule one public, current statement under the adrs/ folder.

## Behavior change

Adds records 0001 to 0014 under `adrs/` and lists them in the README's Records section. The records span issue priority and placement, Requests, service-management workflows and the issue taxonomy, Changes and their approval, managed content writes and work notes, template and schema rules, ownership of decision records and review output, and the home of the two designs. Each states only the rule in force today, with live stopgaps in its Consequences. The REUSE annotation for records now uses a glob that REUSE supports, and the spelling list gains six terms.

## Decisions

tbhb decided each record. tbhb decided 0001 on 2026-11-11, and 0005, 0009, 0012 and 0013 on 2026-11-23. The rest carry 2026-11-22. Record 0001 was a reversible call under tbhb's delegation, made after tbhb asked for real priorities, and so was the list of kernel checks in 0011. Chose to drop the decision auto-merge rules for the records repository over recording them, because decision records moved to the wiki repositories.

## Deviations from the task

The branch also commits `.github/PULL_REQUEST_DRAFT.md` in its own last commit. This run had no push credential, and the run driver opens the pull request from the draft in the pushed commit. Drop that commit, or delete the file, before merge.

## Needs human judgment

Record 0004 says a private repository's name alone doesn't send an issue to the private twin, following the later rule that allows those names in public content, while the adrs/ README and this repository's agent instructions still forbid naming them here. Check that the two read consistently.

## Verification

```text
bazelisk test //...
Executed 20 out of 20 tests: 20 tests pass.
```

Before the REUSE glob change, `reuse_test` failed on every new record, because `adrs/[0-9]*.md` matched none of them.
