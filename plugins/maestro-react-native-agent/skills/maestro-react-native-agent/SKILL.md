---
name: maestro-react-native-agent
description: >
  Agent skill that autonomously generates, runs, debugs, and fixes Maestro test flows (YAML) for React Native mobile apps.
  Given a scenario in natural language, it reads the React Native component code, generates the matching Maestro YAML,
  runs it via the Maestro MCP, and iterates on failures until the flow passes (or escalates if it cannot resolve).
  Supports both Expo Go execution and native app binaries (.app / .apk).
when_to_use: >
  Use when the user wants to write, run, debug, or fix Maestro E2E test flows for a React Native mobile app —
  including creating new flows, modifying existing ones, adding steps, handling test failures,
  OS or in-app dialogs causing flakiness, or building out a regression suite.

  日本語の依頼例: 「Maestroのフローを作って」「画面遷移のテストを自動化したい」「UIテストが欲しい」
  「このフローを直して」「ステップを追加して」「テストが失敗するから対処して」「OSダイアログで止まる」
  「Maestroフローがフレーキー」「回帰テストを整備したい」など。
  新規フロー作成・既存フロー修正・ステップ追加・失敗対処・OSダイアログやアプリ内ダイアログ起因のフレーキー対応・回帰テスト整備が対象。
---

# Maestro React Native Agent

Maestro MCP を活用して、React Native モバイルアプリのテストフロー（YAML）を
自律的に生成・実行・修正するエージェントスキル。Expo Go / アプリバイナリのどちらでも動作する。

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
- **対象が React Native アプリ** であり、JS / TS / JSX / TSX ソースコード（コンポーネントツリー）が読める状態にある（ローカルプロジェクト）。Expo managed / Expo dev build / bare React Native のいずれも対応
- **iOSシミュレータ / Androidエミュレータ / 実機** が起動済み、または起動可能な状態（フロー実行時のみ必須）
- 実行ターゲットとして **Expo Go** または **アプリバイナリ** のいずれかが利用可能であること（Step 1 でプロジェクト構成から自動判別）

上記が満たされない場合、満たされない項目をユーザーに案内し、整ってから着手する。

## エージェントの動作原則

- **MCPを実行基盤とする**: フローの実行・検証・デバッグは Maestro MCP（`mcp__maestro__*`）で行う。`maestro` CLI を直接 Bash で叩くこと、およびユーザーに CLI 実行手順を代替案として案内することは禁止する。MCP が使えない場合は MCP 再接続のみを依頼する
- **失敗時のみユーザーに聞く**: フロー生成→実行→デバッグの一連は自律で行う。3回修正しても解決しない場合のみユーザーに報告する
- **シナリオ忠実性**: 生成するフローは、ユーザーが記述したシナリオに含まれる操作のみを実装する。「ついでの操作」や「確認のための寄り道」は含めない。テストフローが長いほどフレーキーになりやすい
- **コードを先に読む**: 推測でセレクタを書かない。必ず対象画面のUIコードを読んでから生成する

---

## ワークフロー概要

0. **MCP接続チェック** — 「起動時の最初の操作」に従い、最初のツール呼び出しで `mcp__maestro__list_devices` を実行する
1. **シナリオ理解 & プロジェクト判別** — 自然言語からテスト対象・操作・期待結果を特定し、プロジェクト構成から実行モード（Expo Go / アプリバイナリ）を確定
2. **UIコード調査** — 対象画面のUIコードを読み、セレクタ候補を収集
3. **セレクタ戦略決定** — 安定したセレクタ優先順位で決定
4. **フロー生成** — プロジェクトの既存パターンに沿ったYAMLを出力（実行モードに応じた先頭テンプレを採用）
5. **testID 追加** — 必要な testID がなければUIコードへの追加を提案
6. **アプリの起動 / 再ビルド・インストール** — 実行モードに応じて Expo Go 経由または `.app` / `.apk` の再ビルドを実施
7. **MCP実行確認** — Maestro MCPでフローを実行し、失敗時はデバッグループへ

---

## Step 1: シナリオ理解 & プロジェクト判別

### 1-1. シナリオ理解

ユーザーのシナリオから以下を特定する:

