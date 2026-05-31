# 2ケタ÷1ケタ 整数わり算ラボ Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 既存の単一ファイル `index.html`（わり算ラボ）に、筆算がはじめての子向けの「2ケタ÷1ケタ」整数わり算モードを追加する。

**Architecture:** 既存 IIFE に `mode=3` を足す。ホームに4つ目のボタン＋小メニュー（れんしゅう=A型/タイマー無し、ちょうせん=B型/タイマー有り）。1問は ①たば&ばらで分ける → ②筆算（A型フル4ステップ or B型数字えらび）→ ③たしかめ算。タイマー/HUD/結果/たしかめ算の共通インフラは流用し、①と②Aは新規関数で描画する。純粋ロジック（問題生成・筆算モデル）は `#test` ハッシュで走る `console.assert` 自己テストで検証する。

**Tech Stack:** プレーン HTML/CSS/JS（ビルド・依存・テストフレームワーク無し）。検証はブラウザ手動 ＋ `index.html#test` の console.assert。

---

## 前提・実行環境メモ

- このプロジェクトは **git リポジトリではない**。版管理したい場合は最初に `git init` する。
  以降の「Commit」ステップは git を使う場合のみ実行（使わないなら飛ばしてよい）。
- **テストフレームワークは無い**。本プランの「テスト」は2種類:
  1. **ロジック自己テスト**: `index.html` に `runSelfTests()` を仕込み、`location.hash==='#test'`
     のとき `console.assert` で検証。ブラウザで `index.html#test` を開き DevTools コンソールを見る。
  2. **手動ブラウザ確認**: 画面を開いてクリックし、指定の表示・遷移を目視。
- すべての新規 UI 文言は既存トーン（ひらがな多め・絵文字・全角まじり）に合わせる。
- 既存3モード（mode 0/1/2）の挙動は一切変えない。

---

## File Structure

変更は `index.html` の1ファイルに閉じる（単一ファイル方針を維持）。

- `index.html`
  - `<head><style>`: ①たば&ばら と ②A型筆算 用の最小 CSS クラスを追加。
  - `#home` セクション: 4つ目のステージボタン＋小メニュー DOM を追加。
  - `<script>` IIFE 内:
    - 追加グローバル: `variant`（'A'|'B'）, `timed`（bool）。
    - 追加: 整数問題プール `INT_POOL` / `INT_POOL_R`、`genInt()`、`hissanModel()`、
      `runSelfTests()`、`buildBundles()`/`checkBundles()`、`render4Step()`系、
      `renderLongInt()`。
    - 改修（mode===3 分岐を足す）: `STAGE_NAMES`、`gen()`、`buildBeakers()`、
      `checkBeaker()`、`goCalc()`、`goCheck()`、`buildPlan()`、`loadQuestion()`、
      `reloadSameQuestion()`、`startQTimer()`、`startGame()`、ステージボタンの click 配線。

---

## Task 1: 整数問題プールと `genInt()`（ロジック＋自己テスト）

**Files:**
- Modify: `index.html`（`<script>` IIFE 内、`MODE0R` 定義の直後あたり）

純粋ロジックなので自己テストを先に書く。

- [ ] **Step 1: 自己テスト土台 `runSelfTests()` を追加**

`index.html` の IIFE の最後（`})();` の直前）に追加。`#test` のときだけ走る。

```javascript
  function runSelfTests(){
    var fails=0;
    function ok(cond,msg){ if(!cond){ fails++; console.error('FAIL:',msg); } else { console.log('ok:',msg); } }
    window.__divTests={ok:ok, get fails(){return fails;}};
    // 各タスクで下にテストを足していく
    console.log('=== self tests ===');
    // <<TESTS>>
    console.log('=== done, fails =', fails, '===');
  }
  if(location.hash==='#test') runSelfTests();
```

- [ ] **Step 2: プール生成のテストを書く（まだ失敗する）**

`runSelfTests()` の `// <<TESTS>>` を次に置き換える。

```javascript
    // --- INT_POOL: 商は必ず2ケタ・わられる数は2ケタ ---
    ok(typeof INT_POOL!=='undefined' && INT_POOL.length>0, 'INT_POOL exists & non-empty');
    INT_POOL.forEach(function(P){
      ok(P.a>=10 && P.a<=99, 'a is 2-digit: '+P.a);
      ok(P.ans>=10 && P.ans<=99, 'quotient 2-digit: '+P.a+'/'+P.b+'='+P.ans);
      ok(P.a===P.b*P.ans, 'exact divide: '+P.a+'='+P.b+'*'+P.ans);
      ok(P.b>=2 && P.b<=9, 'divisor 2..9: '+P.b);
    });
    INT_POOL_R.forEach(function(P){
      ok(P.a===P.b*P.ans+P.rem, 'rem exact: '+P.a+'='+P.b+'*'+P.ans+'+'+P.rem);
      ok(P.rem>0 && P.rem<P.b, 'rem in range: '+P.rem+'<'+P.b);
      ok(P.ans>=10 && P.ans<=99 && P.a<=99, 'rem case ranges: '+P.a);
    });
    // --- genInt ---
    var g=genInt(false,'A',3);
    ok(g.mode===3 && g.b===3 && g.hasRem===false, 'genInt no-rem mode3 b=3');
    ok(g.tens*10+g.ones===g.a, 'tens/ones decompose a');
    var gr=genInt(true,'A',null);
    ok(gr.hasRem===true && gr.rem>0, 'genInt rem case');
```

- [ ] **Step 3: テストが失敗することを確認**

Run: ブラウザで `index.html#test` を開き DevTools コンソールを見る。
Expected: `INT_POOL is not defined` 系の ReferenceError（または FAIL 多数）。

- [ ] **Step 4: プールと `genInt()` を実装**

既存 `MODE0R=[]...` 行の直後に追加。

