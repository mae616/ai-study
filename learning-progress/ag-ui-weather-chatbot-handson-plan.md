# 🎯 AG-UI 天気予報チャットボット ハンズオン計画

## プロダクト概要

**目的**: AG-UIのイベント駆動アーキテクチャを体感し、ツール呼び出し可視化（TOOL_CALL系4イベント）とストリーミングテキスト表示を習得するにゃ！

**最終成果物**: Next.js + AG-UI Protocolで構築した天気予報チャットボット（Vercelデプロイ可能）

**推定所要時間**: 6-8時間（3セッションに分割推奨）

---

## ステップ1: 環境構築とプロジェクト作成

### 🎯 目的（思想上の意味）
AG-UIはHTTPのAI版。HTTPがWebサーバーを抽象化したように、AG-UIがAIエージェントを抽象化する。まずは**最小の骨組み**を用意して「動く基盤」を作るにゃ！

### 📋 実施手順

```bash
# 1. Next.jsプロジェクト作成
npx create-next-app@latest weather-chatbot-nextjs

# 質問への回答:
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

# 4. Node.jsバージョン確認
node -v
# → v22.x.x であることを確認（推奨: 22 LTS）

# 5. 開発サーバー起動確認
npm run dev
```

### ✅ 検証方法

1. ブラウザで `http://localhost:3000` にアクセス
2. Next.jsのデフォルトページが表示される
3. `package.json` に `@ag-ui/core` と `@ag-ui/client` が追加されている

### ⚠️ 失敗しやすい点と回避策

1. **Node.jsバージョンが18未満**
   - 🛡️ 回避策: `nvm install 22 && nvm use 22` でNode.js 22に切り替え

2. **Turbopackでエラー発生**
   - 🛡️ 回避策: `package.json` の `dev` スクリプトから `--turbo` を削除

3. **ポート3000が使用中**
   - 🛡️ 回避策: `npm run dev -- -p 3001` で別ポートを使用

### ✨ Done条件（観測可能な合格基準）

- [ ] `npm run dev` でエラーなく起動
- [ ] `http://localhost:3000` でNext.jsデフォルトページ表示
- [ ] `node -v` がv22.x.xを返す
- [ ] `package.json` に `@ag-ui/core`, `@ag-ui/client` が存在

---

## ステップ2: SSEストリーム基盤の実装（AG-UIイベント送信）

### 🎯 目的（思想上の意味）
AG-UIはイベント駆動アーキテクチャ。従来の「命令→待機→結果」ではなく、「イベントをリアルタイムでストリーム送信」する。まずは**最小の5イベント**（RUN_STARTED/FINISHED、TEXT_MESSAGE系3種）を実装して、思想を体感するにゃ！

### 📋 実施手順

#### 2-1. API Route作成（`app/api/agent/route.ts`）

```typescript
import { NextRequest } from 'next/server';

// Node.js runtimeを使用（Vercelで確実に動作）
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

      // AG-UIイベント送信（Phase 1: 最小構成）
      sendEvent({ type: 'RUN_STARTED', id: 'run_123' });

      sendEvent({ type: 'TEXT_MESSAGE_START', id: 'msg_123' });

      // ストリーミングテキスト送信
      const words = ['こんにちは！', '天気予報', 'チャットボット', 'です'];
      for (const word of words) {
        await new Promise(resolve => setTimeout(resolve, 200));
        sendEvent({
          type: 'TEXT_MESSAGE_CONTENT',
          id: 'msg_123',
          content: word + ' '
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

### ✅ 検証方法

```bash
# curlでAPI Route単体テスト
curl -N -X POST http://localhost:3000/api/agent \
  -H "Content-Type: application/json" \
  -d '{"input":"こんにちは"}'

