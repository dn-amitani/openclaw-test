# openclaw-test

# 🦞 OpenClaw 実践チュートリアル：AWS EC2 × Azure OpenAI 構築ガイド

AWS EC2 (Ubuntu) 環境に、自律型AIアシスタント「OpenClaw」を構築するためのガイドです。
セキュアなSSM接続、および LiteLLM を介した Azure OpenAI Service の利用を前提としています。

## 1. 環境へのアクセス (SSMポートフォワーディング)

インバウンドポートを開けずにWebダッシュボードへアクセスするため、AWS SSMのポートフォワーディングを利用します。

**ローカルPC（Windows/Mac）から実行:**
※ Windows環境でのJSONパースエラーを避けるため、Shorthand構文を使用します。

```bash
aws ssm start-session \
    --target "i-xxxxxxxxxxxxxxxxx" \
    --document-name AWS-StartPortForwardingSession \
    --parameters "portNumber=3000,localPortNumber=3000"
```

*このターミナルは維持したまま、別のターミナルから EC2 (Ubuntu) へログインして以下の作業を進めてください。*

---

## 2. LiteLLM の導入と Azure OpenAI 設定

Azure OpenAI を OpenAI 互換 API として動作させるため、LiteLLM プロキシを導入します。

### 2.1 LiteLLM のインストール
使い捨て環境を想定し、システムパッケージとして直接インストールします。

```bash
sudo apt update
sudo apt install -y python3-pip

# プロキシ機能を含めてインストール（システム環境への導入を許可するフラグ付き）
pip3 install 'litellm[proxy]' --break-system-packages
```

### 2.2 設定ファイルの作成
`~/litellm_config.yaml` を作成します。

```yaml
model_list:
  - model_name: "gpt-4o"
    litellm_params:
      model: "azure/your-deployment-name" # azure/ プレフィックスは必須
      api_base: "https://your-resource-name.openai.azure.com/"
      api_key: "YOUR_AZURE_OPENAI_API_KEY"
      api_version: "2024-02-15-preview"
```

### 2.3 LiteLLM プロキシの起動
PATHが通っていない場合を考慮し、フルパスで実行します。

```bash
~/.local/bin/litellm --config ~/litellm_config.yaml --port 4000 &
```

### 2.4 LiteLLM の動作確認
まず、LiteLLM プロキシ自体が起動していることを確認します。

```bash
curl -s http://localhost:4000/health
```

次に、OpenAI 互換 API としてモデル一覧が取得できることを確認します。

```bash
curl -s http://localhost:4000/v1/models
```

実際に Azure OpenAI まで到達して応答できることを確認するには、以下を実行します。

```bash
curl -s http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dummy" \
  -d '{
    "model": "gpt-4o",
    "messages": [
      {"role": "user", "content": "hello"}
    ]
  }'
```

モデル一覧またはチャット補完のレスポンスが返れば、LiteLLM が正しく起動し、Azure OpenAI と疎通できています。

---

## 3. OpenClaw のインストールと初期設定

### 3.1 Node.js と OpenClaw のインストール
OpenClaw は Node.js v22 以上を推奨します。

```bash
# Node.js v22.x のセットアップ (最後の - は標準入力を意味します)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# OpenClaw 本体をインストール
sudo npm install -g openclaw@latest
```

### 3.2 オンボーディング (初期設定)
以下のコマンドを実行し、ウィザードに従って入力します。

```bash
openclaw onboard
```

**推奨回答:**
- **Setup mode:** `QuickStart` を選択
- **Provider:** `openai`
- **Model:** `openai/gpt-4o` (プロバイダー明示が必須)
- **Base URL:** `http://localhost:4000/v1`
- **API Key:** `dummy`
- **How do you want to hatch your bot?** `Do this Later`   ← **重要!!!!**

> [!IMPORTANT]
> How do you want to hatch your bot? では Do this Later を選択してください。

### 3.3 初期設定の修正
選択肢にないのでダミーでModelをopenaiのモデルにしているが、上記設定のままではOpenAIのAPIを叩きに行くので、`~/.openclaw/openclaw.json`を編集し、LiteLLMのAPIにアクセスするように設定変更する。 
> [!NOTE]
> onbordで行った各種設定は`~/.openclaw/openclaw.json`に記載されている。編集することで設定変更可能。（ホットリロード）

#### "agents"
"agents"キーの内容を下記のように編集。"workspace"は環境に合わせ適宜。
```json
  "agents": {
    "defaults": {
      "workspace": "/home/ubuntu/.openclaw/workspace",
      "models": {
        "litellm/gpt-4o": {
          "alias": "Azure via LiteLLM"
        }
      },
      "model": {
        "primary": "litellm/gpt-4o"
      }
    }
  },
```
#### "models"
"models"キーは元ファイルにはないはずなので追加。
```
  "models": {
    "mode": "merge",
    "providers": {
      "litellm": {
        "baseUrl": "http://127.0.0.1:4000/v1",
        "apiKey": "dummy",
        "api": "openai-completions",
        "timeoutSeconds": 300,
        "models": [
          {
            "id": "gpt-4o",
            "name": "gpt-4o (LiteLLM)",
            "input": ["text"],
            "reasoning": false
          }
        ]
      }
    }
  },
```
設定変更後に、以下コマンドを実行し、litellmを見るようになっているかを確認。
```
$ openclaw config get agents.defaults.model.primary
```

### 3.4 Web UIへアクセス

勝手に`openclaw-gateway`が立ち上がるが、立ち上がっていなければ下記のコマンドでゲートウェイを起動します。
```bash
openclaw gateway start
```

その後、ブラウザで `http://localhost:18789` にアクセスし、ダッシュボードが表示されることを確認してください。

トークンは上記設定ファイルに記載されいるのでそれを入力してあとはデフォルトのままログインボタンを押下。（２～３分くらいログインにかかるので、待つ）




## 4. プロジェクト 1：最初の "Hello World" と人格設定

OpenClaw は `~/.openclaw/` 配下の Markdown ファイルで AI の挙動を定義します。

### 4.1 SOUL (人格) の定義
`~/.openclaw/SOUL.md` を作成または編集します。

```markdown
# 人格定義
あなたは「Claw-Agent」という名の、優秀で少し皮肉屋なアシスタントです。
挨拶は常に「システムオールグリーン。何か用ですか？」から始めてください。
```

### 4.2 動作確認
ダッシュボードのチャット画面で「自己紹介して」と送り、設定した挨拶が返ってくれば成功です。

---

## 5. プロジェクト 2：自律タスク (スキル) の作成

OpenClaw の特徴である「バックグラウンドでの自律稼働」を試します。

### 5.1 カスタムスキルの作成
`~/.openclaw/skills/system_monitor.md` を作成します。

```markdown
# スキル: system_monitor

## 概要
サーバーのディスク容量をチェックし、結果を報告する。

## 実行ステップ
1. ターミナルで `df -h` コマンドを実行する。
2. ルートディレクトリ(`/`)の使用率が 80% を超えていないか確認する。
3. 確認結果を `~/.openclaw/MEMORY.md` に追記する。
```

### 5.2 定期実行 (Cron) の登録
サーバーを常に監視させるよう指示します。

```bash
openclaw cron add --name "DiskCheck" --cron "0 * * * *" --message "system_monitorスキルを実行して状態をチェックして"
```

これで、あなたがブラウザを閉じていても、OpenClaw が 1 時間ごとに自律的にサーバーの状態をチェックし、ログを残すようになります。