```javascript
  // ===== 整数 2ケタ÷1ケタ プール（商は必ず2ケタ＝十の位に商が立つ） =====
  var INT_POOL=[], INT_POOL_R=[];
  for(var ib=2; ib<=9; ib++){
    for(var iq=10; iq<=99; iq++){
      var ia=ib*iq;
      if(ia>=10 && ia<=99){ INT_POOL.push({a:ia,b:ib,ans:iq,rem:0}); }
    }
  }
  for(var jb=2; jb<=9; jb++){
    for(var jq=10; jq<=99; jq++){
      for(var jr=1; jr<jb; jr++){
        var ja=jb*jq+jr;
        if(ja>=10 && ja<=99){ INT_POOL_R.push({a:ja,b:jb,ans:jq,rem:jr}); }
      }
    }
  }

  function genInt(withRem, variantSel, forceB){
    var o={mode:3, hasRem:!!withRem, variant:variantSel||'A'};
    var pool=(withRem?INT_POOL_R:INT_POOL).filter(function(P){ return forceB? P.b===forceB : true; });
    if(pool.length===0) pool=(withRem?INT_POOL_R:INT_POOL);
    var P=pick(pool);
    o.a=P.a; o.b=P.b; o.ans=P.ans; o.rem=P.rem||0;
    o.tens=Math.floor(P.a/10); o.ones=P.a%10;
    o.containers=P.b;
    o.story='🍬 '+P.a+'こ の あめを '+P.b+'人に 同じだけ！';
    o.prob=P.a+' ÷ '+P.b;
    o.beakerInstr='まず「10のたば」から 分けよう！'+(withRem?' 分けきれない分が あまり':'');
    o.hint='10のたば'+o.tens+'本＋ばら'+o.ones+'本。十の位→一の位の順に '+P.b+'人へ。<br>'
         +'こたえ '+P.ans+(withRem?' あまり '+o.rem:'')+'。';
    return o;
  }
```

- [ ] **Step 5: テストが通ることを確認**

Run: ブラウザで `index.html#test` を開きコンソールを見る。
Expected: 上記 `INT_POOL`/`genInt` 系の `ok:` ログのみ、`FAIL:` が無い。`fails = 0`。

- [ ] **Step 6: Commit（git 使用時のみ）**

```bash
git add index.html
git commit -m "feat: 整数2ケタ÷1ケタの問題プールとgenIntを追加"
```

---

## Task 2: 筆算モデル `hissanModel()`（A型筆算の土台ロジック）

A型の段組み表示と既存 `longDivision()` を使い、各位の「積・ひいた残り・おろし」を
列位置つきで返す純粋関数。ここを正しく作れば描画が楽になる。

**Files:**
- Modify: `index.html`（`longDivision()` の直後）

- [ ] **Step 1: テストを書く（`runSelfTests()` の末尾、`done` ログの前に追加）**

```javascript
    // --- hissanModel ---
    var hm=hissanModel(48,3);
    ok(hm.qd.join('')==='16', 'hm 48/3 quotient 16');
    ok(hm.finalRem===0, 'hm 48/3 rem 0');
    // rows: [product pos0, remainder pos0, product pos1, remainder pos1]
    ok(hm.rows.length===4, 'hm rows=4');
    ok(hm.rows[0].kind==='product' && hm.rows[0].val==='3' && hm.rows[0].endCol===0, 'p0 = 3 @col0');
    ok(hm.rows[1].kind==='remainder' && hm.rows[1].val==='18' && hm.rows[1].endCol===1, 'rem0 brings -> 18 @col1');
    ok(hm.rows[2].val==='18' && hm.rows[2].endCol===1, 'p1 = 18 @col1');
    ok(hm.rows[3].val==='0' && hm.rows[3].endCol===1, 'rem1 = 0 @col1');
    // leading-zero case: 62/2 = 31 ; 6/2=3 r0 bring 2 -> "2" (not "02")
    var hm2=hissanModel(62,2);
    ok(hm2.rows[1].val==='2', '62/2 no leading zero in brought value: '+hm2.rows[1].val);
    // remainder case: 53/4 = 13 r1
    var hm3=hissanModel(53,4);
    ok(hm3.qd.join('')==='13' && hm3.finalRem===1, 'hm 53/4 = 13 r1');
    ok(hm3.rows[hm3.rows.length-1].val==='1', '53/4 final remainder shows 1');
```

- [ ] **Step 2: 失敗を確認**

Run: `index.html#test` をリロード。
Expected: `hissanModel is not defined`（FAIL）。

- [ ] **Step 3: `hissanModel()` を実装**

`longDivision(...)` 関数の閉じ `}` の直後に追加。

```javascript
  // A型筆算の段組みモデル：各位の積／ひいた残り＋おろし を列位置つきで返す
  function hissanModel(a,b){
    var ld=longDivision(a,b);
    var digits=String(a).split('').map(Number);
    var rows=[];
    ld.steps.forEach(function(st,i){
      // 積（商d×b）：いま処理した桁 i に右そろえ
      rows.push({kind:'product', pos:i, endCol:i, val:String(st.sub)});
      var isLast=(i===ld.steps.length-1);
      var val, endCol;
      if(!isLast){
        // ひいた残り st.rem に 次の桁を おろす。残り0なら先頭0は書かない
        val=(st.rem===0? '' : String(st.rem)) + String(digits[i+1]);
        endCol=i+1;
      } else {
        val=String(st.rem); endCol=i; // 最後＝あまり（0ならわりきれ）
      }
      rows.push({kind:'remainder', pos:i, endCol:endCol, val:val});
    });
    return {ld:ld, digits:digits, N:digits.length, rows:rows, qd:ld.qd, finalRem:ld.finalRem};
  }
```

- [ ] **Step 4: 成功を確認**

Run: `index.html#test` をリロード。
Expected: `hm`/`hm2`/`hm3` 系すべて `ok:`、`fails = 0`。

- [ ] **Step 5: Commit（git 使用時のみ）**

```bash
git add index.html
git commit -m "feat: A型筆算の段組みモデルhissanModelを追加"
```

---

## Task 3: ホームの4つ目ボタン＋小メニュー、`startGame` 拡張、`gen`/`STAGE_NAMES` 分岐

ここから画面配線。まず新モードに入れるようにする（①②③の中身は次タスク以降で順に作る。
この時点では既存の beaker フェーズに mode3 が落ちても落ちないよう、最低限のディスパッチを置く）。

