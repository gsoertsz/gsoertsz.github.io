---
layout: post
title: "Maintainable CI/CD pipelines with effective bespoke CLI's"
categories: [python, cli, devops, cicd]
---

*Abstract*

Bespoke CLI's, built productively, and treated as 3rd party tools by CI/CD pipelines, are a compelling approach to dealing with exceptional integration scenarios for deployment and configuration purposes. Multiple frameworks exist supporting the rapid creation of ergonomic and well built bespoke CLI's. If treated as discrete, pre-tested and purpose specific artefacts, bespoke CLI's can improve CI/CD pipeline maintainability, by hiding complexity associated with novel deployment integration requirements. The typical response to these scenarios, is to either abuse the pipeline DSL, or simply 'write a script' that can be invoked - either way degrading overall pipeline maintainability.

<!--excerpt-above-->

# Modern CI/CD pipelines are busy and unmaintainable

As the heterogeneity and complexity of our solution spaces increase, so to do the demands on our often singular and centralised CI/CD tool. Greater flexibility is sought in what the CI/CD pipeline can do, and the tools and interfaces with which it can integrate. Pipeline developers are often tempted to use scripting (bash, powershell etc) to orchestrate complex, cross domain deployments and configuration actions, which creates upward pressure on complexity. The usual remedies to this complexity (testing, logging, etc) are ineffective, and no more productive than re-attempting pipeline executions many times, tweaking inline logic slightly hoping for different results or more information. This can pose challenges for maintainability, not to mention potentially impact production or the ability to safely deploy to the production (or any) environment. While terraform, with its built-in support for multiple providers, solves this problem nicely, we are at the mercy of maintainers of those providers to provide support for their tools, lest we build and maintain our own providers.

# Pipelines are a nightmare to test and debug

# Bash vs. Powershell

# Bespoke CLI's are not that hard

