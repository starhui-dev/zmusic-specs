<!--suppress HtmlDeprecatedAttribute -->
<div align="center">

![][banner]

![][license]
![][last-commit]
![][repo-size]

ZMusic 跨仓库规范与协议文档

简体中文 | [English](README_EN.md) | [日本語](README_JP.md)

</div>

## 简介

ZMusic Specs 用于维护 ZMusic 项目之间共享的规范。

这个仓库是服务端插件、客户端 Mod 以及未来相关实现之间的协议事实源。所有会影响多个仓库的约定，都应优先在这里沉淀，而不是散落在单个实现仓库中。

## 维护内容

本仓库适合维护：

* 通信包协议
* 跨仓库 API 契约
* 兼容性策略
* JSON Schema 和协议测试样例
* 影响多个仓库的架构决策

本仓库不维护：

* 单个实现仓库的 TODO
* 只影响某个仓库的重构说明
* 面向用户的使用文档
* 单个仓库的发布检查清单

## 当前状态

本仓库正在为 ZMusic 协议工作做准备。

第一份计划维护的规范是 [ZMusic 通信包协议](protocol/packet-protocol.md)。

## 相关仓库

* [ZMusic Plugin](https://github.com/starhui-dev/zmusic-plugin) - 服务端插件

## 开源协议

本仓库中的规范和文档使用 [CC BY 4.0](LICENSE) 协议发布。

[banner]: https://socialify.git.ci/starhui-dev/zmusic-specs/image?description=1&forks=1&issues=1&name=1&owner=1&pulls=1&stargazers=1&theme=Auto

[license]: https://img.shields.io/github/license/starhui-dev/zmusic-specs?style=for-the-badge

[last-commit]: https://img.shields.io/github/last-commit/starhui-dev/zmusic-specs?style=for-the-badge

[repo-size]: https://img.shields.io/github/repo-size/starhui-dev/zmusic-specs?style=for-the-badge
