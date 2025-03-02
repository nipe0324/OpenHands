# システムアーキテクチャ

## 全体アーキテクチャの概要

OpenHandsは、AIを活用したソフトウェア開発エージェントのためのプラットフォームです。システムは、大規模言語モデル（LLM）を中心に構築されており、エージェントがユーザーの指示に基づいてタスクを実行できるようにするための様々なコンポーネントで構成されています。

アーキテクチャは、イベント駆動型のメッセージパッシングシステムを採用しており、EventStreamがすべてのコンポーネント間の通信のバックボーンとして機能しています。

## コンポーネントの内訳

OpenHandsの主要なコンポーネントは以下の通りです：

1. **LLM**: 大規模言語モデルとのすべての対話を仲介します。LiteLLMのおかげで、あらゆる基盤となる補完モデルと連携できます。

2. **Agent**: 現在のStateを見て、目標に一歩近づくためのActionを生成する役割を担います。

3. **AgentController**: Agentを初期化し、Stateを管理し、Agentを一歩ずつ前進させるメインループを駆動します。

4. **State**: エージェントのタスクの現在の状態を表します。現在のステップ、最近のイベントの履歴、エージェントの長期計画などが含まれます。

5. **EventStream**: イベントの中央ハブであり、任意のコンポーネントがイベントを発行したり、他のコンポーネントによって発行されたイベントをリッスンしたりできます。
   - **Event**: ActionまたはObservationです。
     - **Action**: ファイルの編集、コマンドの実行、メッセージの送信などの要求を表します。
     - **Observation**: ファイルの内容やコマンドの出力など、環境から収集された情報を表します。

6. **Runtime**: Actionを実行し、Observationを返送する役割を担います。
   - **Sandbox**: Dockerなどの中でコマンドを実行する、ランタイムの一部です。

7. **Server**: HTTPを介してOpenHandsセッションを仲介し、フロントエンドを駆動します。
   - **Session**: 単一のEventStream、単一のAgentController、単一のRuntimeを保持します。一般的に単一のタスク（ただし、複数のユーザープロンプトを含む可能性があります）を表します。
   - **ConversationManager**: アクティブなセッションのリストを保持し、リクエストが正しいSessionにルーティングされるようにします。

## 通信とデータフロー

OpenHandsのコントロールフローは、以下の基本的なループによって駆動されます：

```python
while True:
  prompt = agent.generate_prompt(state)
  response = llm.completion(prompt)
  action = agent.parse_response(response)
  observation = runtime.run(action)
  state = state.update(action, observation)
```

実際には、このほとんどはEventStreamを介したメッセージパッシングによって実現されています。EventStreamは、OpenHandsのすべての通信のバックボーンとして機能します。

コンポーネント間の通信フローは以下の通りです：

1. Agent → AgentController: Actionを送信
2. AgentController → Agent: Stateを送信
3. AgentController → EventStream: Actionを送信
4. EventStream → AgentController: Observationを送信
5. Runtime → EventStream: Observationを送信
6. EventStream → Runtime: Actionを送信
7. Frontend → EventStream: Actionを送信

## デプロイメントアーキテクチャ

OpenHandsは、Dockerを使用してコンテナ化されており、ローカルワークステーション上で単一ユーザーによって実行されることを想定しています。マルチテナントデプロイメント（複数のユーザーが同じインスタンスを共有する）には適していません。

## スケーラビリティとパフォーマンス

現在のバージョンのOpenHandsは、単一ユーザーのローカルワークステーション上での使用を想定しており、組み込みの分離やスケーラビリティはありません。マルチテナント環境でOpenHandsを実行することに興味がある場合は、高度なデプロイメントオプションについて開発チームに連絡することが推奨されています。

## セキュリティとコンプライアンス

OpenHandsは、Sandboxコンポーネントを通じてコマンドを実行する際にDockerを使用して分離を提供します。ただし、これはマルチテナント環境での完全なセキュリティを保証するものではありません。

## 統合ポイント

OpenHandsは、以下の外部サービスと統合しています：

1. **言語モデルプロバイダー**: Anthropicの Claude 3.5 Sonnetが最もよく機能しますが、他の多くのオプションもサポートされています。
2. **Docker**: コマンドの実行とサンドボックス化に使用されます。
3. **ローカルファイルシステム**: OpenHandsをローカルファイルシステムに接続して、ファイルの読み書きを行うことができます。
