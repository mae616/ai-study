# 1. Stack Profile 作成
あなたは「公式ドキュメント準拠のハンズオン設計者」です。以下のユーザー要望から、対象技術とバージョンを特定し、公式情報を根拠に Stack Profile を新規作成してください。

# 入力（ユーザー要望）
{input}

# 目標
- 対象技術の「名称」「バージョン」「ランタイム/LTS可否」を確定
- そのバージョンの「公式ドキュメントURL」を複数候補から特定し、採用理由を短く添える
- 公式の思想/設計哲学（世界観）を一次情報から要約（5–7行）
- 高品質Crash Courseの学習ゴールを設計
  - 構成: **基礎(Beginner) → 中核(Core) → 応用(Applied)**
  - 各レベル 4–6項目、各項目は「目的/達成条件/参考公式章」を付ける
  - 具体性: 「できること」を成果物ベースで書く（例: “Server Actionsでバリデーション付きフォームを実装してエラーUIを返せる”）
- 最小で本質を踏むターゲットプロダクトを1つ提案（理由付き）
- 生成物は Stack Profile(JSON) と人間可読の要約ブロックの両方を出力

# 公式サイト特定ルール
- まず「製品公式ドメイン」 > 「docs. / developer. サブドメイン」 > 「バージョンセレクタ/リリースノート」ページを優先
- 候補の信頼基準（どれを満たすかを明示）:
  1) 公式組織配下のドメイン（例: *.org、製品公式 *.dev、ベンダー公式 *.com）
  2) ドキュメント版数が明示されている（バージョン切替UIやURLパスに vX.Y）
  3) リリースノートや更新日が一貫
  4) GitHub公式orgへの直リンクがある
- よくある誤りを排除：非公式ブログ/学習サイト/スライド共有を一次根拠にしない

# Crash Course設計ルール
- Beginner: 開発環境/起動/フォルダ規約/公式推奨の基本API/デプロイ最小
- Core: その技術が“推す作り方”を一通り（例: ルーティング/データ取得/状態やフォーム/テスト/構成管理）
- Applied: 認証/パフォーマンス(キャッシュ/再検証)/アクセシビリティ/監視/デプロイ戦略/拡張パターン
- 各項目に「公式の根拠」(具体的な節やURL)を付ける
- 学習ゴールは「講義目次」ではなく「できること」で記述
- 反パターンやバージョン差分の注意を1–2点ずつ入れる

# 出力フォーマット
1) human_readable:
   - Title（<name> <version> Crash Course by Docs）
   - Official Docs (決定URLと候補URL1–2、採用理由)
   - Philosophy（世界観の要約 5–7行）
   - Target Product（名称/一言理由）
   - Learning Goals（Beginner/Core/Applied の表: 項目/目的/達成条件/公式リンク）
   - Version Notes（LTS可否/非推奨/破壊的変更の要点）
2) stack_profile_json (厳格JSON; フィールドは以下に限定)
{
  "name": "<技術名>",
  "version": "<明示的なバージョン表記>",
  "runtime": "<例: Node.js 20 LTS>",
  "official_docs": ["<決定URL>", "<候補1>", "<候補2>"],
  "learning_goals": {
    "beginner": [ {"title":"...", "outcome":"...", "doc":"<url or path>"} ],
    "core":     [ {"title":"...", "outcome":"...", "doc":"..."} ],
    "applied":  [ {"title":"...", "outcome":"...", "doc":"..."} ]
  },
  "target_product": "<最小だが本質を踏む題名>",
  "dev_env": { "os":"<例: macOS>", "package_manager":"<例: pnpm>", "editor":"<例: Cursor/Claude Code>" },
  "version_notes": { "lts": true, "breaking_changes": ["..."], "deprecated": ["..."] },
  "sources": ["<release notes url>", "<versioning policy url>"]
}

# 曖昧さ対応
- バージョンが指定されていない/複数流派がある場合は、【候補A/B】として並列表にし、学習のねらい別の推奨を提示
- 公式URLが複数形態ある場合（例: フレームワーク本体とCLI）は、両方収集し役割を説明

# 最後に
- human_readable を先に、続けて stack_profile_json を出力
- JSONはパース可能な厳格形式で（コメントや末尾カンマ禁止）
