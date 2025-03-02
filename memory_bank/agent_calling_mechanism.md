# OpenHandsにおけるエージェントの呼び分け方法

OpenHandsプロジェクトでは、複数のエージェントを効率的に管理・呼び出すための仕組みが実装されています。以下に概要の流れと該当コードを説明します。

## 1. エージェントの登録と取得の仕組み

### 基本的な登録の仕組み

OpenHandsでは、`Agent`クラスが抽象基底クラス（ABC）として実装されており、すべてのエージェントはこのクラスを継承します。エージェントの登録と取得は、`Agent`クラスの静的な`_registry`辞書を使用して行われます。

```python
# openhands/controller/agent.py
class Agent(ABC):
    _registry: dict[str, Type['Agent']] = {}

    @classmethod
    def register(cls, name: str, agent_cls: Type['Agent']):
        """エージェントクラスをレジストリに登録する"""
        if name in cls._registry:
            raise AgentAlreadyRegisteredError(name)
        cls._registry[name] = agent_cls

    @classmethod
    def get_cls(cls, name: str) -> Type['Agent']:
        """レジストリからエージェントクラスを取得する"""
        if name not in cls._registry:
            raise AgentNotRegisteredError(name)
        return cls._registry[name]
```

### マイクロエージェントの登録

マイクロエージェントは、`openhands/agenthub/micro/registry.py`で自動的に登録されます。このファイルは、`micro`ディレクトリ内の各サブディレクトリを走査し、`prompt.md`と`agent.yaml`ファイルを持つディレクトリを見つけます。これらのファイルからエージェントの定義を読み込み、`all_microagents`辞書に登録します。

```python
# openhands/agenthub/micro/registry.py
all_microagents = {}

# ディレクトリのリストを取得し、決定論的な順序を保つためにソート
dirs = sorted(os.listdir(os.path.dirname(__file__)))

for dir in dirs:
    base = os.path.dirname(__file__) + '/' + dir
    if os.path.isfile(base):
        continue
    if dir.startswith('_'):
        continue
    promptFile = base + '/prompt.md'
    agentFile = base + '/agent.yaml'
    if not os.path.isfile(promptFile) or not os.path.isfile(agentFile):
        raise Exception(f'Missing prompt or agent file in {base}. Please create them.')
    with open(promptFile, 'r') as f:
        prompt = f.read()
    with open(agentFile, 'r') as f:
        agent = yaml.safe_load(f)
    if 'name' not in agent:
        raise Exception(f'Missing name in {agentFile}')
    agent['prompt'] = prompt
    all_microagents[agent['name']] = agent
```

## 2. エージェントの呼び出し方法

### AgentControllerによる呼び出し

エージェントの呼び出しは、主に`AgentController`クラスによって管理されます。`AgentController`は、エージェントのライフサイクルを管理し、イベントストリームとの通信を処理します。

```python
# openhands/controller/agent_controller.py
class AgentController:
    def __init__(
        self,
        agent: Agent,
        event_stream: EventStream,
        max_iterations: int,
        # その他のパラメータ...
    ):
        self.id = sid
        self.agent = agent
        self.event_stream = event_stream
        # その他の初期化...
```

### エージェントの委任（Delegation）

OpenHandsでは、あるエージェントが別のエージェントにタスクを委任する機能があります。これは`AgentDelegateAction`を通じて実現されます。

```python
# openhands/controller/agent_controller.py
async def start_delegate(self, action: AgentDelegateAction) -> None:
    """サブタスクを処理するためのデリゲートエージェントを開始する"""
    agent_cls: Type[Agent] = Agent.get_cls(action.agent)
    agent_config = self.agent_configs.get(action.agent, self.agent.config)
    llm_config = self.agent_to_llm_config.get(action.agent, self.agent.llm.config)
    llm = LLM(config=llm_config, retry_listener=self._notify_on_llm_retry)
    delegate_agent = agent_cls(llm=llm, config=agent_config)
    # 状態の設定など...

    # デリゲートを作成
    self.delegate = AgentController(
        sid=self.id + '-delegate',
        agent=delegate_agent,
        event_stream=self.event_stream,
        # その他のパラメータ...
        is_delegate=True,
    )
```

### DelegatorAgentの例

`DelegatorAgent`は、タスクを他のエージェントに委任するための専用エージェントです。これは、複雑なタスクを複数のエージェントに分割して処理するために使用されます。

