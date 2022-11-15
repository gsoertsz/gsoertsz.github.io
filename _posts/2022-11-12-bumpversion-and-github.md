---
layout: post
title: "Bumpversion and preventing github workflow re-execution"
categories: python github
---

If you are using merges to main to bump the version of your updated component, before pushing the tagged commit back to main, this can cause github action workflows to kick off again on main, which is error prone and clumsy.

Configuring bumpversion via `.bumpversion.cfg` to use a special commit message, can prevent this.
<!--excerpt-above-->



```
Here is some code
```