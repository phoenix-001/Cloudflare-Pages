# v31 単一HTML コードレビュー

## (A) 重大バグ（動かない/環境依存で落ちる）

1. **`btnCopyUrl` / `btnCopyA/B/C` のフォールバックコピーが Safari/WebView で失敗しても成功表示になる**
   - 原因: `navigator.clipboard.writeText` 失敗時に `document.execCommand("copy")` の戻り値を確認していません。コピー不可環境でも `setStatus("コピーしました")` が実行されます。
   - 影響: 「コピー」ボタンが動いたように見えて実際には未コピー。iOS WebView/iframe 経由で特に再現しやすい。
   - 修正案: `execCommand` の boolean 戻り値を判定し、失敗時は「手動コピーしてください」を表示。さらに `readonly textarea + setSelectionRange` を使って iOS の選択安定性を上げる。

2. **`openReviewPage()` の `location.assign` が in-app WebView でブロックされた場合にユーザー導線が弱い**
   - 原因: `location.assign` は例外を投げないブロックケースがあり、catch に入らず無反応に見えることがあります。
   - 影響: 「口コミページを開く」が「押しても遷移しない」に見える。
   - 修正案: `window.open(REVIEW_URL, '_blank', 'noopener')` を先に試し、戻り値 `null` なら `location.href` へフォールバック。失敗時は `reviewLink` を目立たせて明示誘導。

3. **住所NG判定の正規表現が過検知しすぎて常時警告に近い**
   - 原因: `/都|道|府|県|市|区|町|村/` は一般語に反応（例: 「雰囲気」「説明」等の漢字文脈でも誤検知し得る）。
   - 影響: 入力チェック警告が多発し、UX悪化。実運用で「常に警告が出る」誤解を招く。
   - 修正案: 住所形式のみに限定（郵便番号＋地名、または `\d+丁目\d+-\d+` など）し、単漢字マッチは削除。

4. **`contenteditable` への貼り付けでHTML断片が残る可能性**
   - 原因: 出力は `textContent` で安全に描画しているが、ユーザーが編集時にリッチテキストを貼ると DOM 内にタグが入り得る。
   - 影響: 直接XSS実行はしにくいが、想定外改行・装飾・不可視文字混入でコピー品質が不安定。
   - 修正案: `paste` イベントで `text/plain` のみ挿入するハンドラを `aBox/bBox/cBox` に追加。

## (B) 改善点（保守性/可読性/テスト）

1. **DOM参照の存在検査を共通化**
   - 現状は `$("id").addEventListener(...)` を直接呼んでおり、将来ID変更時に即時例外になります。
   - `bindClick(id, handler)` を作って null ガード＋警告ログに寄せると安全です。

2. **文章生成ロジックの敬体統一を強化**
   - `ensureMasu` は末尾変換中心で、文中の常体（例: 「〜だが」「〜と感じる」）が混在する可能性があります。
   - 生成前に「断定語尾→敬体」ルールをもう1段追加し、短文連結前に正規化すると品質が安定します。

3. **「箇条書き感の抑制」改善**
   - 接続詞を毎文付けるため、短文が連続すると不自然です。
   - 2文目のみ接続詞、3文目以降は 50% で省略などの確率制御が自然です。

4. **`clearAll()` の完全リセット検証を自動化**
   - 手動では漏れが見落とされます。
   - Playwright 等で「入力→生成→クリア→全要素初期値一致」を1本持つと回 regressions を防げます。

## (C) 具体的パッチ案（関数名＋差し替えコード）

### 1) `safeCopyText` を新設し、URLコピー・原稿コピー双方で利用

```js
async function safeCopyText(text){
  const t = String(text || "");
  if (!t) return { ok:false, reason:"empty" };

  if (navigator.clipboard && navigator.clipboard.writeText) {
    try {
      await navigator.clipboard.writeText(t);
      return { ok:true, method:"clipboard" };
    } catch (_) {}
  }

  try {
    const ta = document.createElement("textarea");
    ta.value = t;
    ta.setAttribute("readonly", "");
    ta.style.position = "fixed";
    ta.style.top = "0";
    ta.style.opacity = "0";
    document.body.appendChild(ta);
    ta.focus();
    ta.select();
    ta.setSelectionRange(0, ta.value.length);
    const ok = document.execCommand("copy");
    ta.remove();
    return ok ? { ok:true, method:"execCommand" } : { ok:false, reason:"blocked" };
  } catch (e) {
    return { ok:false, reason:(e && e.message) || "copy_failed" };
  }
}
```

**置換ポイント**
- `btnCopyUrl` の click handler
- `copyFromBox(which)` 内コピー処理

### 2) `openReviewPage` をポップアップ/ブロック耐性付きへ差し替え

```js
function openReviewPage(){
  try {
    const w = window.open(REVIEW_URL, "_blank", "noopener");
    if (w) return;
  } catch (_) {}

  try {
    window.location.href = REVIEW_URL;
    return;
  } catch (_) {}

  const link = $("reviewLink");
  if (link) {
    link.focus();
    setStatus("自動で開けなかったため、下のリンクから開いてください。", "warn");
  }
}
```

### 3) `NG_PATTERNS` の住所判定を過検知しにくく変更

```js
{ re:/(〒?\d{3}-?\d{4}.+)|(\d{1,4}丁目\d{1,4}-\d{1,4})/, msg:"住所らしき表現が含まれています。必要なら伏せるのがおすすめです。" },
```

### 4) `contenteditable` 貼り付けをプレーンテキスト限定

```js
function bindPlainPaste(id){
  const el = $(id);
  if (!el) return;
  el.addEventListener("paste", (e)=>{
    e.preventDefault();
    const text = (e.clipboardData || window.clipboardData).getData("text") || "";
    document.execCommand("insertText", false, text);
  });
}

bindPlainPaste("aBox");
bindPlainPaste("bBox");
bindPlainPaste("cBox");
```

## (D) iPhone/PC の再現テスト手順（3〜6ステップ）

### iPhone（Safari/Google Drive内プレビュー/WebView想定）
1. 画面読込後、`JS: 稼働中` 表示を確認。
2. 「サンプル」→3案が生成され、各タグが `生成済み` になることを確認。
3. 各「コピー」押下後、メモアプリへ貼り付けて一致確認（URLコピーも同様）。
4. 各「口コミページを開く」押下で遷移可否を確認（失敗時はリンク導線表示を確認）。
5. `aBox` にWebページ由来の装飾付きテキストを貼付し、再コピー時にプレーンテキスト化されることを確認。
6. 「クリア」押下後、全入力・匿名トグル・3出力欄・タグ・ステータスが初期値へ戻ることを確認。

### PC（Chrome）
1. 必須4項目未入力で「3案生成」し、警告表示のみで落ちないことを確認。
2. 必須入力後に「3案生成」、3案の文体（短い/標準/丁寧）差分と敬体終止を確認。
3. 匿名ONで電話/メール/郵便番号を含む入力から生成し、`[非公開]` へ置換されることを確認。
4. 「クリア」でフォームと出力が完全リセットされることを確認。