**Files:**
- Modify: `index.html`（`#home` セクション、`<style>`、IIFE 内）

- [ ] **Step 1: 小メニューの CSS を追加**

`<style>` 内、`.s3{...}` 行の直後に追加。

```css
  .s4{border-color:#0aa; box-shadow:0 5px 0 #088;}
  #intMenu{margin:-4px 0 12px; padding:10px; border:3px dashed #0aa; border-radius:16px; background:#e8fbff;}
  #intMenu .mtitle{font-size:12px; font-weight:800; color:#077; text-align:center; margin-bottom:8px;}
  #intMenu .mbtns{display:grid; grid-template-columns:1fr 1fr; gap:8px;}
  #intMenu button{border:3px solid var(--ink); border-radius:14px; padding:10px 6px; cursor:pointer; background:#fff;
    font-family:inherit; font-weight:800; font-size:14px; color:var(--ink); box-shadow:0 4px 0 var(--ink);}
  #intMenu button:active{transform:translateY(2px); box-shadow:0 2px 0 var(--ink);}
  #intMenu button small{display:block; font-weight:700; font-size:11px; color:#888; margin-top:2px;}
```

- [ ] **Step 2: ホームに4つ目ボタンと小メニュー DOM を追加**

`#home` の `<button class="stage-btn s3" ...>...</button>` の直後、`<div class="best" id="bestline">` の前に追加。

```html
    <button class="stage-btn s4" id="intStageBtn"><span class="emoji">➗</span>2ケタ ÷ 1ケタ ラボ<small>たば＆ばらで分けて、はじめての筆算！</small></button>
    <div id="intMenu" class="hidden">
      <div class="mtitle">どっちで やる？</div>
      <div class="mbtns">
        <button data-variant="A" data-timed="0">🌱 れんしゅう<small>タイマー無し・4ステップ筆算</small></button>
        <button data-variant="B" data-timed="1">🔥 ちょうせん<small>タイマーあり・数字えらび</small></button>
      </div>
    </div>
```

- [ ] **Step 3: グローバル変数を追加**

IIFE 内、`var mode=0, qIndex=0, ...` の行の末尾（同じ var 群）に追加。既存行:
`var mode=0, qIndex=0, combo=0, maxCombo=0, rightCount=0;` を次へ変更。

```javascript
  var mode=0, qIndex=0, combo=0, maxCombo=0, rightCount=0, variant='A', timed=true;
```

- [ ] **Step 4: `STAGE_NAMES` に4つ目を追加**

既存:
```javascript
  var STAGE_NAMES=['🧪 小数 ÷ 整数 ラボ','📏 整数 ÷ 小数 ラボ','🥤 小数 ÷ 小数 ラボ'];
```
を次へ変更。

```javascript
  var STAGE_NAMES=['🧪 小数 ÷ 整数 ラボ','📏 整数 ÷ 小数 ラボ','🥤 小数 ÷ 小数 ラボ','➗ 2ケタ ÷ 1ケタ ラボ'];
```

- [ ] **Step 5: `gen()` に mode3 分岐を追加**

`gen()` の `if(m===0){ ... } else if(m===1){ ... } else { ... }` の冒頭、`if(m===0)` の前に分岐を足す。
`gen` 先頭を次のようにする（既存の `var o={...}` 行はそのまま、その直後に追加）。

```javascript
  function gen(m, withRem, forceB){
    if(m===3){ return genInt(withRem, variant, forceB); }
    var o={mode:m, hasRem:!!withRem};
```

（以降の既存コードは変更しない。）

- [ ] **Step 6: `startGame()` を variant/timed 対応に拡張**

既存:
```javascript
  function startGame(m){
    mode=m; qIndex=0; combo=0; maxCombo=0; rightCount=0; dead=false;
    $('comboN').textContent=0; $('progressbar').style.width='0%';
    buildPlan();
    show('game'); loadQuestion(); startTime=Date.now();
  }
```
を次へ変更。

```javascript
  function startGame(m, variantSel, timedSel){
    mode=m; variant=variantSel||'A'; timed=(timedSel===undefined)? true : !!timedSel;
    qIndex=0; combo=0; maxCombo=0; rightCount=0; dead=false;
    $('comboN').textContent=0; $('progressbar').style.width='0%';
    $('timePill').classList.toggle('hidden', !timed);
    buildPlan();
    show('game'); loadQuestion(); startTime=Date.now();
  }
```

- [ ] **Step 7: ステージボタンの配線を更新**

既存:
```javascript
  [].forEach.call(document.querySelectorAll('.stage-btn'),function(b){ b.onclick=function(){ startGame(parseInt(b.dataset.mode)); }; });
```
を次へ変更（既存3ボタンは従来どおり、4つ目はメニュー開閉）。

```javascript
  [].forEach.call(document.querySelectorAll('.stage-btn'),function(b){
    if(b.id==='intStageBtn') return;
    b.onclick=function(){ startGame(parseInt(b.dataset.mode), 'B', true); };
  });
  $('intStageBtn').onclick=function(){ $('intMenu').classList.toggle('hidden'); };
  [].forEach.call(document.querySelectorAll('#intMenu button'),function(b){
    b.onclick=function(){ startGame(3, b.dataset.variant, b.dataset.timed==='1'); };
  });
```

- [ ] **Step 8: 既存モードが壊れていないか手動確認**

Run: `index.html` を開く。
Expected: ホームに「➗ 2ケタ÷1ケタ ラボ」ボタンが出る。クリックで小メニュー（れんしゅう/ちょうせん）が開閉する。
既存の3モードを開始でき、これまで通り動く（この時点で mode3 開始はまだ ① が未対応なので押さなくてよい）。

- [ ] **Step 9: Commit（git 使用時のみ）**

```bash
git add index.html
git commit -m "feat: ホームに2ケタ÷1ケタボタンと小メニュー・mode3配線を追加"
```

---

## Task 4: フェーズ① たば＆ばらで分ける（`buildBundles`/`checkBundles`）