- **対象画面**: どのページ / 機能が関係するか
- **ユーザー操作**: タップ、入力、スクロール、スワイプなど
- **期待結果**: 何が表示されるべきか、どの画面に遷移するか
- **前提条件**: ログイン状態、特定データの存在など

シナリオが曖昧な場合（対象画面が特定できない、操作手順が不明など）はユーザーに確認する。

### 1-2. プロジェクト形態の判別と実行モードの確定

初回フロー生成前に、エージェントはプロジェクト構成を読み取って実行モードを決定する（一度判定したら、以降のフローでも同じモードを使い回す）。

**判別ルール**（この順で判定）:

1. `app.json` / `app.config.js` / `app.config.ts` / `app.config.json` のいずれかが存在 → **Expo プロジェクト**
2. `package.json` の `dependencies` / `devDependencies` に `expo` が含まれる → **Expo プロジェクト**
3. 上記いずれにも該当せず `ios/` `android/` ディレクトリがある → **bare React Native プロジェクト**

**既定の実行モード**:

| プロジェクト形態 | 既定モード |
|---|---|
| Expo | **Expo Go モード**（Step 6.A） |
| bare React Native | **アプリバイナリモード**（Step 6.B） |

ユーザーが明示的に「dev build / バイナリで動かしたい」「Expo Go ではなく実機ビルドで」などと指示した場合は、アプリバイナリモードへ切り替える。

### 1-3. `APP_ID` の確定

判定した実行モードに応じて `APP_ID` を確定し、以降のフロー生成で使い回す:

- **Expo Go モード**:
  - iOS Simulator: `host.exp.Exponent`
  - Android Emulator / 実機: `host.exp.exponent`
- **アプリバイナリモード**: 以下のいずれかから取得
  - `app.json` の `ios.bundleIdentifier` / `android.package`
  - `app.config.{js,ts}` から evaluate される値
  - `ios/<name>/Info.plist` の `CFBundleIdentifier`
  - `android/app/build.gradle` の `applicationId`

### 1-4. Expo Go モード時の追加項目: プロジェクト名の確定

Expo Go モードでは、`openLink` だけではアプリが直接開かず Expo Go のホーム画面に留まることがある（"Recently opened" にプロジェクトが並んだ状態で停止）。この救済のために、フローから "Recently opened" のセルをタップできるよう **プロジェクト表示名** を取得しておく:

- `app.json` の `expo.name` を最優先（Expo Go のホームに表示される文字列）
- フォールバック: `app.json` の `expo.slug` / `package.json` の `name`

確定した名前は環境変数 `EXPO_GO_PROJECT_NAME` として記憶し、フロー先頭テンプレに埋め込む（Step 4 参照）。

---

## Step 2: UIコード調査

### 対象を特定する

まずプロジェクトのディレクトリ構造を確認し、UIコードの配置を把握する:

```bash
# プロジェクトのUI層を探す（例）
find . -type f \( -name "*.tsx" -o -name "*.ts" -o -name "*.jsx" -o -name "*.js" \) \
  | grep -Ev "(node_modules|\.expo|ios/Pods|android/build)" \
  | grep -E "(screen|page|view|navigator)" | head -30
```

### 収集する情報

React Native コンポーネントツリーから以下のセレクタ候補を収集する:

- `testID="xxx"` props — Maestroの `id:` セレクタで使用（最優先）
- `accessibilityLabel="xxx"` — testID が無いときのフォールバック（Maestro の `label:` セレクタでマッチ可能）
- 表示テキスト（`<Text>...</Text>` の中身）— `text:` セレクタで使用
- タップ可能要素: `Pressable`, `TouchableOpacity`, `TouchableHighlight`, `TouchableWithoutFeedback`, `Button`
- 入力要素: `TextInput`
- ナビゲーション（React Navigation / expo-router）: **標準の `testID` ではなく独自プロパティ名を使う箇所がある**ので注意:
  - `@react-navigation/bottom-tabs` / `@react-navigation/material-top-tabs` / `expo-router` の `<Tabs>`:
    各タブの testID は `<Tab.Screen options={{ tabBarButtonTestID: '...' }} />`。アクセシビリティラベルは `tabBarAccessibilityLabel`
  - `@react-navigation/stack` (JS Stack): ヘッダー戻るボタンは `options.headerBackTestID` / `options.headerBackAccessibilityLabel`
  - `@react-navigation/native-stack`: testID 用オプションは**無い**ため、`headerLeft` / `headerRight` をカスタムコンポーネントにして自前で `testID` を付ける
  - `@react-navigation/drawer`: testID 用オプションは**無い**ため、`drawerContent` をカスタム実装し、各項目に `testID` を付ける（または `drawerLabel` の文字列で `text:` セレクタを使う）