# 期待される出力:
# data: {"type":"RUN_STARTED","id":"run_123"}
# data: {"type":"TEXT_MESSAGE_START","id":"msg_123"}
# data: {"type":"TEXT_MESSAGE_CONTENT","id":"msg_123","content":"こんにちは！ "}
# ...
# data: {"type":"RUN_FINISHED","id":"run_123"}
```

### ⚠️ 失敗しやすい点と回避策

1. **SSE形式の改行コード間違い**
   - 🛡️ 回避策: `data: ${JSON.stringify(event)}\n\n` の `\n\n`（改行2つ）を厳守

2. **Vercel Edgeで動かない**
   - 🛡️ 回避策: `export const runtime = 'nodejs'` を明記

3. **ストリームが途中で切れる**
   - 🛡️ 回避策: `controller.close()` を最後に必ず呼ぶ

### ✨ Done条件

- [ ] `curl` でAPI Routeを叩いてイベントが正しく返る
- [ ] `RUN_STARTED` → `TEXT_MESSAGE_*` → `RUN_FINISHED` の順序が正しい
- [ ] Chrome DevToolsのNetworkタブで `text/event-stream` が確認できる

---

## ステップ3: チャットUIの実装（イベント受信とストリーミング表示）

### 🎯 目的（思想上の意味）
イベント駆動UIの核心は「状態をイベントごとに更新」すること。従来の「結果を待って一括表示」ではなく、「TEXT_MESSAGE_CONTENTが来るたびに1文字ずつ追加」するにゃ。これがAG-UIの**リアルタイム性**の源にゃ！

### 📋 実施手順

#### 3-1. チャットUIコンポーネント作成（`components/ChatUI.tsx`）

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

#### 3-2. トップページ更新（`app/page.tsx`）

```typescript
import ChatUI from '@/components/ChatUI';

export default function Home() {
  return <ChatUI />;
}
```

### ✅ 検証方法

1. `npm run dev` でサーバー起動
2. `http://localhost:3000` にアクセス
3. 「こんにちは」と入力して送信
4. 「こんにちは！ 天気予報 チャットボット です」がストリーミング表示される
5. ローディング表示（⏳）が出る

### ⚠️ 失敗しやすい点と回避策

1. **ストリーミング表示がカクつく**
   - 🛡️ 回避策: `setTimeout` の間隔を調整（200ms → 100ms）

2. **最後のメッセージが空になる**
   - 🛡️ 回避策: `TEXT_MESSAGE_END` で空文字チェック追加

3. **イベントパース時にエラー**
   - 🛡️ 回避策: `try-catch` で `JSON.parse` を囲む

### ✨ Done条件

- [ ] ユーザー入力 → ストリーミング表示が動作
- [ ] `TEXT_MESSAGE_CONTENT` ごとに1文字ずつ追加される
- [ ] `RUN_FINISHED` で送信ボタンが再度有効になる
- [ ] Chrome DevToolsでイベントログ確認（Console）

---

## ステップ4: 天気取得ツールの実装（バックエンド）

### 🎯 目的（思想上の意味）
AG-UIの最大の価値は**ツール呼び出しの可視化**。従来は「天気APIを叩いて結果を返すだけ」だったが、AG-UIでは「TOOL_CALL_START（開始）→ TOOL_CALL_ARGS（引数表示）→ TOOL_CALL_END（終了）→ TOOL_CALL_RESULT（結果）」の4段階でプロセスを透明化するにゃ！

### 📋 実施手順

#### 4-1. 天気取得ツール作成（`lib/tools/weather.ts`）

```typescript
// モックAPI（学習用）
export async function getWeather(city: string): Promise<string> {
  // 実際の天気APIを使う場合はここをOpenWeatherMap等に置き換え
  await new Promise(resolve => setTimeout(resolve, 1500)); // API呼び出しをシミュレート

  const weatherData: Record<string, string> = {
    '東京': '☀️ 晴れ、気温25℃',
    '大阪': '⛅ 曇り、気温23℃',
    '札幌': '🌧️ 雨、気温18℃',
  };

  return weatherData[city] || '❓ データなし';
}
```

#### 4-2. API Routeをツール統合版に更新（`app/api/agent/route.ts`）