mode3 の ① を、既存 `#beakerPhase`（`#beakerArea`/`#beakerCtrl`/`#checkBeaker`）に描画する。
十の位（たば）→ 一の位（ばら）の順に「1人ぶん何本ずつ」をスライダーで決める方式にする。

**Files:**
- Modify: `index.html`（`<style>`、`buildBeakers()`、`checkBeaker()`、IIFE 内に新規関数）

- [ ] **Step 1: たば＆ばら用 CSS を追加**

`<style>` 内、`.beaker-area{...}` 行の前に追加。

```css
  .bundles{display:flex; flex-direction:column; gap:10px; align-items:center; margin:6px 0;}
  .brow{display:flex; align-items:center; gap:6px; flex-wrap:wrap; justify-content:center;}
  .brow .blab{font-size:12px; font-weight:800; color:#077; width:80px; text-align:right;}
  .tab{display:inline-block; width:10px; height:30px; background:linear-gradient(180deg,#ffd23f,#ff9a1f); border:2px solid var(--ink); border-radius:3px;}
  .bara{display:inline-block; width:10px; height:10px; background:#00d9c0; border:2px solid var(--ink); border-radius:50%;}
  .bgroup{display:inline-flex; gap:3px; padding:3px 5px; border:2px dashed #bbb; border-radius:8px; min-height:20px;}
  .bperson{display:inline-flex; flex-direction:column; align-items:center; gap:2px; font-size:11px; font-weight:800; color:#555;}
  .intslider-row{display:flex; flex-direction:column; align-items:center; gap:4px; margin-top:8px;}
  .intslider-row .sval{font-size:14px; font-weight:800; color:var(--juice2);}
```

- [ ] **Step 2: `buildBeakers()` に mode3 分岐を追加**

既存 `buildBeakers()` の先頭 `var area=$('beakerArea'), ctrl=$('beakerCtrl'); area.innerHTML=''; ctrl.innerHTML='';` の直後に追加。

```javascript
    if(current.mode===3){ buildBundles(); return; }
```

- [ ] **Step 3: `buildBundles()` を実装**

IIFE 内、`makeBeaker(...)` 関数の直前に追加。`bundleStage`（'tens'|'ones'）と1人あたり本数をスライダーで決める。

```javascript
  var bundleStage='tens', tensPer=0, onesPer=0;
  function buildBundles(){
    bundleStage='tens'; tensPer=0; onesPer=0;
    renderBundles();
  }
  function renderBundles(){
    var area=$('beakerArea'), ctrl=$('beakerCtrl');
    var b=current.b, tens=current.tens, ones=current.ones;
    // 配り済み計算
    var tensGiven=tensPer*b, tensLeft=tens-tensGiven;
    // 十の位であまったたばは ばら10本へ
    var onesPool=ones + (bundleStage==='ones'? tensLeft*10 : 0);
    var onesGiven=onesPer*b, onesLeft=onesPool-onesGiven;
    function tabs(n){ var s=''; for(var i=0;i<n;i++) s+='<span class="tab"></span>'; return s; }
    function baras(n){ var s=''; for(var i=0;i<n;i++) s+='<span class="bara"></span>'; return s; }
    var people='';
    for(var p=0;p<b;p++){
      people+='<div class="bperson"><div class="bgroup">'+tabs(tensPer)+baras(onesPer)+'</div>'+(p+1)+'人目</div>';
    }
    var srcTabs = bundleStage==='tens'? tensLeft : 0;
    var srcBaras = bundleStage==='tens'? ones : onesLeft;
    area.innerHTML=
      '<div class="bundles">'
      +'<div class="brow"><span class="blab">のこり→</span><div class="bgroup">'+tabs(srcTabs)+baras(srcBaras)+'</div></div>'
      +'<div class="brow" style="gap:12px">'+people+'</div>'
      +'</div>';
    var label = bundleStage==='tens'
      ? '十の位：たば'+tens+'本を '+b+'人に 同じだけ（1人 何本ずつ？）'
      : 'あまったたば'+(tens-tensPer*b)+'本→ばら'+((tens-tensPer*b)*10)+'本に！ ばら'+onesPool+'本を '+b+'人に';
    var maxPer = bundleStage==='tens'? Math.floor(tens/b) : Math.floor(onesPool/b);
    var cur = bundleStage==='tens'? tensPer : onesPer;
    ctrl.innerHTML=
      '<div class="intslider-row" style="width:100%">'
      +'<div style="font-size:12px;font-weight:800;color:#077;text-align:center">'+label+'</div>'
      +'<input type="range" class="hslider" id="intSlider" min="0" max="'+Math.max(maxPer,0)+'" step="1" value="'+cur+'">'
      +'<div class="sval">1人 <span id="intPerVal">'+cur+'</span> 本ずつ</div>'
      +'</div>';
    var sl=$('intSlider');
    sl.oninput=function(){ if(dead)return; var v=parseInt(sl.value);
      if(bundleStage==='tens') tensPer=v; else onesPer=v;
      $('intPerVal').textContent=v; renderBundles(); };
    $('checkBeaker').textContent = bundleStage==='tens'? '十の位を 分けた！' : 'これで 分けた！';
  }
```

- [ ] **Step 4: `checkBeaker()` に mode3 分岐を追加**

既存 `checkBeaker()` 先頭 `if(dead)return; var ok=false, msg='';` の直後に追加。

```javascript
    if(current.mode===3){ checkBundles(); return; }
```

- [ ] **Step 5: `checkBundles()` を実装**

IIFE 内、`checkBeaker()` 関数の直前に追加。十の位の段では「正しく配れたら一の位へ」、
一の位の段では商・あまりが合っていれば ② へ。

