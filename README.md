<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>電気設計マスター・プロフェッショナル (接地線サイズ自動計算版)</title>
    <style>
        :root { 
            --primary: #0f172a; --accent: #2563eb; --danger: #e11d48; 
            --warning: #f59e0b; --success: #10b981; --bg: #f8fafc; 
            --ground: #16a34a; /* 接地用カラー */
        }
        body { font-family: 'Helvetica Neue', Arial, 'Hiragino Kaku Gothic ProN', Meiryo, sans-serif; background: var(--bg); margin: 0; padding: 20px; color: #334155; font-size: 13px; }
        .container { max-width: 1400px; margin: auto; display: grid; grid-template-columns: 450px 1fr; gap: 20px; }
        .card { background: white; padding: 20px; border-radius: 12px; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1); border: 1px solid #e2e8f0; height: fit-content; }
        
        .tab-bar { grid-column: 1 / span 2; display: flex; gap: 5px; margin-bottom: -10px; }
        .tab-btn { padding: 12px 24px; border: 1px solid #e2e8f0; border-bottom: none; background: #e2e8f0; border-radius: 12px 12px 0 0; cursor: pointer; font-weight: bold; color: #64748b; transition: 0.3s; }
        .tab-btn.active { background: white; color: var(--accent); border-top: 3px solid var(--accent); }

        h1 { font-size: 1.2rem; margin: 0 0 15px 0; display: flex; align-items: center; gap: 8px; color: var(--primary); border-bottom: 2px solid #e2e8f0; padding-bottom: 8px; }
        h2 { font-size: 0.8rem; margin: 15px 0 8px 0; color: #64748b; text-transform: uppercase; letter-spacing: 0.05em; border-left: 3px solid var(--accent); padding-left: 8px; }
        
        .input-group { background: #f1f5f9; padding: 12px; border-radius: 8px; margin-bottom: 12px; }
        .grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
        label { display: block; font-size: 0.7rem; font-weight: 700; margin-bottom: 4px; color: #475569; }
        select, input { width: 100%; padding: 8px; border: 1px solid #cbd5e1; border-radius: 6px; box-sizing: border-box; font-size: 0.9rem; }

        .mode-selector { display: flex; gap: 2px; background: #e2e8f0; padding: 2px; border-radius: 6px; margin-bottom: 8px; }
        .mode-btn { flex: 1; border: none; background: none; padding: 6px; cursor: pointer; border-radius: 4px; font-size: 0.75rem; font-weight: bold; }
        .mode-btn.active { background: white; box-shadow: 0 2px 4px rgba(0,0,0,0.1); color: var(--accent); }

        .status-banner { padding: 15px; border-radius: 8px; text-align: center; font-weight: bold; font-size: 1.1rem; margin-bottom: 20px; border: 2px solid transparent; }
        .safe { background: #ecfdf5; color: var(--success); border-color: var(--success); }
        .warn { background: #fffbeb; color: var(--warning); border-color: var(--warning); }
        .danger { background: #fff1f2; color: var(--danger); border-color: var(--danger); }

        .result-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 15px; margin-bottom: 20px; }
        .stat-card { background: #fff; border: 1px solid #e2e8f0; padding: 15px; border-radius: 10px; position: relative; overflow: hidden; }
        .stat-card::before { content: ""; position: absolute; left: 0; top: 0; height: 100%; width: 4px; background: var(--accent); }
        .stat-card.ground::before { background: var(--ground); }
        .stat-value { font-size: 1.4rem; font-weight: 800; display: block; color: var(--primary); margin-bottom: 4px; }
        .stat-label { font-size: 0.7rem; color: #64748b; font-weight: bold; }

        .details-container { display: grid; grid-template-columns: 1fr; gap: 20px; }
        .evidence-box { background: white; border: 1px solid #e2e8f0; border-radius: 12px; padding: 15px; }
        .formula-box { font-family: 'Courier New', Courier, monospace; background: #2d3436; color: #dfe6e9; padding: 12px; border-radius: 8px; font-size: 0.8rem; line-height: 1.6; margin-top: 10px; }
        
        .btn-print { background: var(--primary); color: white; border: none; padding: 10px 20px; border-radius: 6px; cursor: pointer; width: 100%; margin-top: 10px; font-weight: bold; }
        .hidden { display: none; }

        @media print {
            .card:first-child, .btn-print, .tab-bar { display: none; }
            .container { display: block; }
            body { background: white; padding: 0; }
        }
    </style>
</head>
<body>

<div class="container">
    <div class="tab-bar">
        <button class="tab-btn active" id="tab-branch" onclick="switchTab('branch')">⚡ 分岐回路</button>
        <button class="tab-btn" id="tab-feeder" onclick="switchTab('feeder')">🏢 幹線系統</button>
    </div>

    <div class="card">
        <h1>⚙️ <span id="ui-title">末端負荷</span> 設計条件</h1>
        
        <h2>1. 電源・負荷条件</h2>
        <div class="input-group">
            <label>配線方式・電圧</label>
            <select id="sys-type" onchange="calculate()"></select>
            
            <div id="demand-box" class="hidden" style="margin-top:10px;">
                <label>需要率 : <span id="val-demand">80</span>%</label>
                <input type="range" id="demand-rate" min="10" max="100" step="5" value="80" oninput="calculate()">
            </div>

            <div class="grid-2" style="margin-top:10px;">
                <div>
                    <label>入力モード</label>
                    <div class="mode-selector">
                        <button class="mode-btn active" id="btn-W" onclick="setMode('W')">W</button>
                        <button class="mode-btn" id="btn-kW" onclick="setMode('kW')">kW</button>
                        <button class="mode-btn" id="btn-A" onclick="setMode('A')">A</button>
                    </div>
                </div>
                <div>
                    <label>負荷値</label>
                    <input type="number" id="main-input" value="1500" oninput="calculate()">
                </div>
            </div>
            <label style="margin-top:10px;">力率 (cos φ) : <span id="val-pf">100</span>%</label>
            <input type="range" id="pf-percent" min="50" max="100" step="1" value="100" oninput="calculate()">
        </div>

        <h2>2. 環境・施工条件</h2>
        <div class="input-group">
            <div class="grid-2">
                <div>
                    <label>周囲温度 (℃)</label>
                    <select id="temp" onchange="calculate()">
                        <option value="30">30℃以下</option>
                        <option value="40" selected>40℃</option>
                        <option value="50">50℃</option>
                    </select>
                </div>
                <div>
                    <label>同一管内・束線数</label>
                    <select id="bundle" onchange="calculate()">
                        <option value="1">1 (単独)</option>
                        <option value="3">2〜3本</option>
                        <option value="4">4本</option>
                        <option value="6">5〜6本</option>
                    </select>
                </div>
            </div>
        </div>

        <h2>3. 電線・ケーブル選定</h2>
        <div class="input-group">
            <label>電線種別 (仕様書準拠)</label>
            <select id="wire-cat" onchange="updateSizes()"></select>
            <label style="margin-top:8px;">サイズ</label>
            <select id="wire-size" onchange="calculate()"></select>
            <label style="margin-top:8px;">配線長 : <span id="val-length">20</span>m</label>
            <input type="range" id="length" min="1" max="500" value="20" oninput="calculate()">
        </div>

        <h2>4. 配管選定 (G/C/E/PF/CD)</h2>
        <div class="input-group">
            <div class="grid-2">
                <div>
                    <label>配管種別</label>
                    <select id="conduit-cat" onchange="updateConduitSizes()">
                        <option value="G">厚鋼電線管 (G)</option>
                        <option value="C">薄鋼電線管 (C)</option>
                        <option value="E">ねじなし電線管 (E)</option>
                        <option value="PF">PF管 (合成樹脂)</option>
                        <option value="CD">CD管 (埋込用)</option>
                    </select>
                </div>
                <div>
                    <label>呼び径</label>
                    <select id="conduit-size" onchange="calculate()"></select>
                </div>
            </div>
        </div>

        <h2>5. 保護機器</h2>
        <div class="input-group">
            <label>遮断器定格 (AT)</label>
            <select id="breaker-at" onchange="calculate()"></select>
        </div>

        <button class="btn-print" onclick="window.print()">🖨️ 設計計算書を出力</button>
    </div>

    <div class="card">
        <h1>📊 設計計算・判定結果</h1>
        <div id="status-banner" class="status-banner">判定中...</div>

        <div class="result-grid">
            <div class="stat-card">
                <span class="stat-label">設計負荷電流</span>
                <span class="stat-value" id="res-calc-a">0.00 A</span>
            </div>
            <div class="stat-card">
                <span class="stat-label">補正後許容電流</span>
                <span class="stat-value" id="res-limit-a">0.0 A</span>
            </div>
            <div class="stat-card">
                <span class="stat-label">電圧降下率</span>
                <span class="stat-value" id="res-v-rate">0.0 %</span>
            </div>
            <div class="stat-card ground">
                <span class="stat-label">推奨接地線サイズ</span>
                <span class="stat-value" id="res-ground-size">-</span>
            </div>
            <div class="stat-card">
                <span class="stat-label">選択中の配管</span>
                <span class="stat-value" id="res-conduit">-</span>
            </div>
        </div>

        <div class="details-container">
            <div class="evidence-box">
                <h2>📋 技術判定・エビデンス</h2>
                <div id="evidence-list"></div>
                <h2>🧮 計算式プロセス</h2>
                <div id="formula-process" class="formula-box"></div>
            </div>
        </div>
    </div>
</div>

<script>
let currentMode = 'W';
let activeTab = 'branch';

// 電線データベース
const db = {
    "IV (一般ビニル)": [
        {s:"1.6mm", a:19, r:9.2}, {s:"2.0mm", a:24, r:5.8}, {s:"5.5SQ", a:34, r:3.5}, {s:"8SQ", a:42, r:2.41}, {s:"14SQ", a:61, r:1.36}
    ],
    "HIV (耐熱ビニル)": [
        {s:"1.6mm", a:27, r:9.2}, {s:"2.0mm", a:35, r:5.8}, {s:"5.5SQ", a:49, r:3.5}, {s:"8SQ", a:61, r:2.41}, {s:"14SQ", a:88, r:1.36}
    ],
    "EM-IE/IC (エコ耐燃性)": [
        {s:"1.6mm", a:27, r:9.2}, {s:"2.0mm", a:35, r:5.8}, {s:"5.5SQ", a:49, r:3.5}, {s:"14SQ", a:88, r:1.36}, {s:"22SQ", a:115, r:0.86}
    ],
    "CV/EM-CE (2心)": [
        {s:"2.0SQ", a:25, r:9.42}, {s:"3.5SQ", a:33, r:5.38}, {s:"5.5SQ", a:44, r:3.42}, {s:"8SQ", a:54, r:2.35}, {s:"14SQ", a:77, r:1.34}, {s:"22SQ", a:105, r:0.85}
    ],
    "CV/EM-CE (3心)": [
        {s:"2.0SQ", a:20, r:9.42}, {s:"3.5SQ", a:27, r:5.38}, {s:"5.5SQ", a:36, r:3.42}, {s:"8SQ", a:45, r:2.35}, {s:"14SQ", a:64, r:1.34}, {s:"22SQ", a:87, r:0.85}
    ],
    "CVT/EM-CET (トリプレックス)": [
        {s:"8SQ", a:62/0.82, r:2.31}, {s:"14SQ", a:86/0.82, r:1.30}, {s:"22SQ", a:110/0.82, r:0.824},
        {s:"38SQ", a:155/0.82, r:0.487}, {s:"60SQ", a:210/0.82, r:0.303}, {s:"100SQ", a:290/0.82, r:0.180},
        {s:"150SQ", a:380/0.82, r:0.118}, {s:"200SQ", a:465/0.82, r:0.091}, {s:"250SQ", a:535/0.82, r:0.072},
        {s:"325SQ", a:635/0.82, r:0.055}, {s:"400SQ", a:725/0.82, r:0.044}, {s:"500SQ", a:835/0.82, r:0.035}
    ]
};

// 接地線サイズ判定テーブル (内線規程・D種/C種接地)
const groundingDb = [
    {at: 20, size: "1.6mm"},
    {at: 30, size: "2.0mm"},
    {at: 60, size: "5.5SQ"},
    {at: 100, size: "8SQ"},
    {at: 200, size: "14SQ"},
    {at: 400, size: "22SQ"},
    {at: 500, size: "38SQ"},
    {at: 600, size: "60SQ"},
    {at: 9999, size: "要詳細計算"}
];

const conduitDb = {
    "G": ["16", "22", "28", "36", "42", "54", "70", "82", "92", "104"],
    "C": ["19", "25", "31", "39", "51", "63", "75"],
    "E": ["19", "25", "31", "39", "51", "63", "75"],
    "PF": ["14", "16", "22", "28", "36", "42", "54"],
    "CD": ["14", "16", "22", "28", "36", "42", "54"]
};

const sysOptions = {
    branch: [{v:"12-100", t:"単相2線 100V"}, {v:"13-200", t:"単相3線 200V"}, {v:"33-200", t:"三相3線 200V"}],
    feeder: [{v:"13-200", t:"単相3線 200V"}, {v:"33-200", t:"三相3線 200V"}, {v:"33-6600", t:"三相3線 6600V"}]
};

const tempFactors = { 30: 1.0, 40: 0.82, 50: 0.58 };
const bundleFactors = { 1: 1.0, 3: 0.7, 4: 0.63, 6: 0.56 };

function switchTab(tab) {
    activeTab = tab;
    document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
    document.getElementById('tab-' + tab).classList.add('active');
    document.getElementById('ui-title').innerText = (tab === 'branch') ? "末端負荷" : "幹線系統";
    const demandBox = document.getElementById('demand-box');
    (tab === 'feeder') ? demandBox.classList.remove('hidden') : demandBox.classList.add('hidden');
    initDropdowns();
}

function initDropdowns() {
    const sysSel = document.getElementById('sys-type');
    sysSel.innerHTML = '';
    sysOptions[activeTab].forEach(opt => {
        let o = document.createElement('option'); o.value = opt.v; o.innerText = opt.t;
        sysSel.appendChild(o);
    });

    const bSel = document.getElementById('breaker-at');
    bSel.innerHTML = '';
    [15, 20, 30, 40, 50, 60, 75, 100, 125, 150, 175, 200, 225, 250, 300, 400, 500, 600].forEach(amp => {
        let opt = document.createElement('option'); opt.value = amp; opt.innerText = amp + " A";
        bSel.appendChild(opt);
    });
    bSel.value = (activeTab === 'branch') ? 20 : 100;

    const catSel = document.getElementById('wire-cat');
    catSel.innerHTML = '';
    for (let cat in db) {
        let opt = document.createElement('option'); opt.value = cat; opt.innerText = cat;
        catSel.appendChild(opt);
    }
    updateSizes();
    updateConduitSizes();
}

function updateSizes() {
    const cat = document.getElementById('wire-cat').value;
    const sizeSel = document.getElementById('wire-size');
    sizeSel.innerHTML = '';
    db[cat].forEach((item, idx) => {
        let opt = document.createElement('option'); opt.value = idx; opt.innerText = item.s;
        sizeSel.appendChild(opt);
    });
    calculate();
}

function updateConduitSizes() {
    const cat = document.getElementById('conduit-cat').value;
    const sizeSel = document.getElementById('conduit-size');
    sizeSel.innerHTML = '';
    conduitDb[cat].forEach(size => {
        let opt = document.createElement('option'); opt.value = size; opt.innerText = cat + size;
        sizeSel.appendChild(opt);
    });
    calculate();
}

function setMode(m) {
    currentMode = m;
    document.querySelectorAll('.mode-btn').forEach(b => b.classList.remove('active'));
    document.getElementById('btn-' + m).classList.add('active');
    calculate();
}

function calculate() {
    const sys = document.getElementById('sys-type').value;
    const loadVal = parseFloat(document.getElementById('main-input').value) || 0;
    const pf_d = parseFloat(document.getElementById('pf-percent').value) / 100;
    const demand_d = parseFloat(document.getElementById('demand-rate').value) / 100;
    const length = parseFloat(document.getElementById('length').value);
    const temp = document.getElementById('temp').value;
    const bundle = document.getElementById('bundle').value;
    const breakerAT = parseFloat(document.getElementById('breaker-at').value);
    const cat = document.getElementById('wire-cat').value;
    const wireIdx = document.getElementById('wire-size').value;
    const selectedWire = db[cat][wireIdx];
    const conduitCat = document.getElementById('conduit-cat').value;
    const conduitSize = document.getElementById('conduit-size').value;

    document.getElementById('val-pf').innerText = (pf_d * 100).toFixed(0);
    document.getElementById('val-demand').innerText = (demand_d * 100).toFixed(0);
    document.getElementById('val-length').innerText = length;

    let v = parseInt(sys.split('-')[1]);
    let f = sys.startsWith("33") ? 1.732 : (sys === "13-200" ? 1 : 2);
    
    // 1. 負荷電流計算
    let I = 0;
    const effectiveLoad = (activeTab === 'feeder') ? loadVal * demand_d : loadVal;
    if (currentMode === 'A') {
        I = effectiveLoad;
    } else {
        let w = currentMode === 'kW' ? effectiveLoad * 1000 : effectiveLoad;
        I = w / ( (sys.startsWith("33") ? 1.732 : 1) * v * pf_d );
    }
    document.getElementById('res-calc-a').innerText = I.toFixed(2) + " A";

    // 2. 許容電流計算
    const k_temp = tempFactors[temp];
    const k_bundle = bundleFactors[bundle];
    const limitA = selectedWire.a * k_temp * k_bundle;
    document.getElementById('res-limit-a').innerText = limitA.toFixed(1) + " A";

    // 3. 電圧降下計算
    const vDrop = (f * length * I * selectedWire.r) / 1000;
    const vRate = (vDrop / v) * 100;
    document.getElementById('res-v-rate').innerText = vRate.toFixed(2) + " %";
    document.getElementById('res-conduit').innerText = conduitCat + conduitSize;

    // 4. 接地線サイズ判定
    const gSizeObj = groundingDb.find(item => item.at >= breakerAT);
    const gSize = gSizeObj ? gSizeObj.size : "要検討";
    document.getElementById('res-ground-size').innerText = gSize;

    // 5. 判定ロジック
    const banner = document.getElementById('status-banner');
    const list = document.getElementById('evidence-list');
    let html = ""; let status = "safe";

    if (I > limitA) { html += `<div style="color:red">● 【許容電流】 超過: ${I.toFixed(1)}A > ${limitA.toFixed(1)}A</div>`; status = "danger"; }
    if (I > breakerAT) { html += `<div style="color:red">● 【遮断器定格】 超過: トリップの恐れ</div>`; status = "danger"; }
    if (vRate > 3.0) { html += `<div style="color:orange">● 【電圧降下】 3%超過 (設計注意)</div>`; if(status!=="danger") status="warn"; }
    
    html += `<div style="color:var(--ground)">● 【接地線】 遮断器AT ${breakerAT}A に基づき ${gSize} を推奨</div>`;

    if (status === "safe" && I <= limitA && I <= breakerAT) {
        // 安全な場合のメッセージ
    }
    
    list.innerHTML = html;
    banner.className = "status-banner " + status;
    banner.innerText = status === "safe" ? "✅ 設計適正" : (status === "warn" ? "⚠️ 設計注意" : "❌ 設計不適合");

    document.getElementById('formula-process').innerHTML = `
        設計電流 I = ${I.toFixed(2)}A<br>
        補正許容電流 = ${selectedWire.a}A(基底) × ${k_temp}(温度) × ${k_bundle}(束線) = <b>${limitA.toFixed(1)}A</b><br>
        電圧降下 ΔV = (${f} × ${length}m × ${I.toFixed(1)}A × ${selectedWire.r}Ω/km) / 1000 = <b>${vDrop.toFixed(2)}V</b><br>
        接地線選定根拠 = 遮断器定格 ${breakerAT}A に対する内線規程最小サイズ <b>${gSize}</b>
    `;
}

window.onload = () => switchTab('branch');
</script>
</body>
</html>
