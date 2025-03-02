# システムパターン

## アーキテクチャパターン

OpenHandsは、以下の主要なアーキテクチャパターンを採用しています：

1. **イベント駆動アーキテクチャ**: システムの中核はEventStreamであり、これを通じてすべてのコンポーネントが通信します。コンポーネントはイベントを発行し、他のコンポーネントからのイベントをリッスンします。

2. **モデル-ビュー-コントローラ (MVC)**: AgentControllerがコントローラーとして機能し、Stateがモデル、フロントエンドがビューとして機能します。

3. **マイクロサービスアーキテクチャの要素**: 各コンポーネント（Agent、Runtime、LLM、など）は明確に定義された責任を持ち、独立して機能します。

4. **サンドボックス化**: Runtimeコンポーネントは、Dockerを使用してコマンドの実行を分離し、セキュリティを確保します。

## 主要な技術的決定

1. **LiteLLMの採用**: 様々な大規模言語モデルプロバイダーとの統合を容易にするために、LiteLLMライブラリを使用しています。

2. **Dockerの使用**: コマンドの実行とシステム全体のコンテナ化にDockerを使用しています。これにより、一貫した実行環境と分離が提供されます。

3. **Python 3.12**: バックエンドの実装にPython 3.12を選択しています。これは、最新の言語機能と良好なパフォーマンスを提供します。

4. **Node.js 20.x以上**: フロントエンドの実装に最新のNode.jsを使用しています。

5. **Poetry**: 依存関係管理にPoetryを使用しています。これにより、再現可能なビルドと依存関係の管理が容易になります。

## 設計パターン

OpenHandsは、以下の設計パターンを採用しています：

1. **オブザーバーパターン**: EventStreamを通じて実装されており、コンポーネントがイベントを発行し、他のコンポーネントがそれらをリッスンできるようにします。

2. **ステートパターン**: Stateオブジェクトは、エージェントの現在の状態を表し、アクションと観察に基づいて更新されます。

3. **ファクトリーパターン**: 様々なタイプのAgentやRuntimeを作成するために使用されている可能性があります。

4. **ストラテジーパターン**: 異なるLLMプロバイダーや実行環境に対して異なる戦略を使用できるようにしています。

5. **コマンドパターン**: Actionオブジェクトは、実行されるべきコマンドをカプセル化しています。

6. **メディエーターパターン**: EventStreamは、コンポーネント間の通信を仲介するメディエーターとして機能します。

## コンポーネント関係

OpenHandsの主要なコンポーネントとその関係は以下の通りです：

1. **Agent と AgentController**:
   - AgentControllerはAgentを初期化し、制御します。
   - AgentはStateを受け取り、Actionを生成します。
   - AgentControllerはActionをEventStreamに送信し、Observationを受け取ります。

2. **EventStream と他のコンポーネント**:
   - すべてのコンポーネントはEventStreamを通じて通信します。
   - AgentControllerはActionをEventStreamに送信します。
   - RuntimeはEventStreamからActionを受け取り、Observationを返します。
   - フロントエンドはEventStreamにActionを送信できます。

3. **Runtime と Sandbox**:
   - RuntimeはActionを実行し、Observationを生成します。
   - SandboxはRuntimeの一部であり、Dockerなどの中でコマンドを実行します。

4. **Server、Session、ConversationManager**:
   - ServerはHTTPを介してOpenHandsセッションを仲介します。
   - SessionはEventStream、AgentController、Runtimeを保持します。
   - ConversationManagerはアクティブなSessionのリストを管理します。

5. **LLM と Agent**:
   - AgentはLLMを使用して、Stateに基づいてActionを生成します。
   - LLMはLiteLLMを通じて様々な大規模言語モデルプロバイダーと統合されています。

これらのパターンと関係により、OpenHandsは柔軟で拡張可能なアーキテクチャを持ち、様々なユースケースに対応できるようになっています。
