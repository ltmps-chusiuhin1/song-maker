<!DOCTYPE html>
<html lang="zh-HK">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>旋律創作器 Pro Max - 藍田循道衞理小學</title>

<script src="https://www.gstatic.com/firebasejs/10.8.1/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.8.1/firebase-firestore-compat.js"></script>

<style>
  :root {
    --bg-color: #0f0f17;
    --panel-bg: rgba(255, 255, 255, 0.05);
    --accent: #7c4dff;
    --c-d: #ff1744; 
    --c-r: #ff9100; 
    --c-m: #ffea00; 
    --c-f: #00e676; 
    --c-s: #00e5ff; 
    --c-l: #2979ff; 
    --c-t: #d500f9; 
    --cell-size: 60px; 
  }

  body {
    font-family: 'PingFang HK', 'Microsoft JhengHei', sans-serif;
    background: radial-gradient(circle at center, #1e1e2f 0%, var(--bg-color) 100%);
    color: white;
    margin: 0;
    min-height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
    overflow-x: hidden;
    /* 防止在手機上長按選取文字或觸發右鍵選單 */
    -webkit-touch-callout: none;
    -webkit-user-select: none;
    user-select: none;
  }

  /* --- 主選單 --- */
  #main-menu {
    text-align: center;
    transition: opacity 0.5s;
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .main-title {
    font-size: 3.5rem;
    margin: 0; 
    background: linear-gradient(45deg, #00e5ff, #7c4dff);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    filter: drop-shadow(0 0 15px rgba(124, 77, 255, 0.5));
    line-height: 1.2;
  }

  #app-subtitle { margin-bottom: 30px; }

  .big-btn {
    padding: 15px 40px;
    font-size: 1.2rem;
    border: none;
    border-radius: 50px;
    background: var(--accent);
    color: white;
    cursor: pointer;
    box-shadow: 0 10px 30px rgba(124, 77, 255, 0.4);
    transition: transform 0.2s, box-shadow 0.2s;
    margin: 10px;
    width: 250px;
  }
  .big-btn.secondary { background: #00e5ff; color: #111; box-shadow: 0 10px 30px rgba(0, 229, 255, 0.4); }
  .big-btn:hover { transform: scale(1.05); }

  /* --- 創作器內頁 --- */
  #app-container {
    display: none;
    flex-direction: column;
    align-items: center;
    width: 95%;
    max-width: 1400px;
    animation: fadeIn 0.5s ease-out;
  }

  @keyframes fadeIn { from { opacity: 0; transform: translateY(20px); } to { opacity: 1; transform: translateY(0); } }

  .config-panel {
    background: var(--panel-bg);
    padding: 20px;
    border-radius: 20px;
    display: flex;
    gap: 15px;
    margin-bottom: 25px;
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255,255,255,0.1);
    flex-wrap: wrap;
    justify-content: center;
    align-items: center;
  }

  .config-group { display: flex; align-items: center; gap: 8px; font-size: 14px; }
  
  select, input[type="number"], input[type="text"] {
    padding: 8px 12px;
    border-radius: 10px;
    background: #2a2a3d;
    color: white;
    border: 1px solid var(--accent);
    outline: none;
  }

  input[type="number"] { width: 70px; }

  .zoom-btn {
    background: #2a2a3d; color: white; border: 1px solid var(--accent);
    border-radius: 8px; width: 32px; height: 32px; cursor: pointer;
    font-size: 18px; display: flex; justify-content: center; align-items: center;
  }
  .zoom-btn:hover { background: var(--accent); }

  /* 隱藏唱名的 checkbox 樣式 */
  .checkbox-label {
    display: flex;
    align-items: center;
    cursor: pointer;
    user-select: none;
    gap: 5px;
  }
  .checkbox-label input[type="checkbox"] {
    accent-color: var(--accent);
    width: 16px;
    height: 16px;
    cursor: pointer;
  }

  /* --- 網格區域 --- */
  .grid-scroll-wrapper {
    width: 100%; 
    overflow-x: auto; 
    padding-bottom: 15px; 
    margin-bottom: 20px;
    scrollbar-width: thin; 
    scrollbar-color: var(--accent) transparent;
    /* 修正了超出版面時最左邊被切斷的問題：移除 justify-content: center */
    /* 確保手機滑動網格區塊的順暢度 */
    touch-action: pan-x; 
  }

  #grid { 
    display: grid; 
    gap: 8px; 
    width: max-content; 
    /* 取代先前的 flex center，使用 margin 讓它小於螢幕時置中，大於螢幕時靠左排列 */
    margin: 0 auto; 
  }

  .cell {
    width: var(--cell-size); height: var(--cell-size);
    background: rgba(255, 255, 255, 0.05); border-radius: 6px;
    cursor: pointer; display: flex; justify-content: center; align-items: center;
    font-weight: bold; font-size: calc(var(--cell-size) * 0.35); 
    color: rgba(255, 255, 255, 0.1); transition: 0.1s;
    position: relative;
    /* 防止手機預設點擊反白或滑動 */
    touch-action: none; 
  }

  .cell:hover { background: rgba(255, 255, 255, 0.15); }

  /* 隱藏唱名模式：強制文字變透明 */
  #grid.hide-names .cell {
    color: transparent !important;
  }

  /* 音符顏色與長音視覺相連邏輯 */
  .cell.active { border-radius: 6px; z-index: 1; }
  .cell.active.tie-right { border-top-right-radius: 0; border-bottom-right-radius: 0; margin-right: -8px; padding-right: 8px; z-index: 2; border-right: none; }
  .cell.active.tie-left { border-top-left-radius: 0; border-bottom-left-radius: 0; margin-left: -8px; padding-left: 8px; z-index: 2; border-left: none; }
  
  /* 顏色設定 */
  .cell.active[data-note="d"], .cell.active[data-note="d'"] { background: var(--c-d); box-shadow: 0 0 10px var(--c-d); color: white; }
  .cell.active[data-note="r"] { background: var(--c-r); box-shadow: 0 0 10px var(--c-r); color: white; }
  .cell.active[data-note="m"] { background: var(--c-m); box-shadow: 0 0 10px var(--c-m); color: #333; }
  .cell.active[data-note="f"] { background: var(--c-f); box-shadow: 0 0 10px var(--c-f); color: white; }
  .cell.active[data-note="s"], .cell.active[data-note="s,"] { background: var(--c-s); box-shadow: 0 0 10px var(--c-s); color: white; }
  .cell.active[data-note="l"], .cell.active[data-note="l,"] { background: var(--c-l); box-shadow: 0 0 10px var(--c-l); color: white; }
  .cell.active[data-note="t"], .cell.active[data-note="t,"] { background: var(--c-t); box-shadow: 0 0 10px var(--c-t); color: white; }

  .cell.playing-column { filter: brightness(1.5) contrast(1.2); }
  .cell.playing-column::after {
    content: ''; position: absolute; top: 0; left: 0; right: 0; bottom: 0;
    background: rgba(255,255,255,0.2); pointer-events: none; border-radius: inherit;
  }

  .bottom-controls { display: flex; gap: 10px; flex-wrap: wrap; justify-content: center; }
  button.action-btn {
    padding: 10px 20px; border-radius: 20px; border: none; cursor: pointer; font-weight: bold; transition: 0.2s;
  }
  .play { background: #00e676; color: #111; }
  .stop { background: #ff1744; color: white; }
  .reset { background: #757575; color: white; }
  .export { background: #00e5ff; color: #111; }
  .cloud { background: #ffea00; color: #111; }
  .back { background: transparent; color: #aaa; border: 1px solid #aaa; }
  button.action-btn:hover { opacity: 0.8; transform: translateY(-2px); }

  /* --- 彈出視窗 (Modal) --- */
  .modal-overlay {
    display: none; position: fixed; top: 0; left: 0; right: 0; bottom: 0;
    background: rgba(0,0,0,0.8); backdrop-filter: blur(5px); z-index: 100;
    justify-content: center; align-items: center;
  }
  .modal-box {
    background: #1e1e2f; padding: 30px; border-radius: 20px; width: 90%; max-width: 400px;
    border: 1px solid var(--accent); box-shadow: 0 10px 40px rgba(124, 77, 255, 0.3);
    display: flex; flex-direction: column; gap: 15px;
  }
  .modal-box h2 { margin-top: 0; color: #00e5ff; text-align: center; }
  .form-row { display: flex; justify-content: space-between; align-items: center; }
  .modal-btn-group { display: flex; justify-content: space-between; margin-top: 15px; }
  
  /* 作品欣賞清單 */
  #works-list {
    max-height: 300px; overflow-y: auto; background: rgba(0,0,0,0.3); border-radius: 10px; padding: 10px;
  }
  .work-item {
    background: rgba(255,255,255,0.1); padding: 10px; border-radius: 8px; margin-bottom: 8px;
    display: flex; justify-content: space-between; align-items: center; cursor: pointer;
  }
  .work-item:hover { background: var(--accent); color: white; }
  .loading { text-align: center; color: #aaa; }

</style>
</head>
<body>

  <div id="main-menu">
    <h1 class="main-title">藍田循道衞理小學</h1>
    <h1 class="main-title" id="app-subtitle">旋律創作器</h1>
    <button class="big-btn" onclick="openApp()">進入創作器</button>
    <button class="big-btn secondary" onclick="openGallery()">作品欣賞</button>
  </div>

  <div id="app-container">
    <div class="config-panel" id="config-panel">
      <div class="config-group">
        <label class="checkbox-label">
          <input type="checkbox" id="toggle-names" checked onchange="toggleNoteNames()"> 顯示唱名
        </label>
      </div>

      <div class="config-group">
        <label>微調大小：</label>
        <button class="zoom-btn" onclick="changeZoom(-5)">-</button>
        <button class="zoom-btn" onclick="changeZoom(5)">+</button>
      </div>
      <div class="config-group">
        <label>音階：</label>
        <select id="scale-select" onchange="initGrid()">
          <option value="ds" selected>d - s (d r m f s)</option>
          <option value="major">大調音階 (d r m f s l t d')</option>
          <option value="minor">小調音階 (l, t, d r m f s l)</option>
          <option value="penta">五聲音階 (s, l, d r m s l d')</option>
        </select>
      </div>
      <div class="config-group">
        <label>長度：</label>
        <select id="length-select" onchange="handleLengthChange()">
          <option value="4">4 拍</option>
          <option value="5" selected>5 拍</option>
          <option value="8">8 拍</option>
          <option value="16">16 拍</option>
          <option value="32">32 拍</option>
          <option value="custom">自訂...</option>
        </select>
        <input type="number" id="custom-length" value="10" min="1" max="128" style="display:none;" onchange="initGrid()">
      </div>
      <div class="config-group">
        <label>音色：</label>
        <select id="wave-select">
          <option value="sine">純淨音 (Sine)</option>
          <option value="triangle">木琴音 (Triangle)</option>
          <option value="square">復古遊戲 (Square)</option>
          <option value="sawtooth">合成器 (Sawtooth)</option>
        </select>
      </div>
      <div class="config-group">
        <label>速度：<span id="bpm-val">120</span></label>
        <input type="range" id="bpm-slider" min="40" max="280" value="120" oninput="updateBPM()">
      </div>
    </div>

    <div class="grid-scroll-wrapper" id="scroll-wrapper">
      <div id="grid"></div>
    </div>

    <div class="bottom-controls" id="bottom-controls">
      <button class="action-btn back" onclick="backToMenu()">返回主選單</button>
      <button class="action-btn reset" onclick="resetGrid()">重置</button>
      <button class="action-btn cloud" onclick="openUploadModal()">☁️ 上載作品</button>
      <button class="action-btn export" onclick="exportAudio()">⬇️ 下載音檔</button>
      <button class="action-btn stop" onclick="stopSequence()">停止</button>
      <button class="action-btn play" id="playBtn" onclick="startPlayback()">▶ 播放</button>
    </div>
  </div>

  <div class="modal-overlay" id="upload-modal">
    <div class="modal-box">
      <h2>上載作品</h2>
      <div class="form-row">
        <label>年級：</label>
        <select id="up-grade">
          <option value="1">1</option><option value="2">2</option><option value="3">3</option>
          <option value="4">4</option><option value="5">5</option><option value="6">6</option>
        </select>
      </div>
      <div class="form-row">
        <label>班別：</label>
        <select id="up-class">
          <option value="A">A</option><option value="B">B</option><option value="C">C</option>
          <option value="D">D</option><option value="E">E</option>
        </select>
      </div>
      <div class="form-row">
        <label>學號：</label>
        <input type="number" id="up-num" min="1" max="40" value="1">
      </div>
      <div class="modal-btn-group">
        <button class="action-btn back" onclick="closeModals()">取消</button>
        <button class="action-btn play" onclick="submitUpload()" id="upload-btn">確認上載</button>
      </div>
    </div>
  </div>

  <div class="modal-overlay" id="gallery-modal">
    <div class="modal-box" style="max-width: 500px;">
      <h2>作品欣賞</h2>
      <div class="form-row" style="justify-content: center; gap: 15px;">
        <select id="gal-grade">
          <option value="1">1年級</option><option value="2">2年級</option><option value="3">3年級</option>
          <option value="4">4年級</option><option value="5">5年級</option><option value="6">6年級</option>
        </select>
        <select id="gal-class">
          <option value="A">A班</option><option value="B">B班</option><option value="C">C班</option>
          <option value="D">D班</option><option value="E">E班</option>
        </select>
        <button class="action-btn cloud" onclick="searchWorks()">搜尋</button>
      </div>
      <div id="works-list">
        <div class="loading">請選擇班級並搜尋</div>
      </div>
      <div class="modal-btn-group" style="justify-content: center;">
        <button class="action-btn back" onclick="closeModals()">關閉</button>
      </div>
    </div>
  </div>

<script>
  // --- Firebase 初始化 ---
  const firebaseConfig = {
    apiKey: "AIzaSyD2H8wX9hk9n4Qp4-8DV9MDnWv1wZGbcqA",
    authDomain: "song-maker-513c5.firebaseapp.com",
    projectId: "song-maker-513c5",
    storageBucket: "song-maker-513c5.firebasestorage.app",
    messagingSenderId: "839788095535",
    appId: "1:839788095535:web:053550098145809ccffeda"
  };
  if (!firebase.apps.length) {
      firebase.initializeApp(firebaseConfig);
  }
  const db = firebase.firestore();

  // --- 基礎變數與設定 ---
  let audioCtx;
  const gridEl = document.getElementById('grid');
  const appContainer = document.getElementById('app-container');
  const mainMenu = document.getElementById('main-menu');

  const freqTable = {
    "d'": 523.25, "t":  493.88, "l":  440.00, "s":  392.00, "f":  349.23,
    "m":  329.63, "r":  293.66, "d":  261.63, "t,": 246.94, "l,": 220.00, "s,": 196.00  
  };

  const scales = {
    ds: ["s", "f", "m", "r", "d"],
    major: ["d'", "t", "l", "s", "f", "m", "r", "d"],
    minor: ["l", "s", "f", "m", "r", "d", "t,", "l,"],
    penta: ["d'", "l", "s", "m", "r", "d", "l,", "s,"] 
  };

  let cells = [];
  let isPlaying = false;
  let timer;
  let currentCellSize = 60;
  
  // 拖曳繪圖變數與群組 ID
  let isDragging = false;
  let dragStartNote = null;
  let dragMode = 'add'; 
  let currentGroupId = null;

  function getLength() {
    const sel = document.getElementById('length-select').value;
    let customVal = parseInt(document.getElementById('custom-length').value) || 8;
    if (customVal > 128) { customVal = 128; document.getElementById('custom-length').value = 128; }
    return (sel === 'custom') ? customVal : parseInt(sel);
  }

  function autoFitGrid() {
    const scaleKey = document.getElementById('scale-select').value;
    const len = getLength();
    const rows = scales[scaleKey].length;
    const availableWidth = (window.innerWidth * 0.9) - ((len - 1) * 8);
    const availableHeight = window.innerHeight - 250 - ((rows - 1) * 8);
    let optimalSize = Math.min(availableWidth / len, availableHeight / rows);
    currentCellSize = Math.max(30, Math.min(100, Math.floor(optimalSize)));
    document.documentElement.style.setProperty('--cell-size', `${currentCellSize}px`);
    gridEl.style.gridTemplateColumns = `repeat(${len}, var(--cell-size))`;
  }

  function changeZoom(delta) {
    currentCellSize = Math.max(30, Math.min(120, currentCellSize + delta));
    document.documentElement.style.setProperty('--cell-size', `${currentCellSize}px`);
    gridEl.style.gridTemplateColumns = `repeat(${getLength()}, var(--cell-size))`;
  }

  function handleLengthChange() {
    const val = document.getElementById('length-select').value;
    document.getElementById('custom-length').style.display = (val === 'custom') ? 'inline-block' : 'none';
    initGrid();
  }

  function updateBPM() { document.getElementById('bpm-val').innerText = document.getElementById('bpm-slider').value; }

  // 隱藏/顯示唱名功能
  function toggleNoteNames() {
    const isChecked = document.getElementById('toggle-names').checked;
    if (isChecked) {
      gridEl.classList.remove('hide-names');
    } else {
      gridEl.classList.add('hide-names');
    }
  }

  // --- 初始化網格與事件綁定 ---
  function initGrid(savedCells = null) {
    stopSequence();
    const scaleKey = document.getElementById('scale-select').value;
    const len = getLength();
    const activeScale = scales[scaleKey];

    gridEl.innerHTML = '';
    autoFitGrid();
    cells = [];
    gridEl.ondragstart = () => false;

    // 確保狀態同步
    toggleNoteNames();

    for (let r = 0; r < activeScale.length; r++) {
      for (let c = 0; c < len; c++) {
        const cell = document.createElement('div');
        cell.classList.add('cell');
        const noteName = activeScale[r];
        cell.dataset.note = noteName;
        cell.dataset.col = c;
        cell.dataset.row = r;
        cell.innerText = noteName;
        
        // 滑鼠事件
        cell.addEventListener('mousedown', (e) => startDrag(cell, e));
        cell.addEventListener('mouseenter', () => continueDrag(cell));
        
        // 手機觸控事件 (點擊開始)
        cell.addEventListener('touchstart', (e) => {
            e.preventDefault(); 
            startDrag(cell, e);
        }, { passive: false });
        
        gridEl.appendChild(cell);
        cells.push(cell);
      }
    }

    window.addEventListener('mouseup', endDrag, { once: false });
    window.addEventListener('touchend', endDrag, { once: false });
    window.addEventListener('touchcancel', endDrag, { once: false });

    if (savedCells) {
      savedCells.sort((a,b) => a.col - b.col);
      let lastNote = null;
      let lastGroup = null;
      let lastCol = -2;

      savedCells.forEach(sc => {
        if (!sc.group) {
           if (sc.note === lastNote && sc.col === lastCol + 1) {
               sc.group = lastGroup; 
           } else {
               lastGroup = 'legacy_' + sc.col;
               sc.group = lastGroup;
           }
        }
        lastNote = sc.note;
        lastCol = sc.col;

        const cell = cells.find(c => parseInt(c.dataset.col) === sc.col && c.dataset.note === sc.note);
        if (cell) {
            cell.classList.add('active');
            cell.dataset.group = sc.group;
        }
      });
      updateTies();
    }
  }

  // --- 觸控滑動繪圖核心 (Touch Move) ---
  gridEl.addEventListener('touchmove', (e) => {
    if (!isDragging) return;
    e.preventDefault(); 

    const touch = e.touches[0];
    const targetElement = document.elementFromPoint(touch.clientX, touch.clientY);
    
    if (targetElement && targetElement.classList.contains('cell')) {
      continueDrag(targetElement);
    }
  }, { passive: false });

  // --- 拖曳繪圖邏輯 ---
  function startDrag(cell, e) {
    if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    if (audioCtx.state === 'suspended') audioCtx.resume();

    isDragging = true;
    dragStartNote = cell.dataset.note;
    
    if (cell.classList.contains('active')) {
      dragMode = 'remove';
      cell.classList.remove('active');
      delete cell.dataset.group;
    } else {
      dragMode = 'add';
      currentGroupId = Date.now().toString() + Math.random().toString();
      
      cells.filter(c => c.dataset.col === cell.dataset.col && c !== cell).forEach(c => {
          c.classList.remove('active');
          delete c.dataset.group;
      });
      
      cell.classList.add('active');
      cell.dataset.group = currentGroupId; 
      playTone(freqTable[cell.dataset.note], 0.2); 
    }
    updateTies();
  }

  function continueDrag(cell) {
    if (!isDragging || cell.dataset.note !== dragStartNote) return;
    
    if (dragMode === 'add') {
      cells.filter(c => c.dataset.col === cell.dataset.col && c !== cell).forEach(c => {
          c.classList.remove('active');
          delete c.dataset.group;
      });
      cell.classList.add('active');
      cell.dataset.group = currentGroupId; 
    } else {
      cell.classList.remove('active');
      delete cell.dataset.group;
    }
    updateTies();
  }

  function endDrag() {
    isDragging = false;
    dragStartNote = null;
  }

  // --- 更新音符的連接狀態 ---
  function updateTies() {
    cells.forEach(c => {
      c.classList.remove('tie-left', 'tie-right');
      c.innerText = c.dataset.note;

      if (c.classList.contains('active')) {
        const col = parseInt(c.dataset.col);
        const groupId = c.dataset.group;
        
        const prevCell = cells.find(x => parseInt(x.dataset.col) === col - 1 && x.dataset.note === c.dataset.note && x.dataset.group === groupId);
        const nextCell = cells.find(x => parseInt(x.dataset.col) === col + 1 && x.dataset.note === c.dataset.note && x.dataset.group === groupId);
        
        if (prevCell && prevCell.classList.contains('active')) {
            c.classList.add('tie-left');
            c.innerText = '-'; 
        }
        if (nextCell && nextCell.classList.contains('active')) {
            c.classList.add('tie-right');
        }
      } else {
        delete c.dataset.group; 
      }
    });
  }

  // --- 音頻播放系統 ---
  function playTone(freq, durationSec, ctx = audioCtx, destination = null) {
    if(!ctx) return;
    const osc = ctx.createOscillator();
    const gain = ctx.createGain();
    const wave = document.getElementById('wave-select').value;
    
    osc.type = wave;
    osc.frequency.setValueAtTime(freq, ctx.currentTime);
    gain.gain.setValueAtTime(0, ctx.currentTime);
    
    let vol = (wave === 'square' || wave === 'sawtooth') ? 0.15 : 0.5;
    
    gain.gain.linearRampToValueAtTime(vol, ctx.currentTime + 0.05);
    gain.gain.setValueAtTime(vol, ctx.currentTime + durationSec - 0.05);
    gain.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + durationSec + 0.1);

    osc.connect(gain);
    gain.connect(destination || ctx.destination);
    osc.start(ctx.currentTime);
    osc.stop(ctx.currentTime + durationSec + 0.15);
  }

  async function startPlayback() {
    if (isPlaying) return;
    if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    if (audioCtx.state === 'suspended') audioCtx.resume();
    
    isPlaying = true;
    const len = getLength();
    const beatTimeSec = 60 / parseInt(document.getElementById('bpm-slider').value);
    
    for (let step = 0; step < len; step++) {
      if (!isPlaying) break;
      cells.forEach(c => c.classList.remove('playing-column'));
      
      const colCells = cells.filter(c => parseInt(c.dataset.col) === step);
      colCells.forEach(c => c.classList.add('playing-column'));
      
      if (colCells.length > 0) {
        colCells[0].scrollIntoView({ behavior: 'smooth', block: 'nearest', inline: 'center' });
      }

      colCells.forEach(c => {
        if (c.classList.contains('active')) {
          if (!c.classList.contains('tie-left')) {
            let noteDurationBeats = 1;
            let currentCheckCol = step + 1;
            while(currentCheckCol < len) {
              const nextC = cells.find(x => parseInt(x.dataset.col) === currentCheckCol && x.dataset.note === c.dataset.note && x.dataset.group === c.dataset.group);
              if (nextC && nextC.classList.contains('active')) {
                noteDurationBeats++;
                currentCheckCol++;
              } else {
                break;
              }
            }
            playTone(freqTable[c.dataset.note], noteDurationBeats * beatTimeSec);
          }
        }
      });

      await new Promise(r => timer = setTimeout(r, beatTimeSec * 1000));
    }
    stopSequence();
  }

  function stopSequence() {
    isPlaying = false;
    clearTimeout(timer);
    cells.forEach(c => c.classList.remove('playing-column'));
  }

  function resetGrid() { stopSequence(); cells.forEach(c => { c.classList.remove('active'); delete c.dataset.group; }); updateTies(); }
  
  function openApp() { mainMenu.style.display = 'none'; appContainer.style.display = 'flex'; initGrid(); }
  function backToMenu() { stopSequence(); appContainer.style.display = 'none'; mainMenu.style.display = 'flex'; }
  function openUploadModal() { document.getElementById('upload-modal').style.display = 'flex'; }
  function openGallery() { document.getElementById('gallery-modal').style.display = 'flex'; document.getElementById('works-list').innerHTML = '<div class="loading">請選擇班級並搜尋</div>'; }
  function closeModals() { 
    document.getElementById('upload-modal').style.display = 'none'; 
    document.getElementById('gallery-modal').style.display = 'none'; 
    stopSequence(); 
  }

  // --- WAV 音檔導出功能 ---
  async function exportAudio() {
    const btn = document.querySelector('.export');
    btn.innerText = "處理中...";
    btn.disabled = true;

    try {
      const len = getLength();
      const bpm = parseInt(document.getElementById('bpm-slider').value);
      const beatTimeSec = 60 / bpm;
      const totalDuration = (len * beatTimeSec) + 1.0; 
      
      const OfflineAudioContextConstructor = window.OfflineAudioContext || window.webkitOfflineAudioContext;
      const totalSamples = Math.ceil(44100 * totalDuration); 
      const offlineCtx = new OfflineAudioContextConstructor(1, totalSamples, 44100);

      let noteEvents = [];
      for(let step=0; step<len; step++) {
        const colCells = cells.filter(c => parseInt(c.dataset.col) === step && c.classList.contains('active') && !c.classList.contains('tie-left'));
        colCells.forEach(c => {
            let noteDurationBeats = 1;
            let currentCheckCol = step + 1;
            while(currentCheckCol < len) {
              const nextC = cells.find(x => parseInt(x.dataset.col) === currentCheckCol && x.dataset.note === c.dataset.note && x.dataset.group === c.dataset.group);
              if (nextC && nextC.classList.contains('active')) {
                noteDurationBeats++;
                currentCheckCol++;
              } else break;
            }
            noteEvents.push({ freq: freqTable[c.dataset.note], startTime: step * beatTimeSec, duration: noteDurationBeats * beatTimeSec });
        });
      }

      const wave = document.getElementById('wave-select').value;
      noteEvents.forEach(evt => {
        const osc = offlineCtx.createOscillator();
        const gain = offlineCtx.createGain();
        osc.type = wave;
        osc.frequency.setValueAtTime(evt.freq, evt.startTime);
        gain.gain.setValueAtTime(0, evt.startTime);
        
        let vol = (wave === 'square' || wave === 'sawtooth') ? 0.15 : 0.5;
        gain.gain.linearRampToValueAtTime(vol, evt.startTime + 0.05);
        gain.gain.setValueAtTime(vol, evt.startTime + evt.duration - 0.05);
        gain.gain.exponentialRampToValueAtTime(0.001, evt.startTime + evt.duration + 0.1);

        osc.connect(gain);
        gain.connect(offlineCtx.destination);
        osc.start(evt.startTime);
        osc.stop(evt.startTime + evt.duration + 0.15);
      });

      const renderedBuffer = await offlineCtx.startRendering();
      const wavData = audioBufferToWav(renderedBuffer);
      const blob = new Blob([new DataView(wavData)], { type: 'audio/wav' });
      const url = URL.createObjectURL(blob);
      
      const a = document.createElement('a');
      a.href = url;
      a.download = 'MyMelody.wav';
      a.click();
      URL.revokeObjectURL(url);
    } catch(e) {
      alert("導出失敗：" + e.message);
    }
    btn.innerText = "⬇️ 下載音檔";
    btn.disabled = false;
  }

  function audioBufferToWav(buffer, opt) {
    opt = opt || {};
    var numChannels = buffer.numberOfChannels;
    var sampleRate = buffer.sampleRate;
    var format = opt.float32 ? 3 : 1;
    var bitDepth = format === 3 ? 32 : 16;
    var result;
    if (numChannels === 2) {
      result = interleave(buffer.getChannelData(0), buffer.getChannelData(1));
    } else {
      result = buffer.getChannelData(0);
    }
    return encodeWAV(result, format, sampleRate, numChannels, bitDepth);
  }
  function encodeWAV(samples, format, sampleRate, numChannels, bitDepth) {
    var bytesPerSample = bitDepth / 8;
    var blockAlign = numChannels * bytesPerSample;
    var buffer = new ArrayBuffer(44 + samples.length * bytesPerSample);
    var view = new DataView(buffer);
    writeString(view, 0, 'RIFF');
    view.setUint32(4, 36 + samples.length * bytesPerSample, true);
    writeString(view, 8, 'WAVE');
    writeString(view, 12, 'fmt ');
    view.setUint32(16, 16, true);
    view.setUint16(20, format, true);
    view.setUint16(22, numChannels, true);
    view.setUint32(24, sampleRate, true);
    view.setUint32(28, sampleRate * blockAlign, true);
    view.setUint16(32, blockAlign, true);
    view.setUint16(34, bitDepth, true);
    writeString(view, 36, 'data');
    view.setUint32(40, samples.length * bytesPerSample, true);
    if (format === 1) floatTo16BitPCM(view, 44, samples);
    else writeFloat32(view, 44, samples);
    return buffer;
  }
  function interleave(inputL, inputR) {
    var length = inputL.length + inputR.length;
    var result = new Float32Array(length);
    var index = 0, inputIndex = 0;
    while (index < length) {
      result[index++] = inputL[inputIndex];
      result[index++] = inputR[inputIndex];
      inputIndex++;
    }
    return result;
  }
  function writeFloat32(output, offset, input) { for (var i = 0; i < input.length; i++, offset += 4) { output.setFloat32(offset, input[i], true); } }
  function floatTo16BitPCM(output, offset, input) {
    for (var i = 0; i < input.length; i++, offset += 2) {
      var s = Math.max(-1, Math.min(1, input[i]));
      output.setInt16(offset, s < 0 ? s * 0x8000 : s * 0x7FFF, true);
    }
  }
  function writeString(view, offset, string) { for (var i = 0; i < string.length; i++) { view.setUint8(offset + i, string.charCodeAt(i)); } }

  // --- Firebase 上載與讀取 ---
  async function submitUpload() {
    const btn = document.getElementById('upload-btn');
    btn.innerText = "上載中...";
    btn.disabled = true;
    
    const activeCellsData = cells.filter(c => c.classList.contains('active')).map(c => ({
      col: parseInt(c.dataset.col),
      note: c.dataset.note,
      group: c.dataset.group || Date.now().toString() 
    }));

    const data = {
      grade: document.getElementById('up-grade').value,
      classId: document.getElementById('up-class').value,
      studentNum: document.getElementById('up-num').value,
      scale: document.getElementById('scale-select').value,
      length: getLength(),
      wave: document.getElementById('wave-select').value,
      bpm: document.getElementById('bpm-slider').value,
      notes: activeCellsData,
      timestamp: firebase.firestore.FieldValue.serverTimestamp()
    };

    try {
      await db.collection("melodies").add(data);
      alert("上載成功！");
      closeModals();
    } catch (e) {
      alert("上載失敗：" + e.message);
    }
    btn.innerText = "確認上載";
    btn.disabled = false;
  }

  async function searchWorks() {
    const grade = document.getElementById('gal-grade').value;
    const classId = document.getElementById('gal-class').value;
    const listDiv = document.getElementById('works-list');
    
    listDiv.innerHTML = '<div class="loading">搜尋中...</div>';

    try {
      const snapshot = await db.collection("melodies")
        .where("grade", "==", grade)
        .where("classId", "==", classId)
        .orderBy("studentNum", "asc")
        .get();

      if (snapshot.empty) {
        listDiv.innerHTML = '<div class="loading">找不到任何作品。</div>';
        return;
      }

      listDiv.innerHTML = '';
      snapshot.forEach(doc => {
        const data = doc.data();
        const item = document.createElement('div');
        item.className = 'work-item';
        item.innerHTML = `<span>${data.grade}${data.classId}班 - ${data.studentNum}號的作品</span> <span>▶️</span>`;
        item.onclick = () => loadAndPlayWork(data);
        listDiv.appendChild(item);
      });
    } catch (e) {
      listDiv.innerHTML = `<div class="loading">發生錯誤：${e.message}</div>`;
    }
  }

  function loadAndPlayWork(data) {
    closeModals();
    appContainer.style.display = 'flex';
    mainMenu.style.display = 'none';

    document.getElementById('scale-select').value = data.scale;
    document.getElementById('wave-select').value = data.wave;
    document.getElementById('bpm-slider').value = data.bpm;
    updateBPM();

    if ([4,5,8,16,32].includes(parseInt(data.length))) {
        document.getElementById('length-select').value = data.length;
        document.getElementById('custom-length').style.display = 'none';
    } else {
        document.getElementById('length-select').value = 'custom';
        document.getElementById('custom-length').value = data.length;
        document.getElementById('custom-length').style.display = 'inline-block';
    }

    initGrid(data.notes);
    
    setTimeout(() => { startPlayback(); }, 500);
  }

  window.addEventListener('resize', () => { if (appContainer.style.display === 'flex') autoFitGrid(); });
</script>
</body>
</html>
