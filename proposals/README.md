# Carvel Proposals
This directory serves as the home for Carvel proposals. A proposal is a design
document that describes a new feature for a Carvel project. The new feature
can span multiple projects or introduce a new project. 
A proposal must be sponsored by at least one Carvel Maintainer.  Proposals can be worked by anyone in the community.

# Submit a Proposal
To create a proposal, submit a PR to this directory under the appropriate
project with a terse name (for example, `ytt/001-schemas/`). If the proposal concerns multiple projects or is
intended for the entire Carvel suite then please create the proposal at the root
of the `proposals` directory (for example, `./010-carvel-cli/`).

In that directory, create a `README.md` containing the core proposal. Include other files (e.g. sets of example artifacts, if necessary) to help support understanding of the feature.
When creating the proposal, please add a `Status: Draft` line at the top of the
proposal to indicate its state.

## Proposal Template
The below template is an example. Other than the high-level details (such as
title, proposal status, author, and approvers), please use whichever sections
make the most sense for your proposal.

```md
# <Proposal Title>
- Status: *Draft* | In Review | Accepted | Rejected
- Author(s): <your name>
- Approvers: <list of names that must sign off>

## Problem Statement
_This is a short summary of the problem that exists, why it needs to be
solved, and how this proposal will address this need. The goal of this section is to bring
readers up to speed quickly._

## Terminology / Concepts
_Define any terms or concepts that are used throughout this proposal._

## Proposal
_This is the primary content of the proposal explaining how the problem(s) will
be addressed._

### Goals and Non-goals
_A short list of what the goals of this proposal are and are not._

### Specification / Use Cases
_Detailed explanation of the proposal's design._

## Open Questions
_A list of questions that need to be answered._

## Answered Questions
_A list of questions that have been answered._
```

# Proposal Review
Once a proposal PR is submitted, project maintainers will review the proposal.
The goal of the review is to gain an understanding of the problem being solved
and the design of the proposed solution.

# Proposal States
| State | Definition |
| --- | --- |
| Draft | The proposal is actively being written by the proposer. |
| In Review | The proposal is being reviewed by project maintainers. |
| Accepted | The proposal has been accepted by the project maintainers. |
| Rejected | The proposal has been rejected by the project maintainers. |
