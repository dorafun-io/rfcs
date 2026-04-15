# DoraFun RFCs

The "RFC" (request for comments) process is intended to provide a consistent and controlled path for major changes to the **DoraFun** ecosystem (including the `dop` CLI, the DoraFun Platform, and Embodied AI hardware/communication protocols) so that all stakeholders can be confident about the direction of the project.

Many changes, including bug fixes and documentation improvements, can be implemented and reviewed via the normal GitHub pull request workflow in their respective repositories.

Some changes, though, are "substantial", and we ask that these be put through a bit of a design process to produce a consensus among the LoopFun open-source community, core maintainers, and commercial stakeholders (e.g., DoraOne).

## Table of Contents

- [DoraFun RFCs](#dorafun-rfcs)
  - [Table of Contents](#table-of-contents)
  - [When you need to follow this process](#when-you-need-to-follow-this-process)
  - [Before creating an RFC](#before-creating-an-rfc)
  - [What the process is](#what-the-process-is)
  - [The RFC life-cycle](#the-rfc-life-cycle)
  - [Reviewing RFCs](#reviewing-rfcs)
  - [Implementing an RFC](#implementing-an-rfc)
  - [License \& Contributions](#license--contributions)

## When you need to follow this process

You need to follow this process if you intend to make "substantial" changes to `dop`, the DoraFun Platform, or the RFC process itself. What constitutes a "substantial" change varies depending on what part of the ecosystem you are proposing to change, but may include the following:

- Any semantic or syntactic change to the `dop` CLI commands or flags.
- Breaking changes to the DORA zero-copy shared memory bus or Zenoh network topologies.
- New standardizations for hardware interfaces (e.g., Ascend CANN, ARM architectures).
- Large architectural additions to the DoraFun package management and deployment flow.

Some changes do not require an RFC:
- Rephrasing, reorganizing, refactoring, or otherwise "changing shape that does not change meaning".
- Additions that strictly improve objective, numerical quality criteria (speedup, warning removal, better platform coverage).
- Minor bug fixes or platform infrastructure updates invisible to users-of-dorafun.

If you submit a pull request to implement a new feature directly into `dorafun-io/dop` without going through the RFC process, it may be closed with a polite request to submit an RFC first.

## Before creating an RFC

A hastily-proposed RFC can hurt its chances of acceptance. Low-quality proposals, proposals for previously-rejected features, or those that don't fit into the near-term roadmap, may be quickly rejected.

It is generally a good idea to pursue feedback from other project developers beforehand. The most common preparations include discussing the topic in the **LoopFun Community Forums** or GitHub Discussions, and occasionally posting "pre-RFCs". Receiving encouraging feedback from long-standing project developers is a good indication that the RFC is worth pursuing.

## What the process is

In short, to get a major feature added to the DoraFun ecosystem, one must first get the RFC merged into the RFC repository as a markdown file. At that point, the RFC is "active" and may be implemented.

1. **Fork** the RFC repository (`dorafun-io/rfcs`).
2. **Copy** `text/0000-template.md` to `text/0000-my-feature.md` (where "my-feature" is descriptive). Don't assign an RFC number yet.
3. **Fill in the RFC**. Put care into the details: RFCs that do not present convincing motivation, demonstrate a lack of understanding of the design's impact on edge hardware, or are disingenuous about the drawbacks tend to be poorly received.
4. **Submit a pull request**. The RFC will receive design feedback from the larger community.
5. **Update the ID**: Now that your RFC has an open PR, use the issue number of the PR to rename the file (e.g., update the `0000-` prefix to that number).
6. **Build consensus** and integrate feedback. You can make edits, big and small, as new commits to the pull request. **Do not squash or rebase commits after they are visible on the PR.**
7. **FCP (Final Comment Period)**: At some point, a core team member will propose a "motion for final comment period" (FCP), along with a *disposition* for the RFC (merge, close, or postpone). The FCP lasts **7 calendar days**. This way, all stakeholders have a chance to lodge any final objections before a decision is reached.

## The RFC life-cycle

Once an RFC becomes "active", authors may implement it and submit the feature as a pull request to the respective `dorafun-io` repository (e.g., `dorafun-io/dop`). Being "active" is not a rubber stamp, but it does mean that in principle, all the major stakeholders have agreed to the feature.

Furthermore, the fact that a given RFC has been accepted implies nothing about what priority is assigned to its implementation. While it is not *necessary* that the author of the RFC also write the implementation, it is by far the most effective way to see an RFC through to completion.

## Reviewing RFCs

A core sub-team makes final decisions about RFCs after the benefits and drawbacks are well understood. When a decision is made, the RFC pull request will either be merged or closed. The sub-team will add a comment describing the rationale for the decision.

## Implementing an RFC

Every accepted RFC has an associated tracking issue in the relevant repository (e.g., `dorafun-io/dop` or `dorafun-io/platform`). The RFC author is not obligated to implement it, but is welcome to post an implementation for review.

## License & Contributions

This repository is currently in the process of being licensed under either of:

* Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or https://www.apache.org/licenses/LICENSE-2.0)
* MIT license ([LICENSE-MIT](LICENSE-MIT) or https://opensource.org/licenses/MIT)

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you shall be dual-licensed as above, without any additional terms or conditions.