```typescript
import { NextRequest } from 'next/server';
import { getWeather } from '@/lib/tools/weather';

export const runtime = 'nodejs';

export async function POST(req: NextRequest) {
  const { input } = await req.json();

  const stream = new ReadableStream({
    async start(controller) {
      const encoder = new TextEncoder();

      function sendEvent(event: any) {
        const data = `data: ${JSON.stringify(event)}\n\n`;
        controller.enqueue(encoder.encode(data));
      }

      sendEvent({ type: 'RUN_STARTED', id: 'run_123' });

      // 簡易的な天気クエリ判定
      const cityMatch = input.match(/(東京|大阪|札幌)/);
      if (cityMatch) {
        const city = cityMatch[1];

        // ツール呼び出しイベント送信
        sendEvent({ type: 'TOOL_CALL_START', id: 'tool_123', name: 'get_weather' });
        sendEvent({ type: 'TOOL_CALL_ARGS', id: 'tool_123', args: { city } });

        // 実際にツール実行
        const result = await getWeather(city);

        sendEvent({ type: 'TOOL_CALL_END', id: 'tool_123' });
        sendEvent({ type: 'TOOL_CALL_RESULT', id: 'tool_123', result });

        // AIの応答
        sendEvent({ type: 'TEXT_MESSAGE_START', id: 'msg_123' });
        const response = `${city}の天気は${result}です！`;
        for (const char of response) {
          await new Promise(resolve => setTimeout(resolve, 50));
          sendEvent({ type: 'TEXT_MESSAGE_CONTENT', id: 'msg_123', content: char });
        }
        sendEvent({ type: 'TEXT_MESSAGE_END', id: 'msg_123' });
      } else {
        // 天気以外のクエリ
        sendEvent({ type: 'TEXT_MESSAGE_START', id: 'msg_123' });
        const response = '都市名（東京、大阪、札幌）を教えてください！';
        for (const char of response) {
          await new Promise(resolve => setTimeout(resolve, 50));
          sendEvent({ type: 'TEXT_MESSAGE_CONTENT', id: 'msg_123', content: char });
        }
        sendEvent({ type: 'TEXT_MESSAGE_END', id: 'msg_123' });
      }

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

### ✅ 検証方法

```bash
# curlでAPI Route単体テスト
curl -N -X POST http://localhost:3000/api/agent \
  -H "Content-Type: application/json" \
  -d '{"input":"東京の天気は？"}'

# 期待される出力:
# data: {"type":"RUN_STARTED",...}
# data: {"type":"TOOL_CALL_START","id":"tool_123","name":"get_weather"}
# data: {"type":"TOOL_CALL_ARGS","id":"tool_123","args":{"city":"東京"}}
# data: {"type":"TOOL_CALL_END","id":"tool_123"}
# data: {"type":"TOOL_CALL_RESULT","id":"tool_123","result":"☀️ 晴れ、気温25℃"}
# ...
```

### ⚠️ 失敗しやすい点と回避策

1. **ツール実行中にタイムアウト**
   - 🛡️ 回避策: `getWeather()` のsleepを短くする（1500ms → 500ms）

2. **正規表現で都市名抽出失敗**
   - 🛡️ 回避策: `input.toLowerCase()` で正規化してから判定

3. **TOOL_CALL_RESULTのデータ形式ミス**
   - 🛡️ 回避策: `result` はstring型で渡す（オブジェクトの場合は `JSON.stringify` が必要）

### ✨ Done条件

- [ ] 「東京の天気は？」入力 → `TOOL_CALL_*` イベントが4つ順番に送信される
- [ ] `getWeather()` が1.5秒後に結果を返す
- [ ] curlで全イベントが正しく出力される

---

## ステップ5: ツール呼び出しUI表示（フロントエンド）

### 🎯 目的（思想上の意味）
イベント駆動の真価は「プロセスの可視化」。従来は「天気APIを呼び出し中...」と曖昧なローディングだったが、AG-UIでは「🔧 get_weatherを実行中... 引数: {city: "東京"}」と**具体的な進捗を表示**できるにゃ！

### 📋 実施手順

#### 5-1. ツール呼び出し表示コンポーネント作成（`components/ToolCallDisplay.tsx`）

```typescript
'use client';

interface ToolCall {
  id: string;
  name: string;
  args?: any;
  result?: string;
  status: 'running' | 'completed';
}

export default function ToolCallDisplay({ toolCall }: { toolCall: ToolCall }) {
  return (
    <div className="p-3 bg-blue-50 border border-blue-200 rounded">
      <div className="font-semibold text-blue-800">
        🔧 {toolCall.name}
        {toolCall.status === 'running' && ' (実行中...)'}
      </div>
      {toolCall.args && (
        <div className="text-sm text-gray-600 mt-1">
          引数: {JSON.stringify(toolCall.args, null, 2)}
        </div>
      )}
      {toolCall.result && (
        <div className="text-sm text-green-700 mt-1 font-medium">
          結果: {toolCall.result}
        </div>
      )}
    </div>
  );
}
```

#### 5-2. ChatUIコンポーネント更新（`components/ChatUI.tsx`）

```typescript
'use client';

import { useState } from 'react';
import ToolCallDisplay from './ToolCallDisplay';

interface Message {
  type: 'user' | 'ai' | 'tool';
  content: string;
  toolCall?: {
    id: string;
    name: string;
    args?: any;
    result?: string;
    status: 'running' | 'completed';
  };
}

