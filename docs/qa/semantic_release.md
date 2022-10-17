# Semantic Release
Semantic-release is an application created in nodejs, it allows to analyze the commits of a repository for the creation of tags. This allows greater control over the code displayed.

## How Semantic Selease works
---
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
