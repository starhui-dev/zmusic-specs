<!--suppress HtmlDeprecatedAttribute -->
<div align="center">

![][banner]

![][license]
![][last-commit]
![][repo-size]

Shared specifications and protocol documents for ZMusic projects

[简体中文](README.md) | English | [日本語](README_JP.md)

</div>

## Introduction

ZMusic Specs maintains shared specifications for ZMusic projects.

This repository is the source of truth for contracts shared by the server plugin, client mod, and future related implementations. Any agreement that affects more than one repository should be documented here instead of being scattered across implementation repositories.

## Scope

This repository is for:

* Packet protocols
* Cross-repository API contracts
* Compatibility policies
* JSON schemas and protocol test vectors
* Architecture decisions that affect more than one repository

This repository is not for:

* TODOs for a single implementation repository
* Refactoring notes that only affect one repository
* User-facing documentation
* Release checklists for one repository

## Status

This repository is being prepared for ZMusic protocol work.

The first planned specification is the [ZMusic packet protocol](protocol/packet-protocol.md).

## Related Repositories

* [ZMusic Plugin](https://github.com/starhui-dev/zmusic-plugin) - Server plugin

## License

Specifications and documentation in this repository are licensed under [CC BY 4.0](LICENSE).

[banner]: https://socialify.git.ci/starhui-dev/zmusic-specs/image?description=1&forks=1&issues=1&name=1&owner=1&pulls=1&stargazers=1&theme=Auto

[license]: https://img.shields.io/github/license/starhui-dev/zmusic-specs?style=for-the-badge

[last-commit]: https://img.shields.io/github/last-commit/starhui-dev/zmusic-specs?style=for-the-badge

[repo-size]: https://img.shields.io/github/repo-size/starhui-dev/zmusic-specs?style=for-the-badge