export default function ChatUI() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [isRunning, setIsRunning] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (!input.trim()) return;

    setIsRunning(true);
    setMessages(prev => [
      ...prev,
      { type: 'user', content: input },
      { type: 'ai', content: '' }
    ]);

    const response = await fetch('/api/agent', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ input }),
    });

    const reader = response.body?.getReader();
    const decoder = new TextDecoder();
    let currentToolCallId: string | null = null;

    while (true) {
      const { done, value } = await reader!.read();
      if (done) break;

      const chunk = decoder.decode(value);
      const lines = chunk.split('\n');

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          try {
            const event = JSON.parse(line.slice(6));

            switch (event.type) {
              case 'TOOL_CALL_START':
                currentToolCallId = event.id;
                setMessages(prev => [
                  ...prev,
                  {
                    type: 'tool',
                    content: '',
                    toolCall: {
                      id: event.id,
                      name: event.name,
                      status: 'running'
                    }
                  }
                ]);
                break;

              case 'TOOL_CALL_ARGS':
                setMessages(prev => prev.map(msg =>
                  msg.toolCall?.id === event.id
                    ? { ...msg, toolCall: { ...msg.toolCall, args: event.args } }
                    : msg
                ));
                break;

              case 'TOOL_CALL_RESULT':
                setMessages(prev => prev.map(msg =>
                  msg.toolCall?.id === event.id
                    ? {
                        ...msg,
                        toolCall: {
                          ...msg.toolCall,
                          result: event.result,
                          status: 'completed'
                        }
                      }
                    : msg
                ));
                break;

              case 'TEXT_MESSAGE_CONTENT':
                setMessages(prev => {
                  const lastAiIndex = prev.findLastIndex(m => m.type === 'ai');
                  if (lastAiIndex !== -1) {
                    const updated = [...prev];
                    updated[lastAiIndex] = {
                      ...updated[lastAiIndex],
                      content: updated[lastAiIndex].content + event.content
                    };
                    return updated;
                  }
                  return prev;
                });
                break;

              case 'RUN_FINISHED':
                setIsRunning(false);
                break;
            }
          } catch (err) {
            console.error('イベントパースエラー:', err);
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
          <div key={i}>
            {msg.type === 'user' && (
              <div className="p-2 bg-blue-100 rounded">
                👤 あなた: {msg.content}
              </div>
            )}
            {msg.type === 'ai' && (
              <div className="p-2 bg-gray-100 rounded">
                🤖 AI: {msg.content}
              </div>
            )}
            {msg.type === 'tool' && msg.toolCall && (
              <ToolCallDisplay toolCall={msg.toolCall} />
            )}
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

### ✅ 検証方法

1. `npm run dev` で起動
2. 「東京の天気は？」入力
3. 以下の表示順序を確認:
   - `👤 あなた: 東京の天気は？`
   - `🔧 get_weather (実行中...)`
   - `引数: { "city": "東京" }`
   - `結果: ☀️ 晴れ、気温25℃`
   - `🤖 AI: 東京の天気は☀️ 晴れ、気温25℃です！`

### ⚠️ 失敗しやすい点と回避策

1. **ツール表示が2重になる**
   - 🛡️ 回避策: `TOOL_CALL_START` で新規追加、`TOOL_CALL_ARGS/RESULT` で更新のみ

2. **AIメッセージが空のまま**
   - 🛡️ 回避策: `TEXT_MESSAGE_START` で空のAIメッセージを追加しておく

3. **イベント順序が崩れる**
   - 🛡️ 回避策: Chrome DevToolsで `event.id` を確認してマッチング精度を上げる

### ✨ Done条件

- [ ] ツール呼び出しが青い枠で表示される
- [ ] 引数と結果が別々に表示される
- [ ] AIの最終応答がストリーミング表示される
- [ ] 複数の都市（東京、大阪、札幌）で動作確認

---

## ステップ6: エラーハンドリングとエッジケース対応

### 🎯 目的（思想上の意味）
プロダクションレベルのアプリは「正常系だけ動く」では不十分。AG-UIでは`RUN_ERROR`イベントを使って**エラーもストリームで送信**し、ユーザーに透明性を提供するにゃ！

### 📋 実施手順

#### 6-1. API Routeにエラーハンドリング追加（`app/api/agent/route.ts`）

```typescript
// ... 既存のコード ...

async start(controller) {
  // ... 既存のコード ...

  try {
    sendEvent({ type: 'RUN_STARTED', id: 'run_123' });

    // ... ツール呼び出しロジック ...

    sendEvent({ type: 'RUN_FINISHED', id: 'run_123' });
  } catch (error) {
    // エラーイベント送信
    sendEvent({
      type: 'RUN_ERROR',
      id: 'run_123',
      error: {
        message: error instanceof Error ? error.message : 'Unknown error',
        code: 'EXECUTION_ERROR'
      }
    });
  } finally {
    controller.close();
  }
}
```

#### 6-2. ChatUIにエラー表示追加（`components/ChatUI.tsx`）

```typescript
// ... switchケースに追加 ...

case 'RUN_ERROR':
  setMessages(prev => [
    ...prev,
    {
      type: 'ai',
      content: `❌ エラーが発生しました: ${event.error.message}`
    }
  ]);
  setIsRunning(false);
  break;
```

#### 6-3. バリデーション追加

```typescript
// API Routeの冒頭に追加
const { input } = await req.json();
if (!input || typeof input !== 'string') {
  return new Response(
    JSON.stringify({ error: 'Invalid input' }),
    { status: 400 }
  );
}
```

### ✅ 検証方法

1. 空文字入力 → エラーメッセージ表示
2. APIタイムアウトシミュレーション → エラーハンドリング
3. Chrome DevToolsでNetworkエラーを確認

### ⚠️ 失敗しやすい点と回避策

1. **エラー時にストリームが開いたまま**
   - 🛡️ 回避策: `finally` で `controller.close()` を必ず呼ぶ

2. **エラーメッセージがユーザーに優しくない**
   - 🛡️ 回避策: エラーコードをマップして日本語メッセージに変換

3. **ネットワークエラーで無限ローディング**
   - 🛡️ 回避策: フロントエンドで `AbortController` + タイムアウト実装

### ✨ Done条件

- [ ] 空文字入力でエラーメッセージ表示
- [ ] `RUN_ERROR` イベントがChrome DevToolsで確認できる
- [ ] ネットワークエラー時に自動的にローディング終了

---

## ステップ7: Vercelデプロイと動作確認

### 🎯 目的（思想上の意味）
ローカルで動くだけでは意味がない。Vercelに**本番デプロイ**して「世界中からアクセス可能なAG-UIアプリ」にするにゃ！ここでNode.js Runtimeの設定が重要にゃ！

### 📋 実施手順

#### 7-1. Vercel CLIインストール

```bash
npm install -g vercel
```

#### 7-2. Vercelにログイン

```bash
vercel login
```

#### 7-3. デプロイ

```bash
# プロジェクトルートで実行
vercel

# 質問への回答:
# Set up and deploy "...": Yes
# Which scope: (Your account)
# Link to existing project: No
# Project name: weather-chatbot-nextjs
# In which directory is your code located: ./
# Want to override the settings: No
```

#### 7-4. プロダクションデプロイ

```bash
vercel --prod
```

### ✅ 検証方法

1. Vercelから発行されたURL（例: `https://weather-chatbot-nextjs.vercel.app`）にアクセス
2. 「東京の天気は？」入力
3. ツール呼び出しとストリーミング表示が動作
4. Chrome DevToolsでSSEストリームを確認
5. スマホからもアクセスして動作確認

### ⚠️ 失敗しやすい点と回避策

1. **Edge Functionsでタイムアウト**
   - 🛡️ 回避策: `export const runtime = 'nodejs'` が正しく設定されているか確認

2. **環境変数未設定でAPIキーエラー**（実際のAPIを使う場合）
   - 🛡️ 回避策: Vercel Dashboardで `Environment Variables` を設定

3. **デプロイ後に404エラー**
   - 🛡️ 回避策: `vercel --prod` を実行してプロダクションビルド確認

### ✨ Done条件

- [ ] Vercel URLでアプリが正常動作
- [ ] 天気クエリが3つ（東京、大阪、札幌）すべて動作
- [ ] Chrome DevToolsでSSEストリームが10秒以内に完了
- [ ] スマホからもアクセス可能

---

## ステップ8: 応用機能追加（複数都市並行表示）

### 🎯 目的（思想上の意味）
AG-UIは**並行ツール呼び出し**に対応。「東京と大阪の天気」と入力すると、2つの`TOOL_CALL`を同時に実行し、進捗を並行表示できるにゃ！これが従来のRPCとの決定的な違いにゃ！

### 📋 実施手順

#### 8-1. API Routeを複数都市対応に更新（`app/api/agent/route.ts`）

```typescript
// 複数都市を検出
const cities = ['東京', '大阪', '札幌'].filter(city => input.includes(city));

if (cities.length > 0) {
  // 並行ツール呼び出し
  const toolPromises = cities.map(async (city, index) => {
    const toolId = `tool_${index}`;

    sendEvent({ type: 'TOOL_CALL_START', id: toolId, name: 'get_weather' });
    sendEvent({ type: 'TOOL_CALL_ARGS', id: toolId, args: { city } });

    const result = await getWeather(city);

    sendEvent({ type: 'TOOL_CALL_END', id: toolId });
    sendEvent({ type: 'TOOL_CALL_RESULT', id: toolId, result });

    return { city, result };
  });

  const results = await Promise.all(toolPromises);

  // AIの応答
  sendEvent({ type: 'TEXT_MESSAGE_START', id: 'msg_123' });
  const response = results.map(r => `${r.city}: ${r.result}`).join('\n');
  for (const char of response) {
    await new Promise(resolve => setTimeout(resolve, 30));
    sendEvent({ type: 'TEXT_MESSAGE_CONTENT', id: 'msg_123', content: char });
  }
  sendEvent({ type: 'TEXT_MESSAGE_END', id: 'msg_123' });
}
```

### ✅ 検証方法

1. 「東京と大阪の天気」入力
2. 2つのツール呼び出しが並行表示される
3. 結果が両方表示される

### ⚠️ 失敗しやすい点と回避策

1. **ツールIDが重複**
   - 🛡️ 回避策: `tool_${index}` でユニークIDを生成

2. **Promise.allで1つ失敗すると全部失敗**
   - 🛡️ 回避策: `Promise.allSettled` を使う

3. **UIで順序が崩れる**
   - 🛡️ 回避策: `toolCall.id` でソート

### ✨ Done条件

- [ ] 「東京と大阪の天気」→ 2つのツール表示
- [ ] 「東京、大阪、札幌の天気」→ 3つのツール表示
- [ ] 各ツールの進捗が個別に表示される

---

## 📚 全体の振り返りチェックリスト

### 理解チェック（5問）

1. **AG-UIの核心的価値は？**
   - 答え: AIフレームワーク非依存の統一プロトコル。HTTPがWebサーバーを抽象化したように、AG-UIがAIエージェントを抽象化。

2. **イベント駆動の利点は？**
   - 答え: リアルタイム進捗表示、AIの思考プロセス透明化、Web開発者に馴染みがある。

3. **TOOL_CALLの4段階は？**
   - 答え: TOOL_CALL_START → TOOL_CALL_ARGS → TOOL_CALL_END → TOOL_CALL_RESULT

4. **SSEとWebSocketの違いは？**
   - 答え: SSEは単方向（サーバー→クライアント）、WebSocketは双方向。AG-UIでは単純なクエリならSSEで十分。

5. **Node.js Runtimeを使う理由は？**
   - 答え: Edge Functionsは実行時間制限が厳しい（CPU時間10ms）。AG-UIはストリーミングで数秒かかるためNode.js Runtime（10秒制限）が必要。

### 次の一手

**学習の深化**:
- LangGraphやPydantic AIと統合してみる（公式Middleware使用）
- STATE_SNAPSHOT/DELTAでチャット履歴を共有ステート化
- CUSTOMイベントで動的UI生成（天気アイコンカード等）

**プロダクション対応**:
- 認証フロー追加（Clerk/NextAuth）
- ネットワークエラー時の自動再接続
- Observability（OpenTelemetry統合）

---

## 🎉 完成イメージ

```
🌤️ 天気予報AIチャットボット
┌─────────────────────────────┐
│ 👤 あなた: 東京の天気は？      │
│ 🔧 get_weather (実行中...)    │
│   引数: { "city": "東京" }     │
│   結果: ☀️ 晴れ、気温25℃      │
│ 🤖 AI: 東京の天気は☀️ 晴れ... │
└─────────────────────────────┘
[東京の天気は？        ] [⏳]
```

---

これでAG-UIのイベント駆動アーキテクチャを完全マスターできるにゃ！ 🎯✨

ユーザーちゃん、準備できたらステップ1から始めるにゃ〜！ 🚀

---

**作成日**: 2025年10月10日
**推定所要時間**: 6-8時間（3セッション推奨）
**前提知識**: Next.js基礎、React基礎、TypeScript基礎
**参考資料**: [AG-UI公式ドキュメント](https://docs.ag-ui.com/)
