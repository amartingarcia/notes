---
title: Semantic Release
date: 20221023
author: Adrián Martín García
---

# Semantic Release
Semantic-release is an application created in nodejs, it allows to analyze the commits of a repository for the creation of tags. This allows greater control over the code displayed.

## How Semantic Selease works
semantic-release uses the commit messages to determine the consumer impact of changes in the codebase. Following formalized conventions for commit messages, semantic-release automatically determines the next semantic version number, generates a changelog and publishes the release.

By default, semantic-release uses [Angular Commit Message Conventions](https://github.com/angular/angular/blob/main/CONTRIBUTING.md#-commit-message-format). The commit message format can be changed with the preset or config options of the [@semantic-release/commit-analyzer](https://github.com/semantic-release/commit-analyzer#options) and [@semantic-release/release-notes-generator](https://github.com/semantic-release/release-notes-generator#options) plugins.

Tools such as commitizen or commitlint can be used to help contributors and enforce valid commit messages.

The table below shows which commit message gets you which release type when semantic-release runs (using the default configuration):

Starting with v1.0.0:

| Commit message      | Release type | Version |
|:-------------------:|:------------:|:-------:|
| `fix(app): new fix` | Patch Release | v1.0.1 |
| `feat(app): new feature` | MINOR Feature Release | v1.1.0 |
| `perf(app): a change`<br>`BREAKING CHANGE: change to major version` | Major Breaking Release | v2.0.0 |

Starting from forecast-v1.2.5, we can add multiple types of commits but it will only count one:

| Commit message      | Version |
|:-------------------:|:-------:|
| `fix(app): new fix` + `fix(app): other fix ` | v1.2.6 |
| `feat(app): new feature` + `feat(app): other feature ` | v1.3.0 |
