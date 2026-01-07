# BEPs
Backend.AI Enhancement Proposals

> [!NOTE]
> This repository is now moved into the core mono-repo: https://github.com/lablup/backend.ai/tree/main/proposals

## Process

* Create a discussion thread in [the GitHub discussions group](https://github.com/lablup/beps/discussions/categories/beps).
* Create a new branch and pull request for creation.
  - Copy drafts/BEP-0000-template.md to start a new document in the drafts directory.
  - Write and submit the draft. 
* Discuss with other developers and maintainers.
* Submit multiple pull requests to modify and update your proposals.
* Once accepted, move the document into the designated LTS release directory (e.g., `/accepted/25.6`).
  - You may further submit additional pull requests to revise the document when there are changes required found during actual implementation work.

## Rules for PR title

Each PR either creates or updates the BEP documents with the squash-merge strategy.
Please put the high-level summary of the update in the PR title as they will become the commit message of the main branch.
Individual commits inside each PR may be freely titled.

Examples:

* "BEP-9999 (new): A shiny new feature" (with the document title)
* "BEP-9999 (update): Change the implementation plan" (with the summary of change)
* "BEP-9999 (update): Remove unnecessary API changes" (with the summary of change)
* "BEP-9999 (accept): Planned for 25.9 LTS" (with the target version)
* "BEP-9999 (reject): Decided to drop because ..." (with the short description of rejection)
* "BEP-9999 (revise): Update integration with ZZZ subsytem" (with the summary of revision)