```javascript
  function checkBundles(){
    var b=current.b, fb=$('fb');
    function shake(){ var box=$('wrap'); box.classList.remove('shake'); void box.offsetWidth; box.classList.add('shake'); }
    if(bundleStage==='tens'){
      var want=Math.floor(current.tens/b);
      if(tensPer===want){ bundleStage='ones'; onesPer=0; fb.innerHTML='十の位OK！ あまったたばを ばらにくずすよ'; fb.style.color='#149a52'; renderBundles(); }
      else { fb.innerHTML='たば'+current.tens+'本を '+b+'人に 同じだけ。1人 '+want+'本ずつだよ'; fb.style.color='#c23b3b'; shake(); }
      return;
    }
    // ones stage
    var onesTarget=current.ans%10;       // 一の位の商
    var leftover=current.rem;             // あまり
    if(onesPer===onesTarget){
      fb.innerHTML='ぴったり！ 1人 '+current.ans+'こ'+(current.hasRem? '、あまり '+leftover+'こ':'')+'！';
      fb.style.color='#149a52'; $('checkBeaker').disabled=true; setTimeout(goCalc,900);
    } else {
      fb.innerHTML='ばらを '+b+'人に 同じだけ。1人 '+onesTarget+'本ずつだよ'+(current.hasRem? '（分けきれない分が あまり）':''); fb.style.color='#c23b3b'; shake();
    }
  }
```

- [ ] **Step 6: 手動確認（れんしゅうで ① を通す）**

Run: `index.html` →「➗2ケタ÷1ケタ」→「🌱 れんしゅう」。
Expected: たば＆ばらが出る。十の位スライダーを正解にして「十の位を分けた！」→一の位段へ。
一の位を正解にして「これで分けた！」→（②はまだ未実装なら次タスク後に遷移）。タイマーピルは出ていない。

- [ ] **Step 7: Commit（git 使用時のみ）**

```bash
git add index.html
git commit -m "feat: フェーズ①たば&ばら分けを実装"
```

---

## Task 5: フェーズ② B型筆算（数字えらび）＋ `goCalc` 分岐＋タイマー無効化

先に軽い B型（既存 `renderLong` の整数版）を通して、①→②→（③は既存流用）→次問 の骨格を完成させる。
A型は次タスク。

**Files:**
- Modify: `index.html`（`goCalc()`、`renderLong()` のあまり文言、`startQTimer()`、IIFE 内）

- [ ] **Step 1: `startQTimer()` を timed 対応にする**

既存 `startQTimer()` 先頭を次へ変更（`timed===false` のときカウントダウンしない）。

既存:
```javascript
  function startQTimer(limit){
    clearInterval(qTimer); qTimeLeft=limit; var danger=Math.max(5,limit/3);
    $('timeN').textContent=qTimeLeft.toFixed(1); $('timePill').classList.remove('danger');
    qTimer=setInterval(function(){ ... },100);
  }
```
を次へ変更。

```javascript
  function startQTimer(limit){
    clearInterval(qTimer);
    if(!timed){ $('timePill').classList.add('hidden'); return; }
    qTimeLeft=limit; var danger=Math.max(5,limit/3);
    $('timeN').textContent=qTimeLeft.toFixed(1); $('timePill').classList.remove('danger');
    qTimer=setInterval(function(){ qTimeLeft-=0.1; if(qTimeLeft<=danger)$('timePill').classList.add('danger');
      if(qTimeLeft<=0){ qTimeLeft=0; $('timeN').textContent='0.0'; explode(); return; } $('timeN').textContent=qTimeLeft.toFixed(1); },100);
  }
```

- [ ] **Step 2: `goCalc()` に mode3 分岐を追加**

既存 `goCalc()` 末尾:
```javascript
    if(current.mode===0){ calcState={ld:longDivision(current.a10,current.b),stepIdx:0}; renderLong(); }
    else { buildTenfold(); }
```
を次へ変更。

```javascript
    if(current.mode===0){ calcState={ld:longDivision(current.a10,current.b),stepIdx:0}; renderLong(); }
    else if(current.mode===3){
      if(current.variant==='A'){ render4Step(); }
      else { calcState={ld:longDivision(current.a,current.b),stepIdx:0}; renderLong(); }
    }
    else { buildTenfold(); }
```

- [ ] **Step 3: `renderLong()` のあまり文言を mode 別にする**

`renderLong()` 内の `tail` を作る箇所:
```javascript
      var tail = !isLast ? '（つぎに '+ld.steps[s+1].bring+' を下ろす）'
               : (st.rem===0 ? '（わりきれた！）' : '（のこり <span class="remhi">'+(st.rem/10).toFixed(1)+'L</span> が あまり！元のビーカーと同じ）');
```
を次へ変更（整数モードはL表記にしない）。

```javascript
      var remTxt = current.mode===3 ? ('（のこり <span class="remhi">'+st.rem+'</span> が あまり！）')
                                    : ('（のこり <span class="remhi">'+(st.rem/10).toFixed(1)+'L</span> が あまり！元のビーカーと同じ）');
      var tail = !isLast ? '（つぎに '+ld.steps[s+1].bring+' を下ろす）'
               : (st.rem===0 ? '（わりきれた！）' : remTxt);
```

- [ ] **Step 4: 手動確認（ちょうせんで B型を通す）**

Run: `index.html` →「➗2ケタ÷1ケタ」→「🔥 ちょうせん」。
Expected: タイマーピルが出る。① を分けたあと ② で各位の商を4択で選べる（既存と同じ見た目）。
正解で ③ たしかめ算へ進む（③の選択肢は次タスクで整数化。今は小数 delta で表示が変でも遷移はする）。

- [ ] **Step 5: Commit（git 使用時のみ）**

```bash
git add index.html
git commit -m "feat: mode3のB型筆算とタイマー無効化(れんしゅう)を実装"
```

---

## Task 6: フェーズ③ たしかめ算の整数対応（`goCheck` の選択肢生成）

**Files:**
- Modify: `index.html`（`goCheck()`）

- [ ] **Step 1: `goCheck()` の選択肢生成に mode3 分岐を追加**

`goCheck()` 内、選択肢を作る部分:
```javascript
    var correct=a; // もとの数に戻る
    var set={}; set[correct]=1; var arr=[correct];
    var deltas=[0.1,-0.1,0.2,-0.2,1,-1,b*0.1,-b*0.1];
    while(arr.length<4){ var w=Math.round((correct+pick(deltas))*10)/10; if(w>0&&!set[w]){set[w]=1;arr.push(w);} }
    arr.sort(function(x,y){return x-y;});
```
を次へ変更。

