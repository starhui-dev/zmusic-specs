<!--suppress HtmlDeprecatedAttribute -->
<div align="center">

![][banner]

![][license]
![][last-commit]
![][repo-size]

ZMusic プロジェクト向けの共有仕様とプロトコル文書

[简体中文](README.md) | [English](README_EN.md) | 日本語

</div>

## 概要

ZMusic Specs は、ZMusic プロジェクト間で共有する仕様を管理するためのリポジトリです。

このリポジトリは、サーバープラグイン、クライアント Mod、将来の関連実装が共有する契約の信頼できる情報源です。複数のリポジトリに影響する取り決めは、個別の実装リポジトリに散らばらせず、ここに記録します。

## 管理対象

このリポジトリで管理するもの:

* プラグインメッセージ通信プロトコル
* リポジトリ間 API 契約
* 互換性ポリシー
* JSON Schema とプロトコルテストサンプル
* 複数のリポジトリに影響するアーキテクチャ決定

このリポジトリで管理しないもの:

* 単一実装リポジトリの TODO
* 1 つのリポジトリだけに影響するリファクタリングメモ
* ユーザー向け利用ドキュメント
* 単一リポジトリのリリースチェックリスト

## 現在の状態

このリポジトリは ZMusic V4 プロトコル作業の準備中です。

最初に管理する予定の仕様は、ZMusic V4 プラグインメッセージ通信プロトコルです。

## 関連リポジトリ

* [ZMusic Plugin](https://github.com/zmusic-dev/zmusic-plugin) - サーバープラグイン

## ライセンス

このリポジトリの仕様と文書は [CC BY 4.0](LICENSE) の下で公開されています。

[banner]: https://socialify.git.ci/starhui-dev/zmusic-specs/image?description=1&forks=1&issues=1&name=1&pulls=1&stargazers=1&theme=Auto

[license]: https://img.shields.io/github/license/starhui-dev/zmusic-specs?style=for-the-badge

[last-commit]: https://img.shields.io/github/last-commit/starhui-dev/zmusic-specs?style=for-the-badge

[repo-size]: https://img.shields.io/github/repo-size/starhui-dev/zmusic-specs?style=for-the-badge

