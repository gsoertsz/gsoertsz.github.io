---
layout: post
title: "Closing the artifact versioning loop with bumpversion via Github Actions"
categories: [python, continuous_delivery, github_actions, bumpversion]
---

*Abstract*

When versioning and publishing artefacts after merging a PR to a main branch, leverage bumpversion to publish the next version of the artifact, while maintaining full version consistency and traceability between the project definition, git and the published artefact.

<!--excerpt-above-->

The ideal developer experience in a continuous delivery context involves the following events:

1. A developer raises a PR for a branch they want merged to main
2. The branch is merged to main, and a new version is created
3. The new versioned artifact is built, tested and published - awaiting deployment