```javascript
    var correct=a; // もとの数に戻る
    var set={}; set[correct]=1; var arr=[correct];
    if(current.mode===3){
      var ideltas=[1,-1,2,-2,b,-b,10,-10];
      while(arr.length<4){ var iw=correct+pick(ideltas); if(iw>0&&!set[iw]){set[iw]=1;arr.push(iw);} }
    } else {
      var deltas=[0.1,-0.1,0.2,-0.2,1,-1,b*0.1,-b*0.1];
      while(arr.length<4){ var w=Math.round((correct+pick(deltas))*10)/10; if(w>0&&!set[w]){set[w]=1;arr.push(w);} }
    }
    arr.sort(function(x,y){return x-y;});
```

- [ ] **Step 2: 手動確認**

Run: `index.html` →「➗2ケタ÷1ケタ」→「🔥 ちょうせん」を最後まで進める。
Expected: ③ たしかめ算で「？÷b = 商（あまり R）」が出て、選択肢は **整数のみ**・正解1つ。
正解すると「商×b（＋あまり）＝もと」で次へ。わりきれ問題とあまり問題の両方で確認。

- [ ] **Step 3: Commit（git 使用時のみ）**

```bash
git add index.html
git commit -m "feat: たしかめ算を整数(mode3)に対応"
```

---

## Task 7: フェーズ② A型筆算 `render4Step()`（フル4ステップ・ガイド）

れんしゅうの核心。`hissanModel()` を使い、各位で たてる→かける→ひく→おろす を順にタップ。
段組みは「商の行・わる数)わられる数 の行・各 rows（積/残り）」を列そろえで描く。

**Files:**
- Modify: `index.html`（`<style>`、IIFE 内に `render4Step` 系）

- [ ] **Step 1: A型筆算用 CSS を追加**

`<style>` 内、`.calc-box{...}` 行の直後に追加。

```css
  .h4{display:inline-block; font-family:'Mochiy Pop One',sans-serif;}
  .h4 .hr{display:grid; grid-auto-flow:column; justify-content:start; column-gap:0; height:34px; align-items:center;}
  .h4 .hc{width:30px; text-align:center; font-size:26px;}
  .h4 .lead{width:34px; text-align:right; font-size:22px; padding-right:2px;}
  .h4 .qrow .hc{color:#149a52;}
  .h4 .qblank{display:inline-block; width:24px; height:30px; line-height:26px; border:3px dashed var(--orange); border-radius:8px; color:#c98a3a; background:#fff7e6;}
  .h4 .qblank.active{background:var(--yellow); animation:blink 1s infinite;}
  .h4 .barline{border-top:3px solid var(--ink);}
  .h4 .dim{opacity:.28;}
  .step4btns{display:flex; justify-content:center; gap:6px; flex-wrap:wrap; margin:8px 0;}
  .step4btns button{border:3px solid var(--ink); border-radius:12px; padding:8px 12px; font-family:inherit; font-weight:800; font-size:14px; background:#fff; color:var(--ink); box-shadow:0 4px 0 var(--ink); cursor:pointer;}
  .step4btns button.on{background:var(--cyan); animation:blink 1s infinite;}
  .step4btns button.done{background:#7CFFB2; border-color:#149a52; box-shadow:0 4px 0 #149a52;}
  .step4btns button:disabled{opacity:.4; cursor:not-allowed;}
```

- [ ] **Step 2: `render4Step()` 系を実装**

IIFE 内、`buildTenfold()` 関数の直前に追加。状態 `step4` で位置と手順を管理する。
各位の手順は `['tate','kake','hiku','oroshi']`（最終位は `oroshi` 無し）。
`tate`=商を4択、`kake`/`hiku`/`oroshi`=該当ボタンを押すと段が出現。

