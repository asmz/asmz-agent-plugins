---
name: maestro-flutter-agent
description: >
  Agent skill that autonomously generates, runs, debugs, and fixes Maestro test flows (YAML) for Flutter mobile apps.
  Given a scenario in natural language, it reads the Flutter widget code, generates the matching Maestro YAML,
  runs it via the Maestro MCP, and iterates on failures until the flow passes (or escalates if it cannot resolve).
when_to_use: >
  Use when the user wants to write, run, debug, or fix Maestro E2E test flows for a Flutter mobile app —
  including creating new flows, modifying existing ones, adding steps, handling test failures,
  OS or in-app dialogs causing flakiness, or building out a regression suite.

  日本語の依頼例: 「Maestroのフローを作って」「画面遷移のテストを自動化したい」「UIテストが欲しい」
  「このフローを直して」「ステップを追加して」「テストが失敗するから対処して」「OSダイアログで止まる」
  「Maestroフローがフレーキー」「回帰テストを整備したい」など。
  新規フロー作成・既存フロー修正・ステップ追加・失敗対処・OSダイアログやアプリ内ダイアログ起因のフレーキー対応・回帰テスト整備が対象。
---

# Maestro Flutter Agent

Maestro MCP を活用して、Flutterモバイルアプリのテストフロー（YAML）を
自律的に生成・実行・修正するエージェントスキル。

## 起動時の最初の操作

このスキルが起動したら、他のいかなる操作よりも前に `mcp__maestro__list_devices` を最初に呼び出す。
ファイル読み取り（`Read` / `Glob` / `Grep`）、コード解析、YAML 生成、TodoWrite による計画立案など、
MCP 接続が確認できるまでは他の作業に着手しない。

タスクを Todo に分解する場合も、最初の Todo は「MCP接続チェック」のみを単独で置き、
コード解析やフロー設計と同じ Todo にまとめない。

### MCP が利用不可だった場合

- ユーザーに「Maestro MCP サーバに接続できません。`/mcp` から `maestro` サーバを再接続してください」と依頼する
- 再接続完了の確認を得るまで、他の作業を進めない
- ユーザーへの案内は MCP 再接続手順のみにとどめる。`maestro test` などの CLI 実行コマンドを代替案として提示しない（CLI 実行では MCP 前提のデバッグループ・画面確認・修正ループが回せず、本スキルの設計から外れるため）

## 前提条件

このスキルを使用する前に、以下が整っている必要がある:

