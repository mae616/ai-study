# AG-UI 天気予報チャットボット 学習セッション履歴

**日付**: 2025年10月9日
**目標**: AG-UI Protocolを学び、Next.js + Vercelで天気予報チャットボットを構築

---

## 📚 学習した概念

### 1. AG-UI Protocol とは？

**一言で言うと**:
> フロントやバックでAIを今より扱いやすくする中間層（統一プロトコル）

**核心的な価値**:
- **AIフレームワーク非依存**: LangGraph/Pydantic AI/Mastra等、バックエンドを変えてもフロントコード不変
- **HTTPのAI版**: HTTPがWebサーバーを抽象化したように、AG-UIがAIエージェントを抽象化
- **イベント駆動**: JavaScriptのイベント駆動パターンとの相性が抜群

---

### 2. イベント駆動アーキテクチャ

**従来の命令型（同期）**:
```javascript
// ❌ 5秒待つだけ、何が起きてるか不明
const result = await getWeather("東京");
console.log(result);
```

**イベント駆動（非同期ストリーム）**:
```javascript
// ✅ リアルタイムで進捗表示
for await (const event of agentClient.runAgent({ input: "東京の天気" })) {
  if (event.type === 'TOOL_CALL_START') {
    console.log('🔧 天気API呼び出し中...');
  }
  if (event.type === 'TEXT_MESSAGE_CONTENT') {
    console.log('💬', event.content);
  }
}
```

**メリット**:
- ユーザーがリアルタイムで進捗を見れる
- AIの思考プロセスが透明
- Web開発者が慣れてるパターン（addEventListener感覚）

---

### 3. AG-UIの18種コアイベント

| カテゴリ | イベント数 | 具体例 |
|:---|:---:|:---|
| **Lifecycle** | 5種 | RUN_STARTED, RUN_FINISHED, STEP_STARTED... |
| **Text Message** | 4種 | TEXT_MESSAGE_START/CONTENT/END/CHUNK |
| **Tool Call** | 4種 | TOOL_CALL_START/ARGS/END/RESULT |
| **State** | 3種 | STATE_SNAPSHOT/DELTA/MESSAGES_SNAPSHOT |
| **Special** | 2種 | RAW/CUSTOM |

**重要パターン**:
1. **Start-Content-End**: テキストストリーミング
2. **Start-Args-End-Result**: ツール呼び出し可視化
3. **Snapshot-Delta**: 状態同期

---

## 🎯 決定事項

### アプリ仕様

**プロジェクト名**: 天気予報AIチャットボット

**目的**:
- AG-UIのイベント駆動アーキテクチャを体感
- ツール呼び出し可視化（TOOL_CALL系4イベント）
- ストリーミングテキスト表示

**主な機能**:
1. テキスト入力 → AI応答（ストリーミング）
2. 天気API呼び出し → 進捗可視化（"🔧 天気APIを呼び出し中..."）
3. 複数都市対応（東京、大阪、札幌を並行表示）
4. エラーハンドリング

---

### 技術スタック

| レイヤ | 選定技術 | 理由 |
|:---|:---|:---|
| **フレームワーク** | Next.js 15 (App Router) | フロント/バック統合、Vercel最適化 |
| **ランタイム** | Node.js 22 LTS | AG-UI完全互換、ライブラリ制限なし |
| **Vercel Runtime** | Node.js Runtime | Edge Functionsより確実（`runtime: 'nodejs'`） |
| **通信** | SSE (Server-Sent Events) | Next.js API Routeで簡単実装 |
| **AG-UIパッケージ** | @ag-ui/core 0.0.36<br>@ag-ui/client 0.0.37 | 公式パッケージ最新版 |
| **スタイリング** | Tailwind CSS | Next.jsデフォルト |
| **天気API** | モックAPI（学習用）| サインアップ不要、即動作 |
| **デプロイ** | Vercel 無料プラン | git push自動デプロイ、個人開発に十分 |

---

### Vercel vs Cloudflare Workers 比較結果

| 項目 | Vercel（採用） | Cloudflare Workers |
|:---|:---:|:---:|
| 実行時間制限 | 10秒（Node.js） | CPU時間10ms（厳しい） |
| Next.js対応 | ✅ ネイティブ | ⚠️ Pages経由（制限あり） |
| AG-UI互換性 | ✅ 確実 | ⚠️ 未検証 |
| デプロイ | git push自動 | wrangler CLI |
| 学習コスト | 低い | 中〜高 |

**結論**: 学習用途ならVercel、本番で高速化必要ならCloudflare検討

---

### Vercel 料金プラン

**無料プラン（Hobby）で十分な理由**:
- 天気予報処理は4-7秒で完結（10秒以内）
- 個人開発でリクエスト数少ない（月間3,000程度）
- 帯域幅100GBで余裕

**もし課金（$20/月 ≈ 3,000円）が必要な場合**:
- 「Function Timeout (10s)」エラーが出たら検討
- 学生なら GitHub Student Pack で1年無料

**判断フロー**: まず無料で試す → 問題なければ継続 → タイムアウトしたら課金

---

