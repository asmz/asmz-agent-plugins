# Maestro YAML リファレンス

このスキルが生成する Maestro フローで使用可能なコマンドのリファレンス。
詳細な仕様が必要な場合にこのファイルを参照すること。

## Table of Contents

1. [フロー構造](#フロー構造)
2. [操作コマンド](#操作コマンド)
3. [アサーション・待機](#アサーションと待機)
4. [スクロール・スワイプ](#スクロールとスワイプ)
5. [フロー制御](#フロー制御)
6. [入力コマンド](#入力コマンド)
7. [セレクタ](#セレクタ)
8. [変数と環境変数](#変数と環境変数)

---

## フロー構造

```yaml
# 設定セクション（--- の上）
appId: com.example.app    # 必須: アプリのパッケージ名/バンドルID
name: フロー名             # 任意: 表示名
tags:                      # 任意: タグ一覧
  - smokeTest
env:                       # 任意: 環境変数
  KEY: value
onFlowStart:               # 任意: フロー開始前の処理
  - runFlow: setup.yaml
onFlowComplete:            # 任意: フロー完了後の処理
  - runFlow: teardown.yaml
---
# コマンドセクション（--- の下）
- launchApp
- tapOn:
    text: 'ボタン'
```

---

## 操作コマンド

### tapOn
```yaml
- tapOn:
    text: 'ログイン'        # 可視テキストで検索
- tapOn:
    id: 'login_btn'        # testIDで検索
- tapOn:
    label: 'ログインする'   # accessibilityLabelで検索
- tapOn:
    id: 'item'
    index: 0               # N番目の一致要素
- tapOn:
    text: 'ボタン'
    longPress: true         # 長押し
- tapOn:
    point: "50%,80%"       # 座標指定（最終手段）
```

### launchApp / killApp
```yaml
- launchApp                 # appIdのアプリを起動
- killApp                   # アプリを強制終了
```

### back
```yaml
- back                      # システムの戻るボタン（Android/Web）
```

### openLink
```yaml
- openLink: https://example.com
- openLink: ${EXPO_GO_URL}   # Expo Go へのロード（例: exp://127.0.0.1:8081）
```

---

## アサーションと待機

### assertVisible / assertNotVisible
```yaml
- assertVisible:
    text: '期待するテキスト'
- assertVisible:
    id: 'element_id'
- assertNotVisible:
    text: '表示されないはずのテキスト'
```

### waitForAnimationToEnd
```yaml
- waitForAnimationToEnd              # デフォルトタイムアウト
- waitForAnimationToEnd:
    timeout: 15000                   # ミリ秒指定
```

### extendedWaitUntil
```yaml
- extendedWaitUntil:
    visible:
        id: 'element_id'
    timeout: 10000
```

### pause
```yaml
- pause: 1000                        # ミリ秒待機
```

---

## スクロールとスワイプ

### scroll
```yaml
- scroll                             # デフォルト（下方向）
- scroll:
    direction: UP                    # UP, DOWN
    duration: 300
```

### scrollUntilVisible
```yaml
- scrollUntilVisible:
    element: "目標テキスト"           # テキストで検索
    direction: DOWN
    timeout: 60000                   # ミリ秒
    speed: 60                        # 0-100
- scrollUntilVisible:
    element:
      id: "target_id"               # testIDで検索
    direction: DOWN
```

### swipe
```yaml
- swipe:
    direction: LEFT                  # UP, DOWN, LEFT, RIGHT
    duration: 300
```

---

## フロー制御

### runFlow
```yaml
- runFlow: other_flow.yaml
- runFlow:
    file: common/setup.yaml
- runFlow:
    file: auth.yaml
    env:
      EMAIL: test@example.com
```

### repeat
```yaml
- repeat:
    times: 3
    commands:
      - scroll
      - waitForAnimationToEnd
```

### startRecording / stopRecording
```yaml
- startRecording:
    path: maestro/screenshots/${TIMESTAMP}_${PLATFORM}/test_name
# ... テスト操作 ...
- stopRecording
```

---

## 入力コマンド

### inputText / eraseText
```yaml
- inputText: "入力テキスト"
- eraseText                          # フォーカス中のフィールドのテキストを削除
- eraseText:
    charactersToErase: 5             # 指定文字数削除
```

### pressKey
```yaml
- pressKey: enter
- pressKey: backspace
```

### copyTextFrom / pasteText
```yaml
- copyTextFrom:
    id: 'text_element'
- pasteText
```

### ランダム入力
```yaml
- inputRandomEmail
- inputRandomPersonName
- inputRandomNumber
- inputRandomText
```

---

## セレクタ

### 優先順位（このプロジェクトでの方針）

1. `id:` — React Native の `testID` ベース。最も安定
2. `label:` — React Native の `accessibilityLabel` ベース（iOS: accessibilityLabel / Android: contentDescription）
3. `text:` — 可視テキストベース。正規表現対応
4. `point:` — 座標ベース。最終手段

### 正規表現
```yaml
- tapOn:
    text: "^ログイン.*"              # 先頭一致
- tapOn:
    text: ".*確認.*"                 # 部分一致
- assertVisible:
    text: "注文番号: \\d+"           # 数字パターン
```

### index（複数一致時）
```yaml
- tapOn:
    id: 'list_item'
    index: 2                         # 3番目（0始まり）
```

---

## 変数と環境変数

### フロー内定義
```yaml
appId: ${APP_ID}
env:
  SEARCH_KEYWORD: "IT"
  MAX_SCROLL: 5
---
- inputText: ${SEARCH_KEYWORD}
```

### CLI引数
```bash
# アプリバイナリ
maestro test -e APP_ID=com.example.app -e SEARCH_KEYWORD=React flow.yaml

# Expo Go (iOS Simulator)
maestro test \
  -e APP_ID=host.exp.Exponent \
  -e EXPO_GO_URL=exp://127.0.0.1:8081 \
  -e EXPO_GO_PROJECT_NAME=example-app \
  -e SEARCH_KEYWORD=React \
  flow.yaml
```

### 変数参照
```yaml
- inputText: ${VARIABLE_NAME}        # どのコマンドパラメータでも使用可能
```