```javascript
  var step4=null;
  function render4Step(){
    if(!step4){
      var hm=hissanModel(current.a, current.b);
      step4={hm:hm, pos:0, sub:'tate', q:[], shownProd:[], shownRemDigits:[]};
      // shownRemDigits[pos] = 0(なし)/1(ひいた残りまで)/2(おろし済み)
      for(var i=0;i<hm.qd.length;i++){ step4.shownProd.push(false); step4.shownRemDigits.push(0); }
    }
    drawHissan(); drawStepButtons(); drawStepInstr();
  }

  function cellRowHTML(text, endCol, N, extraClass){
    // text の各文字を endCol から左へ並べた N 列グリッド行を返す
    var cells=[]; for(var c=0;c<N;c++) cells[c]='<span class="hc"></span>';
    var startCol=endCol-(text.length-1);
    for(var k=0;k<text.length;k++){ var col=startCol+k; if(col>=0&&col<N) cells[col]='<span class="hc">'+text.charAt(k)+'</span>'; }
    return '<div class="hr '+(extraClass||'')+'"><span class="lead"></span>'+cells.join('')+'</div>';
  }

  function drawHissan(){
    var hm=step4.hm, N=hm.N, b=current.b;
    // 商の行
    var qcells=[]; for(var c=0;c<N;c++){
      if(c<step4.q.length) qcells[c]='<span class="hc">'+step4.q[c]+'</span>';
      else if(c===step4.pos && step4.sub==='tate') qcells[c]='<span class="hc"><span class="qblank active">?</span></span>';
      else qcells[c]='<span class="hc"><span class="qblank">_</span></span>';
    }
    var html='<div class="h4">';
    html+='<div class="hr qrow"><span class="lead"></span>'+qcells.join('')+'</div>';
    // わる数 ) わられる数
    var dcells=[];
    String(current.a).split('').forEach(function(ch,c){ dcells[c]='<span class="hc">'+ch+'</span>'; });
    html+='<div class="hr"><span class="lead">'+b+')</span>'+dcells.join('')+'</div>';
    // 各 rows を順に（pos ごとに product → remainder）
    hm.rows.forEach(function(row){
      var visible=false, text=row.val, cls='';
      if(row.kind==='product'){ visible=step4.shownProd[row.pos]; cls='barline'; }
      else { // remainder：ひいた残り→おろし の2段階表示
        var stg=step4.shownRemDigits[row.pos];
        if(stg===0){ visible=false; }
        else if(stg===1){ // ひいた残りだけ（おろし前）
          visible=true;
          var st=hm.ld.steps[row.pos];
          text = String(st.rem); // 残りのみ（最終位もこれ）
          // 残り0かつ最終でない場合は空文字になるが endCol は次桁。ここでは残りのみ表示
        } else { visible=true; text=row.val; }
      }
      if(!visible) return;
      html+=cellRowHTML(text, row.endCol, N, '');
    });
    html+='</div>';
    $('calcBox').innerHTML=html;
  }

  function drawStepInstr(){
    var hm=step4.hm, b=current.b, pos=step4.pos;
    var st=hm.ld.steps[pos];
    var names={tate:'たてる', kake:'かける', hiku:'ひく', oroshi:'おろす'};
    var detail={
      tate:'いま '+st.cur+' ÷ '+b+' の 商は？',
      kake:'商 '+step4.q[pos]+' × '+b+' を 書こう（ボタン）',
      hiku:st.cur+' − '+(st.qd*b)+' を しよう（ボタン）',
      oroshi:'つぎの '+(hm.digits[pos+1])+' を おろそう（ボタン）'
    };
    $('instr').innerHTML='<b>'+names[step4.sub]+'</b>：'+detail[step4.sub];
  }

  function drawStepButtons(){
    var hm=step4.hm;
    var last=(step4.pos===hm.qd.length-1);
    var order=last? ['tate','kake','hiku'] : ['tate','kake','hiku','oroshi'];
    var ch=$('choices'); ch.innerHTML='';
    if(step4.sub==='tate'){
      // 商えらび（4択）
      var correct=hm.qd[step4.pos];
      makeDigitChoices(correct).forEach(function(v){
        var bb=document.createElement('button'); bb.className='choice'; bb.textContent=v;
        bb.onclick=function(){ pick4Tate(v,bb,correct); }; ch.appendChild(bb);
      });
    }
    // 手順ボタン（たてる以外を押せる状態に）
    var wrapB=document.createElement('div'); wrapB.className='step4btns';
    var labels={tate:'たてる',kake:'かける',hiku:'ひく',oroshi:'おろす'};
    order.forEach(function(s){
      var bb=document.createElement('button'); bb.textContent=labels[s];
      var doneOrder=order.indexOf(s)<order.indexOf(step4.sub);
      if(doneOrder) bb.classList.add('done');
      if(s===step4.sub && s!=='tate') bb.classList.add('on');
      if(s==='tate') bb.disabled=true; // たてるは上の4択で行う
      bb.onclick=function(){ press4Step(s); };
      wrapB.appendChild(bb);
    });
    var host=$('calcBox'); host.appendChild(wrapB);
  }

  function pick4Tate(v,btn,correct){
    if(dead)return;
    if(v===correct){ btn.classList.add('right'); step4.q[step4.pos]=correct; step4.sub='kake'; $('fb').innerHTML='商 '+correct+' を たてた！'; $('fb').style.color='#149a52'; render4Step(); }
    else { btn.classList.add('wrong'); missedThisQ=true; $('fb').innerHTML='この位の 商は '+correct+' だよ'; $('fb').style.color='#c23b3b';
      var box=$('wrap'); box.classList.remove('shake'); void box.offsetWidth; box.classList.add('shake');
      step4.q[step4.pos]=correct; setTimeout(function(){ step4.sub='kake'; render4Step(); },700); }
  }

  function press4Step(s){
    if(dead)return;
    if(s!==step4.sub){ $('fb').innerHTML='つぎは「'+({tate:'たてる',kake:'かける',hiku:'ひく',oroshi:'おろす'})[step4.sub]+'」だよ'; $('fb').style.color='#c23b3b';
      var box=$('wrap'); box.classList.remove('shake'); void box.offsetWidth; box.classList.add('shake'); return; }
    var hm=step4.hm, pos=step4.pos, last=(pos===hm.qd.length-1);
    if(s==='kake'){ step4.shownProd[pos]=true; step4.sub='hiku'; render4Step(); return; }
    if(s==='hiku'){ step4.shownRemDigits[pos]=1; 
      if(last){ // 最終位：ここで完了
        setTimeout(goCheck,700);
        $('fb').innerHTML=(hm.finalRem===0?'わりきれた！':'あまり '+hm.finalRem+'！'); $('fb').style.color='#149a52';
      } else { step4.sub='oroshi'; }
      render4Step(); return; }
    if(s==='oroshi'){ step4.shownRemDigits[pos]=2; step4.pos++; step4.sub='tate'; render4Step(); return; }
  }
```

- [ ] **Step 3: 手動確認（れんしゅうで A型を最後まで）**

Run: `index.html` →「➗2ケタ÷1ケタ」→「🌱 れんしゅう」。①を分けて②へ。
Expected:
- 各位で「たてる（4択）→かける→ひく→おろす」の順に進む。順番外のボタンを押すと「つぎは◯◯だよ」。
- 段組みが列そろえで増えていく（例 48÷3：商16、3、18、18、0）。
- 最終位の「ひく」で「わりきれた！」または「あまり N！」→ ③ へ。
- あまり問題（例 53÷4）でも最終あまり1が表示され、たしかめが 13×4+1=53。

- [ ] **Step 4: `render4Step` の状態リセットを確認（重要）**

`step4` は問題をまたいで残ると壊れる。次タスクの `loadQuestion`/`reloadSameQuestion`/`goCalc`
で問題開始時に `step4=null` へ戻す配線を入れる（Task 8 で対応）。本タスクでは単一問題で動けばよい。

- [ ] **Step 5: Commit（git 使用時のみ）**

```bash
git add index.html
git commit -m "feat: A型フル4ステップ筆算render4Stepを実装"
```

---

## Task 8: 難易度プラン・問題リセット・最終結線・全体確認

複数問・やり直し・結果画面まで通す。`step4`/`bundleStage` のリセットを必ず入れる。