## 📁 プロジェクト構成（予定）

```
weather-chatbot-nextjs/
├── package.json
├── next.config.js
├── tsconfig.json
├── .env.local              # APIキー（後で設定）
├── app/
│   ├── layout.tsx          # ルートレイアウト
│   ├── page.tsx            # トップページ（チャットUI）
│   ├── api/
│   │   └── agent/
│   │       └── route.ts    # AG-UIエージェントエンドポイント (SSE)
│   └── globals.css
├── components/
│   ├── ChatUI.tsx          # チャット画面（クライアントコンポーネント）
│   ├── MessageList.tsx     # メッセージ一覧
│   ├── ToolCallDisplay.tsx # ツール実行表示
│   └── InputForm.tsx       # 入力フォーム
├── lib/
│   ├── agent.ts            # AG-UIエージェントロジック
│   └── tools/
│       └── weather.ts      # 天気取得ツール
└── public/
```

---

## 🚀 実装フェーズ（3段階）

### Phase 1: 最小構成（2-3時間）
**目標**: テキスト入力 → AI応答（ストリーミング）

**実装内容**:
- Reactで入力フォーム + メッセージ一覧
- AG-UI Clientで`runAgent()`呼び出し
- `TEXT_MESSAGE_CONTENT`イベントでストリーミング表示
- `RUN_STARTED/FINISHED`でローディング表示

**扱うイベント**: 5種
- RUN_STARTED
- RUN_FINISHED
- TEXT_MESSAGE_START
- TEXT_MESSAGE_CONTENT
- TEXT_MESSAGE_END

**完了条件（Done）**:
- [ ] ユーザーが「こんにちは」入力 → AIがストリーミングで応答
- [ ] `RUN_STARTED`で「⏳ 実行中...」表示
- [ ] `TEXT_MESSAGE_CONTENT`で1文字ずつ追加表示
- [ ] `RUN_FINISHED`でローディング非表示
- [ ] Chrome DevToolsでイベントログ確認

---

### Phase 2: ツール統合（+2-3時間）
**目標**: 天気APIを呼び出し、進捗を可視化

**実装内容**:
- バックエンドに`get_weather`ツール追加
- `TOOL_CALL_START`でツール名表示
- `TOOL_CALL_ARGS`で引数表示（city: "東京"）
- `TOOL_CALL_END`でローディング表示
- `TOOL_CALL_RESULT`で結果表示（晴れ、25℃）
- エラー時の`TOOL_CALL_ERROR`ハンドリング

**扱うイベント**: 9種（Phase 1の5種 + ツール4種）
- TOOL_CALL_START
- TOOL_CALL_ARGS
- TOOL_CALL_END
- TOOL_CALL_RESULT

**完了条件（Done）**:
- [ ] ユーザーが「東京の天気は？」入力 → AIが天気ツール呼び出し
- [ ] `TOOL_CALL_START`で「🔧 get_weatherを実行中...」表示
- [ ] `TOOL_CALL_ARGS`で「引数: { city: "東京" }」表示
- [ ] `TOOL_CALL_RESULT`で「結果: 晴れ、25℃」表示
- [ ] AIが結果を自然な文章で返答

---

### Phase 3: 応用機能（+1-2時間）
**目標**: 実務で使える機能を追加

**実装内容**:
- 複数都市の並行表示（「東京、大阪、札幌の天気」）
- 天気アイコン表示（☀️🌧️⛅）
- キャンセル機能（実行中にストップボタン）
- チャット履歴の永続化（localStorage）

**完了条件（Done）**:
- [ ] 「東京と大阪の天気」→ 2つのツール呼び出しを並行表示
- [ ] 天気アイコン表示（☀️/🌧️/⛅）
- [ ] ストップボタンでキャンセル可能
- [ ] リロード後もチャット履歴が残る

---

## 🛠️ 次回の作業手順（明日）

### Step 1: プロジェクト作成
```bash
# 1. Next.jsプロジェクト作成
npx create-next-app@latest weather-chatbot-nextjs

# 質問に答える:
# ✅ TypeScript: Yes
# ✅ ESLint: Yes
# ✅ Tailwind CSS: Yes
# ✅ src/ directory: No
# ✅ App Router: Yes
# ✅ Turbopack: Yes
# ✅ import alias: @/*

# 2. プロジェクトに移動
cd weather-chatbot-nextjs

# 3. AG-UI関連パッケージインストール
npm install @ag-ui/core @ag-ui/client

# 4. 開発サーバー起動確認
npm run dev
# → http://localhost:3000 で確認
```

---

### Step 2: API Route作成（`app/api/agent/route.ts`）

