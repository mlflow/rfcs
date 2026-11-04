start_date:   # fill me in with today's date, YYYY-MM-DD
mlflow_issue: # link to the issue in the MLflow repo
rfc_pr:       # leave this empty

# Summary

Brief explanation of the feature.

# Basic example

If the proposal involves a change in API or UI flow, include a basic example.

## Motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

A well-written problem statement is the most important part of an RFC. A clear problem definition makes the rest of the RFC much easier to write and review.

### Out of scope

List any issues or requirements that this proposal will not address.

## Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with MLflow to understand, and for somebody familiar with the implementation to implement. This should get into specifics and corner cases, and include examples of how the feature is used.

If there are multiple viable approaches, use a decision matrix to compare them.

## Drawbacks

Why should we _not_ do this? Consider:

- Implementation cost, both in terms of code size and complexity
- Whether the proposed feature can be implemented in user space
- Impact on teaching people MLflow
- Integration with other existing and planned features
- Cost of migrating existing users (is it a breaking change?)

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Adoption strategy

If we implement this proposal, how will existing MLflow users adopt it? Is this a breaking change? Should we coordinate with other projects or libraries for promoting the change?

# Open questions

What parts of the design are still TBD?

# AI disclosure

List which parts of this RFC were generated or substantially drafted by an LLM (for example, "codebase survey in Detailed design", "first draft of Alternatives"), or write "None".

LLMs are great for searching the codebase and summarizing existing behavior, and we use them for this too. The purpose of an RFC, though, is to start a discussion, and that works best when authors can explain and defend each design decision themselves. RFCs that are mostly unreviewed LLM output may be closed by maintainers. See the [AI usage policy](https://github.com/mlflow/rfcs#ai-usage-policy).
