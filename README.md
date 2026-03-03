<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>장치 직접 제어 패널 (Web Serial)</title>
  <style>
    :root {
      --primary: #007bff;
      --primary-dark: #0056b3;
      --bg-color: #f4f7f6;
      --card-bg: #ffffff;
      --text-main: #333333;
      --text-muted: #666666;
      --border-color: #e0e0e0;
      --danger: #dc3545;
      --success: #10b981;
      --warning: #f59e0b;
    }

    * { box-sizing: border-box; margin: 0; padding: 0; -webkit-tap-highlight-color: transparent; }
    
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
      background-color: var(--bg-color);
      color: var(--text-main);
      overflow-x: hidden;
    }

    /* 상단 네비게이션 바 */
    .header {
      position: sticky; top: 0; z-index: 100;
      display: flex; justify-content: space-between; align-items: center;
      background: var(--primary); color: white;
      padding: 12px 16px; box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
    .header h2 { font-size: 18px; font-weight: 600; }
    
    /* 연결 상태 바 */
    .connection-bar {
      background: #fff; padding: 12px 16px; display: flex; gap: 10px;
      align-items: center; border-bottom: 1px solid var(--border-color);
    }
    .status-dot { width: 12px; height: 12px; border-radius: 50%; background: var(--danger); }
    .status-dot.connected { background: var(--success); }
    select.baud-select { padding: 8px; border-radius: 6px; border: 1px solid #ccc; }

    /* 탭 메뉴 */
    .tabs {
      display: flex; background: var(--card-bg); border-radius: 8px;
      overflow: hidden; margin: 16px; box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    }
    .tab {
      flex: 1; padding: 12px; text-align: center; font-size: 15px; font-weight: 500;
      color: var(--text-muted); cursor: pointer; transition: all 0.2s;
    }
    .tab.active { background: var(--primary); color: white; }

    /* 뷰 전환 */
    .tab-content { display: none; padding: 0 16px; animation: fadeIn 0.3s ease; }
    .tab-content.active { display: block; }
    @keyframes fadeIn { from { opacity: 0; transform: translateY(5px); } to { opacity: 1; transform: translateY(0); } }

    /* 카드형 레이아웃 */
    .card {
      background: var(--card-bg); border-radius: 12px; padding: 16px;
      margin-bottom: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.05);
    }
    .card-title { font-size: 14px; font-weight: bold; color: var(--primary); margin-bottom: 12px; text-transform: uppercase; }

    /* 폼 요소 */
    label { display: block; font-size: 13px; color: var(--text-muted); margin-bottom: 6px; }
    input[type="text"] {
      width: 100%; padding: 12px; border: 1px solid var(--border-color);
      border-radius: 8px; font-size: 16px; margin-bottom: 12px;
      background: #fafafa; transition: border 0.2s;
    }
    input[type="text"]:focus { border-color: var(--primary); outline: none; }
    
    /* 버튼 */
    button.btn-main {
      width: 100%; padding: 14px; background: var(--primary); color: white;
      border: none; border-radius: 8px; font-size: 16px; font-weight: 600;
      margin-bottom: 8px; cursor: pointer; transition: background 0.2s;
    }
    button.btn-main:active { transform: scale(0.98); }
    button.btn-danger { background: var(--danger); }
    button.btn-outline { background: transparent; border: 1px solid var(--primary); color: var(--primary); }
    button.btn-connect { background: var(--success); width: auto; padding: 10px 16px; flex: 1; }

    /* 스위치 */
    .switch-row { display: flex; justify-content: space-between; align-items: center; padding: 10px 0; border-bottom: 1px solid #f0f0f0; }
    .switch-row:last-child { border-bottom: none; }
    .switch-label { font-size: 15px; font-weight: 500; }
    .switch { position: relative; width: 50px; height: 28px; }
    .switch-input { opacity: 0; width: 0; height: 0; }
    .switch-track { position: absolute; inset: 0; background: #e5e7eb; border-radius: 28px; transition: 0.3s; }
    .switch-thumb { position: absolute; top: 2px; left: 2px; width: 24px; height: 24px; background: white; border-radius: 50%; transition: 0.3s; box-shadow: 0 2px 4px rgba(0,0,0,0.2); }
    .switch-input:checked ~ .switch-track { background: var(--success); }
    .switch-input:checked ~ .switch-thumb { transform: translateX(22px); }

    /* 슬라이더 */
    .slider-wrap { display: flex; align-items: center; gap: 12px; margin-bottom: 12px; }
    .slider { flex: 1; height: 6px; border-radius: 3px; background: #ddd; outline: none; -webkit-appearance: none; }
    .slider::-webkit-slider-thumb { -webkit-appearance: none; width: 20px; height: 20px; border-radius: 50%; background: var(--primary); }
    .value-box { width: 40px; text-align: center; font-weight: bold; border: 1px solid #ccc; border-radius: 4px; padding: 4px; }

    /* 리스트 (스와이프 삭제) */
    .list-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px; }
    .listbox { width: 100%; max-height: 200px; overflow-y: auto; border: 1px solid var(--border-color); border-radius: 8px; background: #fff; overflow-x: hidden; }
    .swipe-item {
      padding: 12px 16px; border-bottom: 1px solid #eee; background: #fff;
      font-size: 15px; font-weight: 500; touch-action: pan-y;
      transition: transform 0.2s ease, opacity 0.2s ease;
    }
    .swipe-item.dragging { transition: none; }
    .swipe-item.removing { opacity: 0; transform: translateX(100%); }

    /* 로그 박스 */
    #logBox {
      width: 100%; height: 150px; background: #1e1e1e; color: #4af626;
      padding: 12px; font-family: monospace; font-size: 12px;
      overflow-y: auto; border-radius: 8px; margin-top: 16px;
      word-break: break-all;
    }
  </style>
</head>
<body>

  <!-- 상단 헤더 -->
  <div class="header">
    <h2>비상벨 직접 제어 (USB)</h2>
  </div>

  <!-- 시리얼 연결 컨트롤 -->
  <div class="connection-bar">
    <div id="statusDot" class="status-dot"></div>
    <span id="statusText" style="font-weight: bold; font-size: 14px; width: 60px;">연결대기</span>
    <select id="baudRate" class="baud-select">
      <option value="9600">9600 bps</option>
      <option value="115200" selected>115200 bps</option>
    </select>
    <button id="connectBtn" class="btn-main btn-connect" onclick="toggleConnection()">USB 연결</button>
  </div>

  <div class="tabs">
    <div class="tab active" onclick="switchTab('deviceTab', this)">장치 제어</div>
    <div class="tab" onclick="switchTab('settingTab', this)">상세 설정</div>
  </div>

  <!-- 장치 탭 -->
  <div id="deviceTab" class="tab-content active">
    <div class="card">
      <div class="card-title">관제 및 경보</div>
      <label>관제번호</label>
      <div style="display: flex; gap: 8px;">
        <input type="text" id="PhoneNumber" placeholder="번호 입력" style="margin-bottom: 0;">
        <button class="btn-main" style="width: 80px; margin-bottom: 0;" onclick="updatePhoneNumber()">적용</button>
      </div>
      
      <div style="display: flex; gap: 8px; margin-top: 16px;">
        <button class="btn-main btn-danger" onclick="serialWrite('#0121defID9999W13287:1:97:*')">경보 발생</button>
        <button class="btn-main btn-outline" onclick="serialWrite('#0120defID9999W13287:1:4:*')">경보 해제</button>
      </div>
      
      <label style="margin-top: 16px;">스피커 볼륨</label>
      <div class="slider-wrap">
        <input type="range" min="0" max="7" value="0" step="1" class="slider" id="SpeakerVolume">
        <input type="text" class="value-box" id="SpeakerVolumeTextBox" readonly>
      </div>
    </div>

    <div class="card">
      <div class="list-header">
        <div class="card-title" style="margin:0;">무선 RFID 목록</div>
        <button class="btn-main btn-outline" style="width:auto; padding: 6px 12px; margin:0;" onclick="loadRfidList()">🔄 새로고침</button>
      </div>
      <ul class="listbox" id="RfidList"></ul>
      <p style="font-size: 11px; color: var(--text-muted); margin-top: 4px; text-align: right;">* 항목을 오른쪽으로 밀어서 삭제</p>
    </div>
  </div>

  <!-- 설정 탭 -->
  <div id="settingTab" class="tab-content">
    <div class="card">
      <div class="card-title">터치 동작 설정</div>
      <div class="switch-row"><span class="switch-label">경보사용</span><label class="switch"><input type="checkbox" id="touch_alram" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
      <div class="switch-row"><span class="switch-label">발신</span><label class="switch"><input type="checkbox" id="touch_transmit" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
      <div class="switch-row"><span class="switch-label">SMS</span><label class="switch"><input type="checkbox" id="touch_sms" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
      <div class="switch-row"><span class="switch-label">경광등</span><label class="switch"><input type="checkbox" id="touch_light" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
      <div class="switch-row"><span class="switch-label">MP3</span><label class="switch"><input type="checkbox" id="touch_mp3" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
    </div>

    <div class="card">
      <div class="card-title">이상음원 동작 설정</div>
      <div class="switch-row"><span class="switch-label">경보사용</span><label class="switch"><input type="checkbox" id="emerg_alram" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
      <div class="switch-row"><span class="switch-label">발신</span><label class="switch"><input type="checkbox" id="emerg_transmit" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
      <div class="switch-row"><span class="switch-label">SMS</span><label class="switch"><input type="checkbox" id="emerg_sms" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
      <div class="switch-row"><span class="switch-label">경광등</span><label class="switch"><input type="checkbox" id="emerg_light" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
      <div class="switch-row"><span class="switch-label">MP3</span><label class="switch"><input type="checkbox" id="emerg_mp3" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
    </div>

    <div class="card">
      <div class="card-title">무선 동작 설정</div>
      <div class="switch-row"><span class="switch-label">경보사용</span><label class="switch"><input type="checkbox" id="rf_alram" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
      <div class="switch-row"><span class="switch-label">발신</span><label class="switch"><input type="checkbox" id="rf_transmit" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
      <div class="switch-row"><span class="switch-label">SMS</span><label class="switch"><input type="checkbox" id="rf_sms" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
      <div class="switch-row"><span class="switch-label">경광등</span><label class="switch"><input type="checkbox" id="rf_light" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
      <div class="switch-row"><span class="switch-label">MP3</span><label class="switch"><input type="checkbox" id="rf_mp3" class="switch-input"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
    </div>
  </div>

  <!-- 통신 로그 -->
  <div style="padding: 0 16px 16px 16px;">
    <div id="logBox">Web Serial API 대기 중... (HTTPS 환경 필수)</div>
  </div>

  <script>
    // ==========================================
    // 1. Web Serial API 통신 코어
    // ==========================================
    let port;
    let reader;
    let writer;
    let keepReading = true;

    const logBox = document.getElementById("logBox");
    function appendLog(msg) {
      logBox.textContent += msg + "\n";
      if (logBox.textContent.length > 3000) logBox.textContent = logBox.textContent.slice(-2500);
      logBox.scrollTop = logBox.scrollHeight;
    }

    async function toggleConnection() {
      if (port) {
        await disconnectSerial();
      } else {
        await connectSerial();
      }
    }

    async function connectSerial() {
      if (!("serial" in navigator)) {
        alert("이 브라우저는 Web Serial API를 지원하지 않습니다. 안드로이드 크롬을 사용해주세요.");
        return;
      }

      try {
        const baudRate = parseInt(document.getElementById("baudRate").value);
        port = await navigator.serial.requestPort();
        await port.open({ baudRate: baudRate });

        // UI 업데이트
        document.getElementById("statusDot").classList.add("connected");
        document.getElementById("statusText").innerText = "연결됨";
        document.getElementById("statusText").style.color = "var(--success)";
        const btn = document.getElementById("connectBtn");
        btn.innerText = "연결 해제";
        btn.classList.remove("btn-connect");
        btn.classList.add("btn-danger");

        appendLog(`[SYSTEM] 포트 열림 (${baudRate} bps)`);

        // Writer 설정
        const textEncoder = new TextEncoderStream();
        textEncoder.readable.pipeTo(port.writable);
        writer = textEncoder.writable.getWriter();

        // 초기 데이터 요청
        requestInitialData();

        // Reader 루프 시작
        keepReading = true;
        readLoop();

      } catch (err) {
        console.error(err);
        appendLog("[ERROR] 연결 실패: " + err.message);
      }
    }

    async function disconnectSerial() {
      keepReading = false;
      if (reader) { await reader.cancel(); reader = null; }
      if (writer) { await writer.close(); writer = null; }
      if (port) { await port.close(); port = null; }

      document.getElementById("statusDot").classList.remove("connected");
      document.getElementById("statusText").innerText = "연결대기";
      document.getElementById("statusText").style.color = "var(--text-main)";
      const btn = document.getElementById("connectBtn");
      btn.innerText = "USB 연결";
      btn.classList.remove("btn-danger");
      btn.classList.add("btn-connect");
      appendLog("[SYSTEM] 포트 닫힘");
    }

    async function readLoop() {
      const textDecoder = new TextDecoderStream();
      const readableStreamClosed = port.readable.pipeTo(textDecoder.writable);
      reader = textDecoder.readable.getReader();
      let buffer = "";

      try {
        while (keepReading) {
          const { value, done } = await reader.read();
          if (done) break;
          if (value) {
            buffer += value;
            let lines = buffer.split(/\r?\n/);
            buffer = lines.pop(); // 마지막 불완전한 줄은 버퍼에 남김
            
            for (let line of lines) {
              line = line.trim();
              if (line) {
                appendLog("RX: " + line);
                parseRfidBlock(line);
                const frame = parseFrame(line);
                if (frame) BranchProcess(frame);
              }
            }
          }
        }
      } catch (error) {
        appendLog("[ERROR] 읽기 오류: " + error);
      } finally {
        reader.releaseLock();
      }
    }

    async function serialWrite(data) {
      if (!writer) {
        alert("먼저 USB 장치를 연결해주세요.");
        return;
      }
      try {
        await writer.write(data + "\r\n");
        appendLog("TX: " + data);
      } catch (err) {
        appendLog("[ERROR] 쓰기 실패: " + err);
      }
    }

    // ==========================================
    // 2. 프로토콜 파싱 및 UI 바인딩 (절대 변경 안함)
    // ==========================================
    const SBA = Array(15).fill(false);
    const SWITCH_ID = {
      0: "touch_alram", 1: "touch_transmit", 2: "emerg_transmit", 3: "touch_sms", 4: "emerg_sms",
      5: "emerg_alram", 6: "touch_light", 7: "emerg_light", 8: "touch_mp3", 9: "emerg_mp3",
      10: "rf_alram", 11: "rf_transmit", 12: "rf_sms", 13: "rf_light", 14: "rf_mp3"
    };

    function requestInitialData() {
      serialWrite(`#0124defID9999R21:0000000000:*`);
      serialWrite(`#0119defID9999R1385:1:${document.getElementById("SpeakerVolume").value}:*`);
      serialWrite(`#0128defID9999R1480,1:91,1:104,1:*`);
      serialWrite(`#0135defID9999R1328:9:0,0,0,0,0,0,0,0,0:*`);
      serialWrite(`#0126defID9999R13154:4:0,0,0,0:*`);
    }

    function parseFrame(line) {
      const s = line.indexOf("#");
      const e = line.indexOf("*", s);
      if (s < 0 || e < 0 || line.startsWith("#OK")) return null;
      
      const frame = line.slice(s, e);
      const m = frame.slice(5).match(/[rRcC](?=\d)/);
      if (!m) return null;

      const rwIdx = 5 + m.index;
      const cmdStr = frame.substr(rwIdx + 1, 2);
      let payload = frame.slice(rwIdx + 4);
      if (payload.endsWith(":")) payload = payload.slice(0, -1);

      return { cmdStr, payload };
    }

    function BranchProcess(inFrame) {
      switch(Number(inFrame.cmdStr)) {
        case 13:
          let idx = inFrame.payload.indexOf(":");
          if (idx < 0) return;
          const subCmdStr = inFrame.payload.slice(0, idx);
          let Value = inFrame.payload.slice(idx + 4).split(",");
          
          if(subCmdStr == 85) {
            const SpkVal = inFrame.payload.slice(idx + 1).split(":")[1];
            document.getElementById("SpeakerVolume").value = SpkVal;
            document.getElementById("SpeakerVolumeTextBox").value = SpkVal;
          } else if(subCmdStr == 28) {
            document.getElementById("touch_transmit").checked = SBA[1] = !!Number(Value[0]);
            document.getElementById("emerg_transmit").checked = SBA[2] = !!Number(Value[1]);
            document.getElementById("touch_sms").checked = SBA[3] = !!Number(Value[2]);
            document.getElementById("emerg_sms").checked = SBA[4] = !!Number(Value[3]);
            document.getElementById("touch_light").checked = SBA[6] = !!Number(Value[5]);
            document.getElementById("emerg_light").checked = SBA[7] = !!Number(Value[6]);
            document.getElementById("touch_mp3").checked = SBA[8] = !!Number(Value[7]);
            document.getElementById("emerg_mp3").checked = SBA[9] = !!Number(Value[8]);
          } else if(subCmdStr == 54) {
            document.getElementById("rf_transmit").checked = SBA[11] = !!Number(Value[0]);
            document.getElementById("rf_sms").checked = SBA[12] = !!Number(Value[1]);
            document.getElementById("rf_light").checked = SBA[13] = !!Number(Value[2]);
            document.getElementById("rf_mp3").checked = SBA[14] = !!Number(Value[3]);
          }
          break;
        case 14:
          for (const m of inFrame.payload.matchAll(/([^:]+)(?::|$)/g)) {
            const [a, b] = m[1].split(",").map(Number);
            if(a == 80) document.getElementById("touch_alram").checked = SBA[0] = !!b;
            if(a == 91) document.getElementById("emerg_alram").checked = SBA[5] = !!b;
            if(a == 104) document.getElementById("rf_alram").checked = SBA[10] = !!b;
          }
          break;
        case 21:
          document.getElementById("PhoneNumber").value = inFrame.payload;
          break;
      }
    }

    // ==========================================
    // 3. UI 이벤트 리스너
    // ==========================================
    function switchTab(tabId, element) {
      document.querySelectorAll('.tab-content').forEach(el => el.classList.remove('active'));
      document.querySelectorAll('.tab').forEach(el => el.classList.remove('active'));
      document.getElementById(tabId).classList.add('active');
      element.classList.add('active');
    }

    // 전화번호 업데이트
    function updatePhoneNumber() {
      const val = document.getElementById("PhoneNumber").value;
      serialWrite(`#0121defID9999W21:${val}:*`);
    }

    // 슬라이더 동기화 및 전송
    const slider = document.getElementById("SpeakerVolume");
    const valBox = document.getElementById("SpeakerVolumeTextBox");
    slider.addEventListener("input", () => valBox.value = slider.value);
    slider.addEventListener("change", function() {
      serialWrite(`#0119defID9999W1385:1:${this.value}:*`);
    });

    // 스위치 토글 이벤트 (프로토콜 패킷 생성)
    document.querySelectorAll(".switch-input").forEach(input => {
      input.addEventListener("change", function() {
        const idx = Number(Object.keys(SWITCH_ID).find(k => SWITCH_ID[k] === this.id));
        SBA[idx] = this.checked;
        const n = SBA.map(v => v ? 1 : 0);
        serialWrite(`#0128defID9999W1480,${n[0]}:91,${n[5]}:104,${n[10]}:*`);
        serialWrite(`#0135defID9999W1328:9:${n[1]},${n[2]},${n[3]},${n[4]},1,${n[6]},${n[7]},${n[8]},${n[9]}:*`);
        serialWrite(`#0126defID9999W13154:4:${n[11]},${n[12]},${n[13]},${n[14]}:*`);
      });
    });

    // ==========================================
    // 4. RFID 스와이프 삭제 로직
    // ==========================================
    function loadRfidList() {
      document.getElementById("RfidList").innerHTML = "";
      serialWrite("#0121defID9999W13287:1:92:*");
    }

    function parseRfidBlock(s) {
      const p = s.indexOf("RFid:");
      if (p < 0) return;
      const a = p + 5;
      const c1 = s.indexOf(",", a);
      const c2 = s.indexOf(",", c1 + 1);
      if (c1 < 0 || c2 < 0) return;

      const idx = parseInt(s.substring(a, c1).trim(), 10);
      const device = s.substring(c1 + 1, c2).trim();
      if (!Number.isNaN(idx) && idx != 33 && device != "Z999") {
        addSwipeItem(device, idx);
      }
    }

    function addSwipeItem(text, value) {
      const li = document.createElement("li");
      li.className = "swipe-item";
      li.textContent = text;
      li.dataset.val = value;
      document.getElementById("RfidList").appendChild(li);

      let startX = 0, currentX = 0, dragging = false;
      const threshold = 80;

      const onStart = (e) => { dragging = true; startX = e.touches ? e.touches[0].clientX : e.clientX; li.classList.add("dragging"); };
      const onMove = (e) => {
        if (!dragging) return;
        currentX = (e.touches ? e.touches[0].clientX : e.clientX) - startX;
        if (currentX > 0) li.style.transform = `translateX(${currentX}px)`;
      };
      const onEnd = () => {
        if(!dragging) return;
        dragging = false; li.classList.remove("dragging");
        if (currentX > threshold) {
          li.classList.add("removing");
          setTimeout(() => li.remove(), 200);
          // 삭제 패킷 전송
          serialWrite(`#0121defID9999W25:${value}:Z999:*`);
        } else {
          li.style.transform = `translateX(0)`;
        }
        currentX = 0;
      };

      li.addEventListener("touchstart", onStart, { passive: true });
      li.addEventListener("touchmove", onMove, { passive: true });
      li.addEventListener("touchend", onEnd);
    }
  </script>
</body>
</html>