### inspect_screen を補助的に使う

実機 / シミュレータが接続されている場合、`mcp__maestro__inspect_screen` でビュー階層を直接確認し、
UIコードの調査結果と照合する。コードに testID が無い場合の探索にも有効。

---

## Step 3: セレクタ戦略

### 優先順位

1. **`testID`** (Maestroの `id:`) — 最も安定。言語変更・テキスト変更に影響されない
   - React Native の `testID` prop は iOS では `accessibilityIdentifier`、Android では `resource-id` にマップされ、Maestro の `id:` セレクタでマッチする
   - 一部のスタイリング / ラッパーライブラリで testID が hierarchy に出ないことがあるため、不安定な場合は `accessibilityLabel` を併設して `label:` セレクタにフォールバック
2. **`accessibilityLabel`** (Maestroの `label:`) — testID が無いがアクセシビリティラベルがある場合のフォールバック。iOS は accessibilityLabel、Android は contentDescription にマップされる
3. **表示テキスト** (Maestroの `text:`) — 上記がない場合のフォールバック
4. **正規表現テキスト** (`text: "パターン.*"`) — 以下のケースで必要:
   - テキストが動的に変わる場合（例: 件数・日時・ユーザー名を含むテキスト）
   - accessibility text が表示テキスト以外の補足情報を含み、複数行になっている場合（タブの選択状態、リンクの補足説明など）
5. **座標指定** (`point:`) — 最終手段のみ。上記すべてが使えない場合に限る

### アンチパターン: 改行を含むリテラル文字列の text セレクタ

iOS / Android はタブ系ナビゲーション要素（React Navigation の Bottom Tab / Material Top Tab など）に対し、
accessibility text に `<ラベル>\nTab N of M` のような OS 補足情報を自動付加する。
これを `text:` セレクタに**改行込みリテラル全文**として書いてはならない:

```yaml
# 悪い例
- tapOn:
    text: "Profile\nTab 1 of 3"
```

理由: 言語設定・OS バージョン・アクセシビリティ機能で補足文言が変わると即座にフレーキー化する。

代わりに次のいずれかを使う:

1. **`testID`（最優先）**
   ```yaml
   - tapOn:
       id: "nav_profile_tab"
   ```
2. **正規表現でラベル部分のみマッチ**
   ```yaml
   - tapOn:
       text: "Profile.*"
   ```

UI コードに該当の `testID` が無い場合は、Step 5 の手順で追加を提案する。

### testID の追加ルール

対象コンポーネントに `testID` がない場合、UIコードへの追加を提案する（ユーザー確認後に変更）:

```tsx
// 追加前
<Pressable onPress={...}>
  <Text>送信</Text>
</Pressable>

// 追加後
<Pressable testID="form_submit_button" onPress={...}>
  <Text>送信</Text>
</Pressable>
```

React Navigation の Bottom Tab に testID を付ける例（プロパティ名は `tabBarButtonTestID`）:

```tsx
<Tab.Screen
  name="Profile"
  component={ProfileScreen}
  options={{
    tabBarButtonTestID: 'nav_profile_tab',
    tabBarAccessibilityLabel: 'プロフィール',
  }}
/>
```

JS Stack（`@react-navigation/stack`）の戻るボタンに testID を付ける例:

```tsx
<Stack.Screen
  name="Detail"
  component={DetailScreen}
  options={{
    headerBackTestID: 'detail_back_button',
    headerBackAccessibilityLabel: '戻る',
  }}
/>
```

native-stack / Drawer には testID 専用オプションが**無い**ため、`headerLeft` / `drawerContent` をカスタム実装してその中で `testID` を付ける:

