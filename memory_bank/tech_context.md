# 技術コンテキスト

## 使用技術

OpenHandsは、以下の主要な技術を使用しています：

1. **バックエンド**:
   - **Python 3.12**: バックエンドの実装に使用されています。
   - **Poetry**: 依存関係管理に使用されています。
   - **LiteLLM**: 様々な大規模言語モデルプロバイダーとの統合を容易にするために使用されています。

2. **フロントエンド**:
   - **Node.js 20.x以上**: フロントエンドの実装に使用されています。
   - **React**: ユーザーインターフェースの構築に使用されています。
   - **TypeScript**: 型安全なJavaScriptコードを書くために使用されています。
   - **Tailwind CSS**: スタイリングに使用されています。
   - **Vite**: ビルドツールとして使用されています。

3. **コンテナ化とデプロイメント**:
   - **Docker**: コマンドの実行とシステム全体のコンテナ化に使用されています。
   - **Docker Compose**: 複数のコンテナの管理に使用されています。

4. **テスト**:
   - **pytest**: Pythonコードのテストに使用されています。
   - **Vitest**: フロントエンドのテストに使用されています。
   - **Playwright**: ブラウザテストに使用されています。

## 開発セットアップ

OpenHandsの開発環境をセットアップするには、以下の要件が必要です：

1. **オペレーティングシステム**:
   - Linux
   - Mac OS
   - Windows（WSL、Ubuntu 22.04以上）

2. **必須ソフトウェア**:
   - Docker
   - Python 3.12
   - Node.js 20.x以上
   - Poetry 1.8以上

3. **OS固有の依存関係**:
   - Ubuntu: build-essential
   - WSL: netcat

開発環境のセットアップ手順は以下の通りです：

1. プロジェクトをビルドする: `make build`
2. 言語モデルを設定する: `make setup-config`
3. アプリケーションを実行する: `make run`

また、バックエンドとフロントエンドを個別に起動することもできます：
- バックエンドサーバーの起動: `make start-backend`
- フロントエンドサーバーの起動: `make start-frontend`

## 技術的制約

OpenHandsには、以下の技術的制約があります：

1. **単一ユーザー向け**: OpenHandsは、ローカルワークステーション上で単一ユーザーによって実行されることを想定しています。マルチテナントデプロイメント（複数のユーザーが同じインスタンスを共有する）には適していません。

2. **Dockerの依存性**: コマンドの実行とサンドボックス化にDockerを使用しているため、Dockerがインストールされていない環境では実行できません。

3. **言語モデルの依存性**: OpenHandsは、大規模言語モデルに依存しています。デフォルトではClaude 3.5 Sonnetを使用していますが、他のモデルも使用できます。

4. **インターネット接続**: 言語モデルAPIにアクセスするためにインターネット接続が必要です。

## 依存関係

OpenHandsの主要な依存関係は以下の通りです：

1. **Python依存関係**:
   - LiteLLM: 言語モデルとの統合
   - その他のPython依存関係はpyproject.tomlとpoetry.lockファイルで管理されています。

2. **Node.js依存関係**:
   - React: ユーザーインターフェースの構築
   - TypeScript: 型安全なJavaScriptコード
   - その他のNode.js依存関係はpackage.jsonとpackage-lock.jsonファイルで管理されています。

3. **外部サービス**:
   - 言語モデルAPI（例：Anthropic Claude 3.5 Sonnet）
   - Docker: コマンドの実行とサンドボックス化

## デバッグとトラブルシューティング

言語モデルに関する問題が発生した場合、環境変数`DEBUG=1`をエクスポートしてバックエンドを再起動することで、プロンプトとレスポンスが`logs/llm/CURRENT_DATE`ディレクトリに記録されます。これにより、問題の原因を特定することができます。

## 依存関係の追加または更新

1. `pyproject.toml`に依存関係を追加するか、`poetry add xxx`を使用します。
2. `poetry lock --no-update`を使用してpoetry.lockファイルを更新します。

## Dockerコンテナ内での開発

開発時間を短縮するために、既存のDockerコンテナイメージを使用することができます。これは、環境変数`SANDBOX_RUNTIME_CONTAINER_IMAGE`を目的のDockerイメージに設定することで実現できます。

例: `export SANDBOX_RUNTIME_CONTAINER_IMAGE=ghcr.io/all-hands-ai/runtime:0.27-nikolaik`

また、`make docker-dev`コマンドを使用して、Dockerコンテナ内で開発することもできます。ホスト上に必要なツールをすべてインストールせずにOpenHandsを実行したい場合は、`make docker-run`コマンドを使用できます。