```typescript
import { NextRequest } from 'next/server';

// Node.js runtimeを使用
export const runtime = 'nodejs';

export async function POST(req: NextRequest) {
  const { input } = await req.json();

  // SSEストリーム作成
  const stream = new ReadableStream({
    async start(controller) {
      const encoder = new TextEncoder();

      function sendEvent(event: any) {
        const data = `data: ${JSON.stringify(event)}\n\n`;
        controller.enqueue(encoder.encode(data));
      }

      // Phase 1: 簡易応答（後でAG-UIエージェントに置き換え）
      sendEvent({ type: 'RUN_STARTED', id: 'run_123' });

      sendEvent({ type: 'TEXT_MESSAGE_START', id: 'msg_123' });

      const words = ['こんにちは！', '天気予報', 'チャットボット', 'です'];
      for (const word of words) {
        await new Promise(resolve => setTimeout(resolve, 200));
        sendEvent({
          type: 'TEXT_MESSAGE_CONTENT',
          id: 'msg_123',
          content: word
        });
      }

      sendEvent({ type: 'TEXT_MESSAGE_END', id: 'msg_123' });
      sendEvent({ type: 'RUN_FINISHED', id: 'run_123' });

      controller.close();
    }
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  });
}
```

---

### Step 3: チャットUIコンポーネント作成（`components/ChatUI.tsx`）

```typescript
'use client';

import { useState } from 'react';

export default function ChatUI() {
  const [messages, setMessages] = useState<string[]>([]);
  const [input, setInput] = useState('');
  const [isRunning, setIsRunning] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (!input.trim()) return;

    setIsRunning(true);
    setMessages(prev => [...prev, `👤 あなた: ${input}`, '🤖 AI: ']);

    const response = await fetch('/api/agent', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ input }),
    });

    const reader = response.body?.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader!.read();
      if (done) break;

      const chunk = decoder.decode(value);
      const lines = chunk.split('\n');

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const event = JSON.parse(line.slice(6));

          switch (event.type) {
            case 'TEXT_MESSAGE_CONTENT':
              setMessages(prev => {
                const last = prev[prev.length - 1];
                return [...prev.slice(0, -1), last + event.content];
              });
              break;

            case 'RUN_FINISHED':
              setIsRunning(false);
              break;
          }
        }
      }
    }

    setInput('');
  }

  return (
    <div className="flex flex-col h-screen max-w-2xl mx-auto p-4">
      <h1 className="text-2xl font-bold mb-4">🌤️ 天気予報AIチャットボット</h1>

      <div className="flex-1 overflow-y-auto mb-4 space-y-2">
        {messages.map((msg, i) => (
          <div key={i} className="p-2 bg-gray-100 rounded">
            {msg}
          </div>
        ))}
      </div>

      <form onSubmit={handleSubmit} className="flex gap-2">
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="東京の天気は？"
          disabled={isRunning}
          className="flex-1 p-2 border rounded"
        />
        <button
          type="submit"
          disabled={isRunning}
          className="px-4 py-2 bg-blue-500 text-white rounded disabled:bg-gray-300"
        >
          {isRunning ? '⏳' : '送信'}
        </button>
      </form>
    </div>
  );
}
```

---

### Step 4: トップページ更新（`app/page.tsx`）

```typescript
import ChatUI from '@/components/ChatUI';

export default function Home() {
  return <ChatUI />;
}
```

---

### Step 5: 動作確認

```bash
npm run dev
```

ブラウザで `http://localhost:3000` にアクセス:
- ✅ 「こんにちは」入力 → 送信
- ✅ 「こんにちは！天気予報チャットボットです」がストリーミング表示
- ✅ ローディング表示（⏳）が出る

---

## 📖 参考資料

### AG-UI公式ドキュメント
- 公式サイト: https://docs.ag-ui.com/
- GitHub: https://github.com/ag-ui-protocol/ag-ui
- イベント仕様: https://docs.ag-ui.com/concepts/events
- アーキテクチャ: https://docs.ag-ui.com/concepts/architecture

### Node.js バージョン情報
- 推奨: Node.js 22 LTS（2024年10月29日昇格、コードネーム'Jod'）
- Active LTS期間: 2025年10月21日まで
- Maintenance LTS期間: 2027年4月30日まで
- Node.js 18: 2025年4月30日にEOL到達済み

### Stack Profile
- 保存場所: `/workspace/stack-profiles/ag-ui-protocol.json`
- バージョン: 0.0.36-37

---

## ✅ 理解チェック（3問）

1. **AG-UIの最大の価値は？**
   - 答え: AIフレームワーク非依存の統一プロトコル。バックエンド（LangGraph/Pydantic AI等）を変えてもフロントエンドコードが不変。

2. **イベント駆動アーキテクチャの利点は？**
   - 答え: リアルタイムで進捗表示、AIの思考プロセスが透明、Web開発者に馴染みがある（addEventListener感覚）。

3. **TOOL_CALLイベントの4段階は？**
   - 答え: TOOL_CALL_START（開始） → TOOL_CALL_ARGS（引数） → TOOL_CALL_END（終了） → TOOL_CALL_RESULT（結果）

---

## 🎯 次の一手

**明日の最初のタスク**:
```bash
npx create-next-app@latest weather-chatbot-nextjs
```

Phase 1（ストリーミングテキスト表示）を実装して動かす！

---

**作成日**: 2025年10月9日
**次回予定**: Phase 1実装 → Phase 2実装 → Vercelデプロイ
**推定所要時間**: Phase 1-2で合計4-6時間