```tsx
<NativeStack.Screen
  name="Detail"
  component={DetailScreen}
  options={{
    headerLeft: () => (
      <Pressable
        testID="detail_back_button"
        accessibilityLabel="戻る"
        onPress={...}
      >
        <Text>←</Text>
      </Pressable>
    ),
  }}
/>
```

**命名規則**: `[feature]_[element_type]` パターン
- 例: `login_email_field`, `profile_edit_button`, `search_result_list_item`
- 既存の命名パターンに揃える（`inspect_screen` で確認できる）

追加する testID は、テストで操作対象となる要素のみ。装飾的な要素やテスト不要な要素には追加しない。

### 既知の落とし穴

セレクタを設計するときに踏みやすい罠:

- **`Modal` の testID は内側ラッパー View に付く** — Modal そのものではなく中身の View に id が付与されるので、`inspect_screen` で確認してから参照する
- **`FlatList` / `SectionList` のアイテムには専用 testID プロパティが無い** — `renderItem` 内で `testID={\`row_${item.id}\`}` のように自前で振る
- **UI ライブラリ（`react-native-paper` の `Button` 等）はライブラリ実装によって内部で `testID` の既定値を持つことがある** — ドキュメント・ソースを確認し、衝突を避けるため対象要素には明示的に testID を渡す
- **一部のスタイリングライブラリでは特定の prop 併用で testID が伝搬しない事例が報告されている** — `inspect_screen` で testID が見えない場合は `accessibilityLabel` を併設して `label:` セレクタにフォールバック
- **Android では親 View に `accessible={true}` が付くと子の testID が hierarchy 上隠れる** — `inspect_screen` で見えない場合は、親の `accessible` 設定を疑う

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

Step 1 で確定した実行モードに応じて、フロー先頭テンプレを使い分ける。

> 以下の各テンプレ内 `# ...` で始まる**説明用コメント**（`# host.exp.Exponent (iOS) / ...` など、変数の取りうる値や用途を解説するもの）は読み手向けの注釈なので、**生成する実物の YAML には残さない**。シナリオの意図を示すコメント（操作意図の説明）は残す。

**Expo Go モード**:

```yaml
appId: ${APP_ID}        # host.exp.Exponent (iOS) / host.exp.exponent (Android)
---
- startRecording:
    path: <screenshot_dir>/[scenario_name]
# [シナリオの説明コメント]
- launchApp
- openLink: ${EXPO_GO_URL}   # 例: exp://127.0.0.1:8081
- waitForAnimationToEnd
# Expo Go のホーム画面に留まった場合の救済:
# "Recently opened" にプロジェクトが並んでいるだけのケースがあるため、その時はプロジェクト名のセルをタップする
- runFlow:
    when:
      visible:
        text: "Recently opened"
    commands:
      - tapOn:
          text: ${EXPO_GO_PROJECT_NAME}
      - waitForAnimationToEnd
# ... 操作ステップ
- stopRecording
```

**アプリバイナリモード**:

