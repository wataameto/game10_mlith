---
name: verify-game
description: index.html への変更を、実際にブラウザで動かして検証する。ローカルサーバー起動 → Playwright で操作 → コンソールエラー確認、まで一括で行う。index.html を編集した後、コミット/報告前に使う。
---

# verify-game

`index.html`(Mlith本体)への変更を、使い捨てスクリプトを都度書かずに検証するためのスキル。

## 手順

1. **ローカルサーバー起動**(ポート衝突を避けるため一度殺してから起動)
   ```bash
   pkill -f "http.server 8934" 2>/dev/null
   cd /Users/user/work/ai/game10_nlith && python3 -m http.server 8934 &
   ```

2. **Playwright を用意する**
   このプロジェクトには`node_modules`が存在しない(単一HTML・外部ライブラリなし構成のため)。`npx playwright ...`はグローバルキャッシュに落ちるだけでNode.jsのESM importからは解決できず、`import { chromium } from 'playwright'`は失敗する。検証スクリプトを動かす前に毎回ローカルインストールが必要:
   ```bash
   cd /Users/user/work/ai/game10_nlith
   npm install --no-save playwright
   npx --yes playwright install chromium   # 未インストールなら
   ```

3. **Playwright で操作確認**
   `node -e '...'` か一時スクリプトで、以下を目的に応じて組み合わせる。

   - **起動確認**: `http://localhost:8934/index.html` を開き、コンソールエラーが出ていないか(`page.on("console")` / `page.on("pageerror")`)を確認
   - **ゲーム開始**: 難易度ボタン(初級/中級/上級のいずれか)をクリックしないと `started` にならない(SPEC.md §9)。テキストセレクタでクリックする
   - **キーボード操作確認**: `←/→`(移動)、`↑`(回転)、`↓`(ソフトドロップ)、`スペース`(ハードドロップ)、`C`(ブロック変更)、`M`(ミュート)
   - **タップ/ドラッグ操作確認**: `page.mouse` で `pointerdown → pointermove → pointerup` をシミュレートし、以下を区別してテストする(SPEC.md §6.2、過去にここで何度もバグが出ている):
     - 小さい移動(1セル未満)→タップとして扱われる(回転)
     - 1〜1.3セルの「軽いドラッグ」→ 二重発火しないこと(604a99aで修正済みの回帰確認)
     - 上スワイプ1.5セル以上 → ブロック変更
     - 下フリック(1.5セル以上・300ms未満) → ハードドロップ
   - **状態確認**: `page.evaluate(() => window.score)` 等、グローバル変数を直接読んで期待値と比較(index.html はモジュールでないため主要変数は `window` から触れる想定。触れない場合は `evaluate` で描画結果やDOMテキストを見る)

4. **後片付け**
   検証用に入れた`node_modules`/`package.json`/`package-lock.json`は、このプロジェクトが「外部ライブラリなし」構成である体裁を保つため、検証後は必ず削除する。
   ```bash
   pkill -f "http.server 8934" 2>/dev/null
   cd /Users/user/work/ai/game10_nlith && rm -rf node_modules package.json package-lock.json
   ```

## 使う場面

- `index.html` の入力判定・当たり判定・スコア計算などロジックを変更した直後
- 「動くはず」を確認してからユーザーに報告したいとき
- 過去のバグ(タップ/ドラッグの二重発火、`setPointerCapture`の例外、回転握りつぶし — SPEC.md §6.2 参照)の回帰確認をしたいとき

## 注意

- 起動しっぱなしのサーバーが残らないよう、検証後は必ず `pkill` すること
- 一時テストスクリプトを作った場合、使い捨てとして再利用しない(このスキルの手順自体をテンプレートとして使う)