**Files:**
- Modify: `index.html`（`buildPlan()`、`loadQuestion()`、`reloadSameQuestion()`、`goCalc()`、`endGame()`）

- [ ] **Step 1: `buildPlan()` に mode3 分岐を追加**

既存 `buildPlan()` 先頭:
```javascript
  function buildPlan(){
    qPlan=[];
    if(mode!==0){ for(var i=0;i<TOTAL_Q;i++) qPlan.push({b:null,rem:false}); return; }
```
を次へ変更（mode3 もプランを作る：前半わりきれ＆小さいわる数、後半あまり＆大きいわる数）。

```javascript
  function buildPlan(){
    qPlan=[];
    if(mode===3){
      var bases=[2,3,2,4,3,5,6,4,7,8]; // 前半小さめ→後半大きめ
      for(var k=0;k<bases.length-1;k++){ if(Math.random()<0.3){ var t=bases[k]; bases[k]=bases[k+1]; bases[k+1]=t; } }
      // 後半 index4 以降から3つを あまりあり に
      var rpool=[]; for(var j=4;j<TOTAL_Q;j++) rpool.push(j);
      rpool.sort(function(){return Math.random()-0.5;});
      var rset={}; rpool.slice(0,REM_COUNT).forEach(function(idx){ rset[idx]=true; });
      for(var q=0;q<TOTAL_Q;q++) qPlan.push({b:bases[q], rem:!!rset[q]});
      return;
    }
    if(mode!==0){ for(var i=0;i<TOTAL_Q;i++) qPlan.push({b:null,rem:false}); return; }
```

- [ ] **Step 2: `loadQuestion()` を mode3 のプラン適用＆状態リセット対応にする**

既存 `loadQuestion()` 先頭:
```javascript
  function loadQuestion(){
    var plan = (mode===0 && qPlan[qIndex]) ? qPlan[qIndex] : {b:null, rem:false};
    current=gen(mode, mode===0 ? plan.rem : false, mode===0 ? plan.b : null);
```
を次へ変更。

```javascript
  function loadQuestion(){
    var usePlan = (mode===0 || mode===3);
    var plan = (usePlan && qPlan[qIndex]) ? qPlan[qIndex] : {b:null, rem:false};
    current=gen(mode, usePlan ? plan.rem : false, usePlan ? plan.b : null);
    step4=null; bundleStage='tens'; tensPer=0; onesPer=0;
```

（残りの既存行はそのまま。）

- [ ] **Step 3: `reloadSameQuestion()` でも状態リセット**

既存 `reloadSameQuestion()` 先頭 `phase='beaker'; missedThisQ=false;` の直後に追加。

```javascript
    step4=null; bundleStage='tens'; tensPer=0; onesPer=0;
```

- [ ] **Step 4: `goCalc()` 開始時に `step4` を初期化（保険）**

既存 `goCalc()` 先頭 `phase='calc';` の直後に追加。

```javascript
    if(current.mode===3 && current.variant==='A') step4=null;
```

- [ ] **Step 5: たしかめ算フェーズで `phaseTag` の番号表記を確認**

`goCheck()` は phaseTag を「③ たしかめ算」にする既存処理を流用。mode3 でも同様でよい（変更不要）。
確認のみ。

- [ ] **Step 6: れんしゅう編の終了演出を確認**

`endGame()` は `byBoom` で分岐。れんしゅう（timed=false）は時間切れが無いので
常に `endGame(false)`（ぜんぶクリア）に到達する。`getBest`/`setBest` は mode をキーにするため、
れんしゅうとちょうせんで同じ mode=3 のベストを共有する。これは許容（タイム表示自体は出る）。
**確認のみ・変更不要。** 気になる場合の改善は本プランのスコープ外。

- [ ] **Step 7: 全体通し確認（受け入れテスト）**

Run: `index.html` で次をすべて確認。
- [ ] 既存3モードがこれまで通り動く（リグレッション無し）。
- [ ] れんしゅう：10問、タイマー無し・バクハツ無し。①たば&ばら→②A型4ステップ→③たしかめ。
- [ ] れんしゅう後半にあまり問題が出る（例 53÷4=13あまり1 が成立）。
- [ ] ちょうせん：タイマー作動、時間切れでバクハツ→同じ問題から再開。②はB型（数字えらび）。
- [ ] 商が必ず2ケタ（十の位に商が立つ）。`index.html#test` で `fails = 0`。
- [ ] たしかめ算の選択肢は整数のみ・正解1つ。
- [ ] 結果画面までいく（10問クリアで 🏆）。

- [ ] **Step 8: 自己テスト最終確認**

Run: `index.html#test` を開きコンソール。
Expected: `=== done, fails = 0 ===`。

- [ ] **Step 9: Commit（git 使用時のみ）**

```bash
git add index.html
git commit -m "feat: mode3の難易度プラン・状態リセット・最終結線"
```

---

## Self-Review メモ（作成者チェック済み）

- **Spec coverage**: 追加位置/メニュー(Task3)・数の範囲と商2ケタ保証(Task1)・①たば&ばら(Task4)・
  ②A型(Task7)/B型(Task5)・③たしかめ整数化(Task6)・わりきれ→あまりの難易度(Task1/8)・
  タイマー無し/有り(Task3/5)・既存非変更(各タスクで分岐追加のみ) をカバー。
- **Placeholders**: 各コードステップに実コードを記載。`<<TESTS>>`/`<<...>>` はテストの追記位置を示す
  実在マーカーで、Task1で実体に置換済み。
- **Type/名前整合**: `genInt/hissanModel/buildBundles/checkBundles/render4Step/cellRowHTML/`
  `drawHissan/drawStepButtons/drawStepInstr/pick4Tate/press4Step` と
  グローバル `variant/timed/step4/bundleStage/tensPer/onesPer/INT_POOL/INT_POOL_R` を
  全タスクで一貫使用。`current` のフィールド（a,b,ans,rem,tens,ones,variant,hasRem）も一貫。
- **既知の許容事項**: れんしゅう/ちょうせんは mode=3 のベストタイムを共有（Task8 Step6）。スコープ外。