```yaml
appId: ${APP_ID}        # 例: com.example.app
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

1. **appId は環境変数 `${APP_ID}` で管理** — ハードコードしない。Expo Go の場合も `${APP_ID}` 経由で渡す
2. **Expo Go モードでは `openLink: ${EXPO_GO_URL}` + "Recently opened" 救済ブロックをテンプレに含める** — `openLink` だけで目的アプリが開かず Expo Go ホームに留まるケースがあるため、`runFlow.when.visible: "Recently opened"` でホーム画面検出時に `${EXPO_GO_PROJECT_NAME}` のセルをタップする救済を必ず入れる
3. **`startRecording` / `stopRecording` で囲む** — スクリーンショット記録のため。`path` はプロジェクトの既存パターンに従う（既存パターンがなければ `maestro/screenshots/${TIMESTAMP}_${PLATFORM}/[scenario_name]` を標準とする）
4. **`waitForAnimationToEnd` を適切に挿入** — 画面遷移やアニメーション後に挿入。ただし動画再生など永続的なアニメーション画面では使わない（コメントで理由を明記）
5. **コメントで操作意図を記載** — 各ステップの意図をコメントで明確にする
6. **`scroll` よりも `scrollUntilVisible` を優先** — 画面サイズに依存しない安定したテストのため
7. **`extendedWaitUntil` でネットワーク待機** — 通信を伴う画面表示の待機に使用
8. **`assertVisible` で画面遷移を必ず検証** — タップ後に想定の画面に遷移したことを確認する:
   - 画面固有の `testID`（`id: "[feature]_screen"` 等）が最優先
   - 固定タイトルテキスト（動的に変わる場合は使わない）
   - タブの選択状態（`selected: true` プロパティ）

9. **フロー前後の状態対称性を守る** — フロー終了時の画面状態を、フロー開始時と一致させる。複数フローを連続実行できるようにするため:
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
Expo Go モードでは加えて、開発メニュー（Cmd+D / Cmd+M）が出ることもある。
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

## Step 5: testID の追加

> 追加コード例・命名規則・各 Navigator のプロパティ名は Step 3 を参照。本セクションは実行手順と推奨ケースのみ。

### 実行手順

1. 対象コンポーネントを特定し、追加コードをユーザーに提示
2. ユーザーの確認を得てからコードを変更
3. 変更後は通常 Metro の Fast Refresh で反映される（再ビルド不要）。ネイティブモジュール追加や Expo 設定変更を伴う場合のみ Step 6.B のネイティブ再ビルドを実施

### 識別子追加を特に推奨するケース

以下の要素は OS の accessibility 補足が付きやすく text セレクタが不安定になりやすい、または専用 testID オプションが無いため、`testID` の追加を積極的にユーザーに提案する。**プロパティ名はナビゲーターごとに異なる**ので Step 2 の対応表に従う:

- React Navigation の Bottom Tab / Material Top Tab の各タブ（`options.tabBarButtonTestID`）
- JS Stack のヘッダー戻るボタン（`options.headerBackTestID`）
- native-stack / Drawer のヘッダー・項目（testID 用オプションが無いため `headerLeft` / `drawerContent` をカスタム実装して自前 `testID`）
- 画面遷移先の画面ルート（`<View testID="[screen]_screen">` で画面ラッパに付与）
- リスト項目の親 View（先頭一致 + `index` で操作するため。`FlatList` / `SectionList` には専用 testID プロパティが無いので `renderItem` 内で自前付与）
- `Modal` の中身（Modal そのものではなく内側 View に付く点に注意）
- 動的に変わるテキストを含む要素

---

## Step 6: アプリの起動 / 再ビルド・インストール

Step 1 で確定した実行モードに応じて分岐する。

### 6.A Expo Go モード（Expo プロジェクトの既定）

Expo Go アプリ上で JS バンドルを読み込んで実行する。testID の追加は Metro の Fast Refresh で反映されるため、原則として再ビルドは不要。

#### 6.A-1. Metro 開発サーバの起動確認

Bash で port 8081 の listen 状態を確認する:

```bash
lsof -nP -iTCP:8081 -sTCP:LISTEN >/dev/null 2>&1 && echo running || echo stopped
```

- 起動済みなら何もせず Step 7 へ進む
- 停止していたら、ユーザーに「Expo 開発サーバ（`npx expo start`）をバックグラウンドで起動してよいか」を確認し、承諾後に Bash で `npx expo start --port 8081` を `run_in_background=true` で起動。port 8081 が listen するまで待機（数秒間隔で再チェック）
- 拒否された場合はユーザー側での起動を依頼し、サーバが立ち上がるまで Step 6 を保留する

#### 6.A-2. 起動後の落とし穴

- **testID 反映**: 通常 Fast Refresh で反映される。`inspect_screen` で新規 testID が見えなければ Expo Go 内でリロード（端末を振る → "Reload" / シミュレータで Cmd+R）
- **開発メニュー（Cmd+D / Cmd+M）対策**: フローを妨げる場合、`launchApp` 直後で `back` を入れるか、`tapOn: text: "Dismiss" optional: true` で閉じる

### 6.B アプリバイナリモード（bare RN の既定、または Expo + dev build を選んだ場合）

`.app` / `.apk` のネイティブビルドをデバイスにインストールして実行する。testID の追加は通常 Fast Refresh で反映されるため、ネイティブモジュール追加や Expo 設定（`app.json` / `app.config.*` の native 設定）変更を伴わない限り再ビルドは不要。

Debug ビルドを実行する場合は Metro 開発サーバ（`npx expo start` または `npx react-native start`）が必要なため、6.A-1 と同じ手順でサーバ起動状態を確認する。

#### 6.B-1. iOS シミュレータ向け

```bash
# 1. ビルド
#    Expo dev/preview build:
npx expo run:ios --configuration Release [--scheme <name>]
#    bare React Native:
# cd ios && xcodebuild -workspace <name>.xcworkspace -scheme <name> \
#     -configuration Debug -sdk iphonesimulator -derivedDataPath build