```python
# openhands/agenthub/delegator_agent/agent.py
class DelegatorAgent(Agent):
    def step(self, state: State) -> Action:
        """現在のステップが完了したかどうかを確認し、完了した場合はAgentFinishActionを返す。
        それ以外の場合は、パイプラインの次のエージェントにタスクを委任する。"""
        if self.current_delegate == '':
            self.current_delegate = 'study'
            task, _ = state.get_current_user_intent()
            return AgentDelegateAction(
                agent='StudyRepoForTaskAgent', inputs={'task': task}
            )

        # 委任の状態に基づいて次のエージェントを決定
        if self.current_delegate == 'study':
            self.current_delegate = 'coder'
            return AgentDelegateAction(
                agent='CoderAgent',
                inputs={
                    'task': goal,
                    'summary': last_observation.outputs['summary'],
                },
            )
        # その他の委任ロジック...
```

## 3. マイクロエージェントの実装

マイクロエージェントは、`MicroAgent`クラスを通じて実装されます。各マイクロエージェントは、`prompt.md`ファイルでプロンプトを定義し、`agent.yaml`ファイルで設定を定義します。

```python
# openhands/agenthub/micro/agent.py
class MicroAgent(Agent):
    VERSION = '1.0'
    prompt = ''
    agent_definition: dict = {}

    def __init__(self, llm: LLM, config: AgentConfig):
        super().__init__(llm, config)
        if 'name' not in self.agent_definition:
            raise ValueError('Agent definition must contain a name')
        self.prompt_template = Environment(loader=BaseLoader()).from_string(self.prompt)
        self.delegates = all_microagents.copy()
        del self.delegates[self.agent_definition['name']]

    def step(self, state: State) -> Action:
        # プロンプトのレンダリングと実行...
        prompt = self.prompt_template.render(
            state=state,
            instructions=instructions,
            to_json=to_json,
            history_to_json=self.history_to_json,
            delegates=self.delegates,
            latest_user_message=last_user_message,
        )
        # LLMの呼び出しとアクションの解析...
        action = parse_response(action_resp)
        return action
```

### マイクロエージェントの例

例えば、`CoderAgent`と`PostgresAgent`は、それぞれ特定のタスクに特化したマイクロエージェントです。

**CoderAgent（openhands/agenthub/micro/coder/prompt.md）**:
```markdown
# Task
You are a software engineer. You've inherited an existing codebase, which you
need to modify to complete this task:

{{ state.inputs.task }}

{% if state.inputs.summary %}
Here's a summary of the codebase, as it relates to this task:

{{ state.inputs.summary }}
{% endif %}

## Available Actions
{{ instructions.actions.run }}
{{ instructions.actions.write }}
{{ instructions.actions.read }}
{{ instructions.actions.message }}
{{ instructions.actions.finish }}

Do NOT finish until you have completed the tasks.
```

**PostgresAgent（openhands/agenthub/micro/postgres_agent/prompt.md）**:
```markdown
# Task
You are a database engineer. You are working on an existing Postgres project, and have been given
the following task:

{{ state.inputs.task }}

You must:
* Investigate the existing migrations to understand the current schema
* Write a new migration to accomplish the task above
* Test that the migrations work properly

## Actions
You may take any of the following actions:
{{ instructions.actions.message }}
{{ instructions.actions.read }}
{{ instructions.actions.write }}
{{ instructions.actions.run }}
```

## 4. エージェント呼び出しの全体的な流れ

1. **初期化**: ユーザーからのタスクを受け取ると、`AgentController`が適切なエージェントを初期化します。
2. **ステップ実行**:
   - コントローラーが`_step`メソッドを呼び出します。
   - エージェントの`step`メソッドが呼び出され、現在の状態に基づいてアクションを生成します。
   - アクションがイベントストリームに送信されます。
   - ランタイムがアクションを実行し、観察結果を返します。
   - 観察結果がイベントストリームを通じてコントローラーに戻ります。
   - コントローラーが状態を更新します。
3. **委任**:
   - エージェントが`AgentDelegateAction`を返すと、コントローラーは`start_delegate`メソッドを呼び出します。
   - 新しい`AgentController`が作成され、委任されたタスクを実行します。
   - 委任が完了すると、結果が`AgentDelegateObservation`としてイベントストリームに送信されます。
4. **終了**:
   - エージェントが`AgentFinishAction`または`AgentRejectAction`を返すと、コントローラーは状態を更新し、実行を終了します。

この仕組みにより、OpenHandsは複数のエージェントを効率的に管理し、複雑なタスクを分割して処理することができます。