- **Maestro CLI** がインストール済み（[Maestro公式ドキュメント](https://docs.maestro.dev/) 参照）— このプラグインは `maestro mcp` を MCP サーバとして自動起動するため、Maestro CLI が `PATH` から実行できる必要がある
- **対象が Flutter アプリ** であり、Dartソースコード（ウィジェットツリー）が読める状態にある（ローカルプロジェクト）
- **iOSシミュレータ / Androidエミュレータ / 実機** が起動済み、または起動可能な状態（フロー実行時のみ必須）

上記が満たされない場合、満たされない項目をユーザーに案内し、整ってから着手する。

## エージェントの動作原則

- **MCPを実行基盤とする**: フローの実行・検証・デバッグは Maestro MCP（`mcp__maestro__*`）で行う。`maestro` CLI を直接 Bash で叩くこと、およびユーザーに CLI 実行手順を代替案として案内することは禁止する。MCP が使えない場合は MCP 再接続のみを依頼する
- **失敗時のみユーザーに聞く**: フロー生成→実行→デバッグの一連は自律で行う。3回修正しても解決しない場合のみユーザーに報告する
- **シナリオ忠実性**: 生成するフローは、ユーザーが記述したシナリオに含まれる操作のみを実装する。「ついでの操作」や「確認のための寄り道」は含めない。テストフローが長いほどフレーキーになりやすい
- **コードを先に読む**: 推測でセレクタを書かない。必ず対象画面のUIコードを読んでから生成する

---

## ワークフロー概要

0. **MCP接続チェック** — 「起動時の最初の操作」に従い、最初のツール呼び出しで `mcp__maestro__list_devices` を実行する
1. **シナリオ理解** — 自然言語からテスト対象・操作・期待結果を特定
2. **UIコード調査** — 対象画面のUIコードを読み、セレクタ候補を収集
3. **セレクタ戦略決定** — 安定したセレクタ優先順位で決定
4. **フロー生成** — プロジェクトの既存パターンに沿ったYAMLを出力
5. **Semantics identifier 追加** — 必要なidentifierがなければUIコードへの追加を提案
6. **アプリ再ビルド・インストール（identifier追加時のみ）** — Step 5 で identifier を追加した場合に実施
7. **MCP実行確認** — Maestro MCPでフローを実行し、失敗時はデバッグループへ

---

## Step 1: シナリオ理解

ユーザーのシナリオから以下を特定する:

- **対象画面**: どのページ / 機能が関係するか
- **ユーザー操作**: タップ、入力、スクロール、スワイプなど
- **期待結果**: 何が表示されるべきか、どの画面に遷移するか
- **前提条件**: ログイン状態、特定データの存在など

シナリオが曖昧な場合（対象画面が特定できない、操作手順が不明など）はユーザーに確認する。

---

## Step 2: UIコード調査

### 対象を特定する

まずプロジェクトのディレクトリ構造を確認し、UIコードの配置を把握する:

```bash
# プロジェクトのUI層を探す（例）
find . -type f -name "*.dart" | grep -E "(screen|page|view)" | head -30
```

### 収集する情報

Flutterウィジェットツリーから以下のセレクタ候補を収集する:

- `Semantics(identifier: 'xxx')` — Maestroの `id:` セレクタで使用（最優先）
- `Text('xxx')` や `label:` — `text:` セレクタで使用
- タップ可能要素: `GestureDetector`, `InkWell`, `ElevatedButton`, `TextButton`
- 入力要素: `TextField`, `TextFormField`
- ナビゲーション: `TabBar`, `BottomNavigationBar`, `Drawer`

### inspect_screen を補助的に使う

実機 / シミュレータが接続されている場合、`mcp__maestro__inspect_screen` でビュー階層を直接確認し、
UIコードの調査結果と照合する。コードに identifier が無い場合の探索にも有効。

---

## Step 3: セレクタ戦略

### 優先順位

1. **Semantics identifier** (`id:`) — 最も安定。言語変更・テキスト変更に影響されない
2. **表示テキスト** (`text:`) — identifierがない場合のフォールバック
3. **正規表現テキスト** (`text: "パターン.*"`) — 以下のケースで必要:
   - テキストが動的に変わる場合（例: 件数・日時・ユーザー名を含むテキスト）
   - accessibility text が表示テキスト以外の補足情報を含み、複数行になっている場合（タブの選択状態、リンクの補足説明など）
4. **座標指定** (`point:`) — 最終手段のみ。上記すべてが使えない場合に限る

### アンチパターン: 改行を含むリテラル文字列の text セレクタ

iOS / Android はナビゲーション要素（TabBar / BottomNavigationBar / Tab 系ウィジェット）に対し、
accessibility text に `<ラベル>\nTab N of M` のような OS 補足情報を自動付加する。
これを `text:` セレクタに**改行込みリテラル全文**として書いてはならない:

```yaml
# 悪い例
- tapOn:
    text: "Profile\nTab 1 of 3"
```

理由: 言語設定・OS バージョン・アクセシビリティ機能で補足文言が変わると即座にフレーキー化する。

代わりに次のいずれかを使う:

1. **Semantics identifier（最優先）**
   ```yaml
   - tapOn:
       id: "nav_profile_tab"
   ```
2. **正規表現でラベル部分のみマッチ**
   ```yaml
   - tapOn:
       text: "Profile.*"
   ```

UI コードに該当の Semantics identifier が無い場合は、Step 5 の手順で追加を提案する。

### identifier の追加ルール

対象ウィジェットに identifier がない場合、UIコードへの追加を提案する（ユーザー確認後に変更）:

```dart
// 追加前
InkWell(
  onTap: () => ...,
  child: Text('送信'),
)

// 追加後
Semantics(
  identifier: 'form_submit_button',
  child: InkWell(
    onTap: () => ...,
    child: Text('送信'),
  ),
)
```

**命名規則**: `[feature]_[element_type]` パターン
- 例: `login_email_field`, `profile_edit_button`, `search_result_list_item`
- 既存の命名パターンに揃える（`inspect_screen` で確認できる）

追加するidentifierは、テストで操作対象となる要素のみ。装飾的な要素やテスト不要な要素には追加しない。

---

## Step 4: フロー生成

### ディレクトリ構成

プロジェクトの既存 `maestro/` 構成を確認し、その命名・配置パターンに従う。
既存構成が不明な場合は以下を標準として採用する:

```
maestro/
└── flows/                   # テストフロー本体
    └── [feature_name].yaml  # 機能・シナリオごとに1ファイル
```

フロー名は対象機能を表すスネークケース（例: `login.yaml`, `search.yaml`, `profile_edit.yaml`）。

### YAMLテンプレート

```yaml
appId: ${APP_ID}
---
- startRecording:
    path: <screenshot_dir>/[scenario_name]
# [シナリオの説明コメント]
- launchApp
- waitForAnimationToEnd
# ... 操作ステップ
- stopRecording
```

### 生成ルール

1. **appId は環境変数 `${APP_ID}` で管理** — ハードコードしない。プロジェクトに合わせて変数名を調整
2. **`startRecording` / `stopRecording` で囲む** — スクリーンショット記録のため。`path` はプロジェクトの既存パターンに従う（既存パターンがなければ `maestro/screenshots/${TIMESTAMP}_${PLATFORM}/[scenario_name]` を標準とする）
3. **`waitForAnimationToEnd` を適切に挿入** — 画面遷移やアニメーション後に挿入。ただし動画再生など永続的なアニメーション画面では使わない（コメントで理由を明記）
4. **コメントで操作意図を記載** — 各ステップの意図をコメントで明確にする
5. **`scroll` よりも `scrollUntilVisible` を優先** — 画面サイズに依存しない安定したテストのため
6. **`extendedWaitUntil` でネットワーク待機** — 通信を伴う画面表示の待機に使用
7. **`assertVisible` で画面遷移を必ず検証** — タップ後に想定の画面に遷移したことを確認する:
   - 画面固有のSemantics identifier（`id: "[feature]_page"`）が最優先
   - 固定タイトルテキスト（動的に変わる場合は使わない）
   - タブの選択状態（`selected: true` プロパティ）

8. **フロー前後の状態対称性を守る** — フロー終了時の画面状態を、フロー開始時と一致させる。複数フローを連続実行できるようにするため:
   - 詳細画面に遷移した場合は戻るボタンで起点画面に戻して終了する
   - ドロワー・モーダルを開いた場合は閉じて終了する

### 典型的な操作パターン

```yaml
# タブ切り替え（id がある場合 — 最優先）
- tapOn:
    id: "nav_[tab_name]"
- waitForAnimationToEnd
- assertVisible:
    id: "nav_[tab_name]"
    selected: true

# タブ切り替え（id が無い場合のフォールバック）
# OS が "<ラベル>\nTab N of M" を付加するため、リテラル全文ではなく正規表現でラベル部分のみマッチ
- tapOn:
    text: "Profile.*"
- waitForAnimationToEnd
- assertVisible:
    text: "Profile.*"
    selected: true

# 戻るボタン
- tapOn:
    id: "back_button"
- waitForAnimationToEnd

# リストアイテムをタップ（先頭要素）
- tapOn:
    id: "[item_identifier]"
    index: 0
- waitForAnimationToEnd

# テキスト入力
- tapOn:
    id: "[field_identifier]"
- eraseText
- inputText: "${SEARCH_KEYWORD}"
- pressKey: enter
- waitForAnimationToEnd

# 要素が見えるまでスクロール
- scrollUntilVisible:
    element:
      id: "[target_id]"
    direction: DOWN
    timeout: 60000
    speed: 60
```

### ダイアログ・オーバーレイ対策

アプリ初回起動やログイン直後はOSダイアログ・アプリ内モーダルが重畳することが多い。
確実に閉じるのが難しい場合は、`stopApp` → `launchApp` で再起動すると多くが再表示されなくなる:

```yaml
# OSダイアログやアプリ内モーダルが出た場合の対策
- extendedWaitUntil:
    visible:
      text: "許可|OK|ホーム画面の要素"   # いずれかが出るまで待機
    timeout: 30000
- stopApp
- launchApp
- extendedWaitUntil:
    visible:
      id: "home_screen_element"
    timeout: 20000
```

フレーキー対策として `tapOn` に `optional: true` を使う:

```yaml
- tapOn:
    text: "許可"
    optional: true   # 要素がなくてもエラーにしない
```

---

## Step 5: Semantics identifier の追加

フロー生成で必要なidentifierがUIコードに存在しない場合:

1. 対象ウィジェットを特定し、追加コードをユーザーに提示
2. ユーザーの確認を得てからコードを変更
3. 変更後はアプリの再ビルド・インストールが必要（Step 6 参照）

identifier追加は、テストのために必要な最小限の箇所のみに留める。

### 識別子追加を特に推奨するケース

以下の要素は OS の accessibility 補足が付きやすく、text セレクタが不安定になるため、
identifier の追加を積極的にユーザーに提案する:

- TabBar / BottomNavigationBar の各タブ
- 画面遷移先の画面ルート（`<screen>_page` 等）
- 動的に変わるテキストを含む要素

---

## Step 6: アプリの再ビルド・インストール（identifier追加時のみ）

Step 5 でUIコードに identifier を追加した場合、**Step 7 に入る前に再ビルド・インストールが必要**。
既存の identifier のみ使用した場合は不要（デバイスにアプリがインストール済みであることを確認するだけでよい）。

`flutter run` はホットリロード待機でブロックするためエージェント実行では使わず、ビルドとインストールを分けて実行する。

### Flutter / iOSシミュレータ向け

```bash
# 1. ビルド（プロジェクトの flavor / scheme に合わせて調整）
flutter build ios --simulator [--flavor <flavor>]

# 2. アプリプロセス停止（uninstallはしない。ログイン状態などのデータを保持するため）
xcrun simctl terminate <device_id> <app_bundle_id>

# 3. インストール（同一bundle IDで上書き）
# .app ファイル名はプロジェクトによって異なる（Flutterデフォルトは Runner.app）
# 不明な場合は `ls build/ios/iphonesimulator/` で確認する
xcrun simctl install <device_id> build/ios/iphonesimulator/<app_name>.app

# 4. 起動
xcrun simctl launch <device_id> <app_bundle_id>
```

### Flutter / Androidエミュレータ・実機向け

```bash
# 1. ビルド（プロジェクトの flavor に合わせて調整）
flutter build apk [--flavor <flavor>] [--debug]

# 2. アプリプロセス停止
adb -s <device_id> shell am force-stop <app_package_id>

# 3. インストール（-r で既存データ保持で上書き）
adb -s <device_id> install -r <path/to/app.apk>

# 4. 旧セッション残留防止のため、再度プロセス停止してから起動
adb -s <device_id> shell am force-stop <app_package_id>
adb -s <device_id> shell monkey -p <app_package_id> -c android.intent.category.LAUNCHER 1
```

> **注意**: `uninstall` はログイン状態などのアプリ内データが消えるため使わない。
> 完全リセットが必要な場合のみ `uninstall` を使い、直後にログインフローで状態を復旧する。

---

## Step 7: Maestro MCPによる実行確認

生成したフローの品質を高めるため、Maestro MCP（`mcp__maestro__*` ツール群）を使って実行確認する。

### 利用可能なMaestro MCPツール

実在するツールは以下の **9種類のみ**。これ以外のツール名は存在しないため呼ばないこと:

| ツール | 用途 |
|---|---|
| `mcp__maestro__list_devices` | 接続中のデバイス/シミュレータ一覧 |
| `mcp__maestro__list_cloud_devices` | Maestro Cloudの利用可能デバイス一覧 |
| `mcp__maestro__run` | フロー実行（構文検証も兼ねる） |
| `mcp__maestro__run_on_cloud` | Maestro Cloud上でフロー実行（非同期） |
| `mcp__maestro__get_cloud_run_status` | Cloud runのステータス取得 |
| `mcp__maestro__inspect_screen` | 現在画面のビュー階層を取得 |
| `mcp__maestro__take_screenshot` | 現在画面のスクリーンショット |
| `mcp__maestro__cheat_sheet` | Maestro YAMLコマンドのチートシート |
| `mcp__maestro__open_maestro_viewer` | Maestro ViewerのローカルURLを案内 |

> アプリの起動・停止・インストール状態の操作はMCPからは行えない。
> フロー内の `launchApp` / `stopApp` / `clearState` などのYAMLコマンドで行う。

### 7-1. デバイス確認

```
mcp__maestro__list_devices で接続中のデバイス/シミュレータを確認
```

デバイスが見つからない場合:
- Bash で `xcrun simctl boot <udid>` (iOS) または `emulator -avd <avd_name>` (Android) を実行して起動
- それも不可ならユーザーに「シミュレータ/エミュレータを起動してください」と依頼

### 7-2. フロー実行（構文検証兼ねる）

`mcp__maestro__run` の3パターンのいずれかを使う:

| パラメータ | 用途 |
|---|---|
| `yaml`（文字列） | インラインYAMLで素早く検証（探索・デバッグ向け） |
| `files`（配列） | 生成した特定のYAMLファイルを実行 |
| `dir` + タグ絞り込み | ディレクトリ単位での実行 |

**実行時の環境変数**（最低限 `APP_ID` を渡す）:

```json
{
  "device_id": "<device_id>",
  "files": ["maestro/flows/<feature_name>.yaml"],
  "env": {
    "APP_ID": "<com.example.app>",
    "TIMESTAMP": "<YYYYMMDD_HHMMSS>",
    "PLATFORM": "ios or android"
  }
}
```

`APP_ID` はプロジェクトの設定（`maestro/*.yaml` の `appId`、`package.json`、`pubspec.yaml` 等）から確認する。

### 7-3. 実行失敗時のデバッグループ

フロー実行が失敗した場合、以下を組み合わせて原因特定・修正する:

1. **画面状態の確認**: `mcp__maestro__take_screenshot` で現在の画面をキャプチャ
2. **ビュー階層の調査**: `mcp__maestro__inspect_screen` で画面要素一覧（id / text / bounds / selected）を取得し、セレクタが実在するか・状態が想定通りかを確認
3. **インラインフローで切り分け**: 失敗ステップ前後を `yaml` パラメータで小さく切り出して実行し、どの操作が原因かを特定
4. **チートシート参照**: `mcp__maestro__cheat_sheet` でコマンドの正確な構文を確認
5. **フローを修正して再実行**: 原因に応じてYAMLを修正し、再度 `mcp__maestro__run` で確認

**3回修正しても解決しない場合**: エラー内容・スクリーンショット・試みた修正をユーザーに報告し、判断を委ねる。

### 7-4. MCPが不安定な場合

接続エラーやタイムアウトが繰り返し発生する場合は、Claude Code の `/mcp` コマンドで
MCPサーバを再接続するか、シミュレータ/エミュレータの再起動をユーザーに依頼する。

### 7-5. MCP利用の判断基準

| 状況 | 対応 |
|---|---|
| MCPとデバイスが揃っている | `run` で実行確認（構文も同時に検証） |
| MCPが利用不可 | ユーザーに `/mcp` での再接続を依頼して待機。CLI の直接実行も、ユーザーへの CLI 実行手順の案内も行わない |
| デバイスが未接続 | 起動を依頼するか Bash で起動。不可なら実行をスキップ |
| 失敗した | デバッグループ（7-3）で原因特定・修正。3回で解決しなければユーザーに報告 |

---

## 認証情報・機密値の外部化

テストアカウントのID / パスワード等はYAMLにハードコードしない。
プロジェクトルートの `.env`（`.gitignore` 済）に `KEY=value` 形式で定義し、
実行時に `-e KEY=$KEY` または MCP の `env` パラメータで渡す:

```bash
# .env の例（ローカル各自で記入、commit禁止）
E2E_TEST_EMAIL=...
E2E_TEST_PASSWORD=...
```

```yaml
# フロー内での参照
- tapOn:
    id: "login_email_field"
- inputText: ${E2E_TEST_EMAIL}
- tapOn:
    id: "login_password_field"
- inputText: ${E2E_TEST_PASSWORD}
```

---

## 出力確認チェックリスト

フロー生成後、以下を確認する:

- [ ] `appId` が環境変数（`${APP_ID}` 等）で管理されている
- [ ] `startRecording` / `stopRecording` で囲まれている
- [ ] 画面遷移後に `waitForAnimationToEnd` がある
- [ ] `assertVisible` で各画面遷移の結果を検証している
- [ ] 使用しているidentifierが実際のコードに存在する（または追加を提案済み）
- [ ] コメントで各操作の意図が説明されている
- [ ] `id:` セレクタを最優先で使用し、`text:` は代替手段として使っている
- [ ] フロー終了時の画面状態がフロー開始時と対称になっている
- [ ] 機密値（認証情報等）がハードコードされていない

---

## 参考リソース

- 詳細なMaestro YAMLコマンドのリファレンス: `references/maestro-yaml-reference.md`
- 利用可能なコマンドの確認: `mcp__maestro__cheat_sheet`