# 2. アプリプロセス停止（uninstallはしない。ログイン状態などのデータを保持するため）
xcrun simctl terminate <device_id> <app_bundle_id>

# 3. インストール（同一bundle IDで上書き）
# .app の出力先はビルド方式に依存するため `ls` で確認する
# 例: ios/build/Build/Products/Debug-iphonesimulator/<name>.app
xcrun simctl install <device_id> <path/to/Build.app>

# 4. 起動
xcrun simctl launch <device_id> <app_bundle_id>
```

#### 6.B-2. Android エミュレータ・実機向け

```bash
# 1. ビルド
#    Expo dev build:
npx expo run:android --variant release
#    bare React Native:
# cd android && ./gradlew assembleDebug   # (or assembleRelease)

# 2. アプリプロセス停止
adb -s <device_id> shell am force-stop <app_package_id>

# 3. インストール（-r で既存データ保持で上書き）
adb -s <device_id> install -r <path/to/app.apk>

# 4. 旧セッション残留防止のため、再度プロセス停止してから起動
adb -s <device_id> shell am force-stop <app_package_id>
adb -s <device_id> shell monkey -p <app_package_id> -c android.intent.category.LAUNCHER 1
```

> **注意**: `uninstall` / Simulator のアプリ削除はログイン状態などのアプリ内データが消えるため使わない。
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

**実行時の環境変数**（最低限 `APP_ID` を渡す。Expo Go モードでは `EXPO_GO_URL` と `EXPO_GO_PROJECT_NAME` も渡す）:

```json
// アプリバイナリモード
{
  "device_id": "<device_id>",
  "files": ["maestro/flows/<feature_name>.yaml"],
  "env": {
    "APP_ID": "com.example.app",
    "TIMESTAMP": "<YYYYMMDD_HHMMSS>",
    "PLATFORM": "ios or android"
  }
}

// Expo Go モード
{
  "device_id": "<device_id>",
  "files": ["maestro/flows/<feature_name>.yaml"],
  "env": {
    "APP_ID": "host.exp.Exponent",
    "EXPO_GO_URL": "exp://127.0.0.1:8081",
    "EXPO_GO_PROJECT_NAME": "example-app",
    "TIMESTAMP": "<YYYYMMDD_HHMMSS>",
    "PLATFORM": "ios"
  }
}
```

`APP_ID` はプロジェクトの設定から確認する:
- `app.json` の `expo.ios.bundleIdentifier` / `expo.android.package`
- `app.config.{js,ts}` の同等プロパティ
- `ios/<name>/Info.plist` の `CFBundleIdentifier`
- `android/app/build.gradle` の `applicationId`
- Expo Go の場合は固定値 `host.exp.Exponent` (iOS) / `host.exp.exponent` (Android)

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
- [ ] Expo Go モードの場合、フロー先頭に `openLink: ${EXPO_GO_URL}` と "Recently opened" 救済ブロック（`${EXPO_GO_PROJECT_NAME}` を tap）が含まれている
- [ ] `startRecording` / `stopRecording` で囲まれている
- [ ] 画面遷移後に `waitForAnimationToEnd` がある
- [ ] `assertVisible` で各画面遷移の結果を検証している
- [ ] 使用している `testID` が実際のコードに存在する（または追加を提案済み）
- [ ] コメントで各操作の意図が説明されている
- [ ] セレクタが優先順位（`id:` → `label:` → `text:`）に沿って選ばれている
- [ ] フロー終了時の画面状態がフロー開始時と対称になっている
- [ ] 機密値（認証情報等）がハードコードされていない

---

## 参考リソース

- 詳細なMaestro YAMLコマンドのリファレンス: `references/maestro-yaml-reference.md`
- 利用可能なコマンドの確認: `mcp__maestro__cheat_sheet`
