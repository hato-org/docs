# Hato フロントエンド開発環境構築

Hatoのフロントエンドページを開発するための環境の構築ガイドです。

## 必要なもの

- Node.js (v18以降)
- Git
- GitHubアカウント
- 任意のテキストエディタ (VSCode、vim等)

## 構築手順

1. Node.jsのインストール

[Node.js公式サイト](https://nodejs.org/en)から、Node.jsをダウンロード・インストールしてください

- LTS、Currentの2つがありますがどちらでも大丈夫です

1. yarnの有効化

Hatoではパッケージマネージャにyarnを使用しているため、yarnを有効化します

Windowsの場合、PowerShellまたはコマンドプロンプトを開き、以下を実行してください  
MacOSの場合はターミナルを開き以下を実行してください

```bash
corepack enable
```

1. Gitのインストール

[Git公式サイト](https://git-scm.com/downloads)から、Gitをダウンロード・インストールしてください

- インストール時に様々なオプションがありますが、全てデフォルトでも大丈夫です
- コミットメッセージを入力するエディタの選択画面では自分が使っているエディタを選ぶと便利です

1. リポジトリのクローン

Gitを使い、GitHubのHatoのリポジトリからファイルをダウンロードします

```bash
git clone https://github.com/hato-org/hato.git
```

