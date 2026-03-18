# MLflow RFCs

Many changes, including bug fixes and documentation improvements, can be implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these go through a lightweight design process to build consensus among the MLflow maintainer team.

The "RFC" (request for comments) process provides a consistent path for significant new features to enter the project.

## When to follow this process

You should consider writing an RFC if you intend to make "substantial" changes to MLflow. Examples that would benefit from an RFC:

- A new feature that creates new API surface area
- Removal or deprecation of features that already shipped
- Changes to core architecture or data models
- Introduction of new conventions or workflows that affect users broadly

Some changes do **not** require an RFC:

- Bug fixes and performance improvements
- Rephrasing, reorganizing, or refactoring
- Addition or removal of warnings
- Changes only visible to MLflow developers, not end users

## What the process is

1. Copy `0000-template.md` to `text/0000-my-feature.md` (where "my-feature" is descriptive — don't assign a number yet).
2. Fill in the RFC. Put care into the details: RFCs that do not present convincing motivation or are not honest about drawbacks tend to be poorly received.
3. Submit a pull request. The RFC will receive design feedback from the community and maintainers. Be prepared to revise it.
4. Build consensus and integrate feedback.
5. The maintainer team will decide whether the RFC is a candidate for inclusion. This may take time — we encourage you to seek community review first.
6. An RFC may be **accepted** (merged), **rejected** (closed with rationale), or **deferred** for a later time.

## AI usage policy

We encourage using AI tools to explore the MLflow codebase, draft sections, and polish writing. However, an RFC is a thinking exercise, not a generation exercise. Proposals that read as unreviewed AI output may be rejected without detailed feedback.

Before submitting, ask yourself: *Would I be comfortable defending every sentence in this document?*

## Implementing an RFC

The author of an RFC is not obligated to implement it. Once an RFC is accepted, anyone is welcome to submit an implementation as a pull request.

Being accepted is not a guarantee the feature will be merged — it means the team has agreed to it in principle.

## Reviewing RFCs

We read all RFCs but cannot always review them promptly. When you submit an RFC, your primary goal should be to generate a rich discussion. Even if a proposal isn't accepted as-is, the resulting conversation often shapes future work in the same area.

## Inspiration

This process is inspired by the [React RFC process](https://github.com/reactjs/rfcs).
