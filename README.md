<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>비상벨 통합 제어 패널</title>
    <style>
        :root {
            --primary: #2563eb; --primary-hover: #1d4ed8;
            --bg-color: #f3f4f6; --card-bg: #ffffff;
            --text-main: #1f2937; --text-muted: #6b7280;
            --border-color: #e5e7eb; --danger: #ef4444; --danger-hover: #dc2626;
            --success: #10b981; --warning: #f59e0b;
        }

        * { box-sizing: border-box; margin: 0; padding: 0; -webkit-tap-highlight-color: transparent; }
        body { font-family: 'Pretendard', -apple-system, BlinkMacSystemFont, system-ui, Roboto, sans-serif; background-color: var(--bg-color); color: var(--text-main); }

        /* 공통 헤더 */
        .header {
            position: sticky; top: 0; z-index: 100; display: flex; justify-content: space-between; align-items: center;
            background: var(--primary); color: white; padding: 16px 20px; box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        .header h2 { font-size: 18px; font-weight: 700; margin: 0; letter-spacing: -0.5px; }

        /* 모바일 차단 페이지 */
        #mobileWarningPage { display: none; flex-direction: column; justify-content: center; align-items: center; height: 100vh; text-align: center; background: white; padding: 20px; }
        .warning-icon { font-size: 64px; margin-bottom: 20px; }
        .warning-title { font-size: 24px; font-weight: 800; color: var(--danger); margin-bottom: 15px; }
        .warning-desc { font-size: 16px; color: var(--text-muted); line-height: 1.6; margin-bottom: 30px; word-break: keep-all; }
        
        /* 메인 페이지 */
        #mainControlPage { display: none; padding-bottom: 80px; }

        /* 연결 상태 바 */
        .connection-bar { background: #fff; padding: 12px 20px; display: flex; gap: 10px; align-items: center; border-bottom: 1px solid var(--border-color); }
        .status-indicator { display: flex; align-items: center; gap: 8px; flex: 1; }
        .status-dot { width: 12px; height: 12px; border-radius: 50%; background: var(--danger); box-shadow: 0 0 0 3px rgba(239,68,68,0.2); transition: 0.3s; }
        .status-dot.connected { background: var(--success); box-shadow: 0 0 0 3px rgba(16,185,129,0.2); }
        
        button { cursor: pointer; font-family: inherit; transition: all 0.2s ease; }
        .btn { padding: 12px 16px; border: none; border-radius: 8px; font-weight: 600; font-size: 15px; color: white; }
        .btn-connect { background: #4b5563; }
        .btn-connect.connected { background: var(--danger); }
        .btn-primary { background: var(--primary); }
        .btn-primary:active { background: var(--primary-hover); }
        .btn-danger { background: var(--danger); }
        .btn-danger:active { background: var(--danger-hover); }
        .btn-outline { background: transparent; border: 1px solid var(--primary); color: var(--primary); }
        .btn-outline:active { background: #eff6ff; }

        /* 탭 메뉴 */
        .tabs { display: flex; background: #fff; border-bottom: 1px solid var(--border-color); overflow-x: auto; }
        .tab { flex: 1; text-align: center; padding: 14px; font-weight: 600; color: var(--text-muted); border-bottom: 3px solid transparent; cursor: pointer; white-space: nowrap; }
        .tab.active { color: var(--primary); border-bottom-color: var(--primary); }
        .tab-content { display: none; padding: 16px; }
        .tab-content.active { display: block; animation: fadeIn 0.3s; }

        @keyframes fadeIn { from { opacity: 0; transform: translateY(5px); } to { opacity: 1; transform: translateY(0); } }

        /* 카드 UI */
        .card { background: var(--card-bg); border-radius: 12px; padding: 20px; margin-bottom: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.04); border: 1px solid var(--border-color); }
        .card-title { font-size: 16px; font-weight: 700; margin-bottom: 16px; color: var(--text-main); display: flex; align-items: center; gap: 8px; }
        
        /* 그리드 레이아웃 */
        .grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
        .grid-4 { display: grid; grid-template-columns: repeat(auto-fit, minmax(140px, 1fr)); gap: 12px; }
        
        /* 상태 위젯 */
        .status-widget { background: var(--bg-color); padding: 12px; border-radius: 8px; text-align: center; border: 1px solid var(--border-color); }
        .status-widget .label { font-size: 12px; color: var(--text-muted); margin-bottom: 4px; display: block; }
        .status-widget .value { font-size: 18px; font-weight: 700; color: var(--primary); }

        /* 입력 폼 */
        .form-group { margin-bottom: 12px; }
        .form-group label { display: block; font-size: 13px; color: var(--text-muted); margin-bottom: 6px; font-weight: 500; }
        .input-row { display: flex; gap: 8px; }
        input[type="text"], input[type="number"], select { flex: 1; padding: 12px; border: 1px solid #d1d5db; border-radius: 8px; font-size: 15px; outline: none; transition: border 0.2s; }
        input:focus, select:focus { border-color: var(--primary); }

        /* 로그 박스 */
        #logBox { height: 300px; background: #1e293b; color: #a7f3d0; padding: 12px; font-family: 'Courier New', monospace; font-size: 13px; overflow-y: auto; border-radius: 8px; line-height: 1.4; }
        .log-tx { color: #93c5fd; }
        .log-rx { color: #fde047; }
        .log-sys { color: #cbd5e1; }
    </style>
</head>
<body>

    <!-- [모바일 브라우저 차단 화면] -->
    <div id="mobileWarningPage">
        <div class="warning-icon">📱🚫</div>
        <div class="warning-title">전용 앱 사용 안내</div>
        <div class="warning-desc">
            일반 모바일 브라우저에서는 USB 기기 직접 제어 기능을 지원하지 않습니다.<br><br>
            장치 통신을 위해 <strong>[비상벨 제어 전용 앱]</strong>을 이용해 주시기 바랍니다.
        </div>
        <button class="btn btn-primary" style="width: 100%; max-width: 300px;" onclick="alert('앱 다운로드 링크 연결 예정')">안드로이드 앱 다운로드</button>
    </div>

    <!-- [메인 제어 화면] -->
    <div id="mainControlPage">
        <div class="header">
            <h2>🚨 비상벨 통합 제어 패널</h2>
            <span id="deviceIdDisplay" style="font-size: 13px; opacity: 0.8;">ID: defID9999</span>
        </div>

        <!-- 연결 UI -->
        <div id="ui-pc-connection" class="connection-bar" style="display: none;">
            <div class="status-indicator">
                <div id="pcStatusDot" class="status-dot"></div>
                <span id="pcStatusText" style="font-weight: 700; font-size: 14px;">연결대기</span>
            </div>
            <select id="baudRate" style="width: auto; padding: 8px;">
                <option value="9600">9600</option>
                <option value="115200" selected>115200</option>
            </select>
            <button id="btnPcConnect" class="btn btn-connect" onclick="SerialManager.connectPC()">USB 연결</button>
        </div>

        <div id="ui-app-connection" class="connection-bar" style="display: none; flex-direction: column; align-items: stretch;">
            <div class="status-indicator" style="margin-bottom: 8px; justify-content: space-between;">
                <span style="font-size: 13px; color: var(--text-muted);">안드로이드 네이티브 통신</span>
                <div style="display: flex; align-items: center; gap: 6px;">
                    <div id="appStatusDot" class="status-dot"></div>
                    <span id="appStatusText" style="font-weight: 700; font-size: 14px;">장치 검색중</span>
                </div>
            </div>
            <button id="btnAppConnect" class="btn btn-primary" onclick="SerialManager.connectApp()">비상벨 장치 연결</button>
        </div>

        <!-- 탭 메뉴 -->
        <div class="tabs">
            <div class="tab active" onclick="switchTab('tab-dashboard')">대시보드</div>
            <div class="tab" onclick="switchTab('tab-settings')">상세설정</div>
            <div class="tab" onclick="switchTab('tab-terminal')">터미널/로그</div>
        </div>

        <!-- 탭 1: 대시보드 -->
        <div id="tab-dashboard" class="tab-content active">
            
            <!-- 퀵 컨트롤 -->
            <div class="card">
                <div class="card-title">⚡ 빠른 제어 (명령 287)</div>
                <div class="grid-2">
                    <button class="btn btn-danger" onclick="DeviceControl.triggerAlarm()">🚨 경보 발생</button>
                    <button class="btn btn-outline" onclick="DeviceControl.clearAlarm()">✅ 경보 해제</button>
                    <button class="btn btn-primary" onclick="DeviceControl.resetDevice()">🔄 장치 리셋</button>
                    <button class="btn btn-primary" onclick="DeviceControl.resetModem()">📶 모뎀 리셋</button>
                </div>
            </div>

            <!-- 상태 모니터링 -->
            <div class="card">
                <div class="card-title">📊 장치 상태 모니터링</div>
                <div class="grid-4">
                    <div class="status-widget">
                        <span class="label">LTE 신호 (RSRP)</span>
                        <span class="value" id="val-rsrp">--</span>
                    </div>
                    <div class="status-widget">
                        <span class="label">LTE 신호 (RSSI)</span>
                        <span class="value" id="val-rssi">--</span>
                    </div>
                    <div class="status-widget">
                        <span class="label">배터리 전압</span>
                        <span class="value" id="val-battery">-- V</span>
                    </div>
                    <div class="status-widget">
                        <span class="label">USIM 상태</span>
                        <span class="value" id="val-sim">--</span>
                    </div>
                </div>
                <button class="btn btn-outline" style="width: 100%; margin-top: 12px;" onclick="DeviceControl.requestStatus()">상태 새로고침</button>
            </div>
        </div>

        <!-- 탭 2: 상세설정 -->
        <div id="tab-settings" class="tab-content">
            <div class="card">
                <div class="card-title">🔊 볼륨 설정</div>
                <div class="form-group">
                    <label>스피커 볼륨 (Map 85, 0~7)</label>
                    <div class="input-row">
                        <input type="number" id="inp-spk-vol" min="0" max="7" value="5">
                        <button class="btn btn-primary" onclick="DeviceControl.setVolume('spk')">적용</button>
                    </div>
                </div>
                <div class="form-group">
                    <label>마이크 볼륨 (Map 88, 0~63)</label>
                    <div class="input-row">
                        <input type="number" id="inp-mic-vol" min="0" max="63" value="40">
                        <button class="btn btn-primary" onclick="DeviceControl.setVolume('mic')">적용</button>
                    </div>
                </div>
            </div>

            <div class="card">
                <div class="card-title">📞 관제 번호 설정</div>
                <div class="form-group">
                    <label>메인 관제 번호 (Map 21)</label>
                    <div class="input-row">
                        <input type="text" id="inp-call-num" placeholder="01012345678">
                        <button class="btn btn-primary" onclick="DeviceControl.setCallNumber()">적용</button>
                    </div>
                </div>
            </div>

            <div class="card">
                <div class="card-title">🌐 서버 IP 설정</div>
                <div class="form-group">
                    <label>메인 서버 (Map 200)</label>
                    <div class="input-row">
                        <input type="text" id="inp-server-ip" placeholder="192.168.0.1:12100">
                        <button class="btn btn-primary" onclick="DeviceControl.setServerIP()">적용</button>
                    </div>
                </div>
            </div>
        </div>

        <!-- 탭 3: 터미널 -->
        <div id="tab-terminal" class="tab-content">
            <div class="card" style="padding: 10px;">
                <div class="card-title" style="margin-bottom: 8px; padding: 0 6px;">💻 통신 로그</div>
                <div id="logBox"></div>
                <div class="input-row" style="margin-top: 10px;">
                    <input type="text" id="inp-raw-cmd" placeholder="직접 패킷 입력 (#01...)">
                    <button class="btn btn-primary" onclick="DeviceControl.sendRaw()">전송</button>
                    <button class="btn btn-outline" onclick="document.getElementById('logBox').innerHTML=''">지우기</button>
                </div>
            </div>
        </div>
    </div>

    <script>
        // ==========================================
        // 1. UI 및 탭 제어
        // ==========================================
        function switchTab(tabId) {
            document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
            document.querySelectorAll('.tab-content').forEach(c => c.classList.remove('active'));
            event.currentTarget.classList.add('active');
            document.getElementById(tabId).classList.add('active');
        }

        const logBox = document.getElementById("logBox");
        function appendLog(msg, type = 'sys') {
            const time = new Date().toLocaleTimeString('ko-KR', { hour12: false });
            const span = document.createElement('span');
            span.className = `log-${type}`;
            span.innerText = `[${time}] ${msg}\n`;
            logBox.appendChild(span);
            logBox.scrollTop = logBox.scrollHeight;
        }

        // ==========================================
        // 2. 환경 감지 (PC Web Serial vs Android App)
        // ==========================================
        document.addEventListener("DOMContentLoaded", () => {
            const isAppEnvironment = typeof window.Android !== "undefined";
            const isMobileBrowser = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent);

            if (isAppEnvironment) {
                document.getElementById("mainControlPage").style.display = "block";
                document.getElementById("ui-app-connection").style.display = "flex";
                appendLog("안드로이드 전용 앱 모드로 초기화되었습니다.", "sys");
            } else if (isMobileBrowser) {
                document.getElementById("mobileWarningPage").style.display = "flex";
            } else {
                document.getElementById("mainControlPage").style.display = "block";
                document.getElementById("ui-pc-connection").style.display = "flex";
                if (!navigator.serial) {
                    appendLog("이 브라우저는 Web Serial API를 지원하지 않습니다. (Chrome/Edge 사용 권장)", "sys");
                    document.getElementById("btnPcConnect").disabled = true;
                } else {
                    appendLog("PC Web Serial 통신 모드로 초기화되었습니다.", "sys");
                }
            }
        });

        // ==========================================
        // 3. 통신 매니저 (SerialManager)
        // ==========================================
        const SerialManager = {
            port: null, writer: null, reader: null, keepReading: false, isConnected: false,

            async connectPC() {
                if (this.isConnected) { await this.disconnectPC(); return; }
                try {
                    const baudRate = parseInt(document.getElementById("baudRate").value);
                    this.port = await navigator.serial.requestPort();
                    await this.port.open({ baudRate: baudRate });
                    
                    const textEncoder = new TextEncoderStream();
                    textEncoder.readable.pipeTo(this.port.writable);
                    this.writer = textEncoder.writable.getWriter();
                    
                    this.setUIConnected(true, "PC");
                    appendLog(`포트 열림 (${baudRate}bps)`, "sys");
                    
                    this.keepReading = true;
                    this.readLoopPC();
                } catch (err) {
                    appendLog(`PC 연결 실패: ${err.message}`, "sys");
                }
            },

            async disconnectPC() {
                this.keepReading = false;
                if (this.reader) { await this.reader.cancel(); this.reader = null; }
                if (this.writer) { await this.writer.close(); this.writer = null; }
                if (this.port) { await this.port.close(); this.port = null; }
                this.setUIConnected(false, "PC");
                appendLog("PC 포트 닫힘", "sys");
            },

            connectApp() {
                if(this.isConnected) {
                    window.Android.disconnectUSB(); // 가상의 앱 브릿지 함수
                    this.setUIConnected(false, "App");
                } else {
                    window.Android.connectUSB(); 
                    appendLog("앱에 USB 연결을 요청했습니다.", "sys");
                    // 실제로는 앱에서 콜백을 주어야 함. 여기서는 즉시 연결된 것으로 UI 처리
                    this.setUIConnected(true, "App");
                }
            },

            setUIConnected(connected, type) {
                this.isConnected = connected;
                if(type === "PC") {
                    document.getElementById("pcStatusDot").className = `status-dot ${connected ? 'connected' : ''}`;
                    document.getElementById("pcStatusText").innerText = connected ? "연결됨" : "연결대기";
                    document.getElementById("pcStatusText").style.color = connected ? "var(--success)" : "var(--text-main)";
                    const btn = document.getElementById("btnPcConnect");
                    btn.innerText = connected ? "연결 해제" : "USB 연결";
                    btn.className = `btn ${connected ? 'btn-danger' : 'btn-connect'}`;
                } else {
                    document.getElementById("appStatusDot").className = `status-dot ${connected ? 'connected' : ''}`;
                    document.getElementById("appStatusText").innerText = connected ? "장치 연결됨" : "장치 검색중";
                    document.getElementById("appStatusText").style.color = connected ? "var(--success)" : "var(--text-main)";
                    const btn = document.getElementById("btnAppConnect");
                    btn.innerText = connected ? "연결 해제 (앱)" : "비상벨 장치 연결";
                    btn.className = `btn ${connected ? 'btn-danger' : 'btn-primary'}`;
                }
            },

            async write(data) {
                if (!this.isConnected) { alert("먼저 장치를 연결해주세요."); return; }
                
                appendLog(data, "tx");
                
                if (typeof window.Android !== "undefined") {
                    window.Android.sendData(data + "\r\n");
                } else if (this.writer) {
                    await this.writer.write(data + "\r\n");
                }
            },

            onReceiveData(line) {
                appendLog(line, "rx");
                DeviceParser.parse(line);
            },

            async readLoopPC() {
                const textDecoder = new TextDecoderStream();
                const readableStreamClosed = this.port.readable.pipeTo(textDecoder.writable);
                this.reader = textDecoder.readable.getReader();
                let buffer = "";
                try {
                    while (this.keepReading) {
                        const { value, done } = await this.reader.read();
                        if (done) break;
                        if (value) {
                            buffer += value;
                            let lines = buffer.split(/\r?\n/);
                            buffer = lines.pop(); 
                            for (let line of lines) {
                                if (line.trim()) this.onReceiveData(line.trim());
                            }
                        }
                    }
                } catch (error) {
                    appendLog(`PC 읽기 오류: ${error}`, "sys");
                } finally {
                    this.reader.releaseLock();
                }
            }
        };

        // 앱에서 호출할 전역 수신 함수
        window.receiveDataFromAndroid = function(data) {
            SerialManager.onReceiveData(data);
        };

        // ==========================================
        // 4. 비즈니스 로직 (프로토콜 생성 및 파싱)
        // ==========================================
        const targetID = "defID9999"; // 기본 브로드캐스트 ID

        const DeviceControl = {
            // 패킷 생성 코어 함수 (VB.NET make_packet 완벽 이식)
            // 형식: #01[길이][ID][R/W][CODE][DATA]*
            makePacket(rw, code, dataStr) {
                const trId = "01";
                const payload = targetID.padStart(9, '0') + rw + code + dataStr;
                const lengthStr = payload.length.toString().padStart(2, '0');
                return `#${trId}${lengthStr}${payload}*`;
            },

            // Map 데이터 쓰기 (Code 13: 연속 쓰기)
            writeMap(address, values) {
                // 형식: 주소:갯수:데이터1,데이터2:
                const dataStr = `${address}:${values.length}:${values.join(',')}:`;
                const packet = this.makePacket("W", "13", dataStr);
                SerialManager.write(packet);
            },

            // Map 데이터 읽기 (Code 13 Read)
            readMap(address, count) {
                const dataStr = `${address}:${count}::`;
                const packet = this.makePacket("R", "13", dataStr);
                SerialManager.write(packet);
            },

            // --- 개별 기능 제어 ---
            triggerAlarm() { this.writeMap(287, [97]); }, // 97: 경보 발생
            clearAlarm() { this.writeMap(287, [4]); },    // 4: 경보 해제
            resetDevice() { 
                if(confirm("장치를 재시작하시겠습니까?")) this.writeMap(287, [1]); 
            },
            resetModem() { 
                if(confirm("LTE 모뎀을 재시작하시겠습니까?")) this.writeMap(287, [2]); 
            },
            
            setVolume(type) {
                if(type === 'spk') {
                    const vol = document.getElementById('inp-spk-vol').value;
                    this.writeMap(85, [vol]); // CODECCTRL_SPK_VOL
                } else {
                    const vol = document.getElementById('inp-mic-vol').value;
                    this.writeMap(88, [vol]); // CODECCTRL_MIC_VOL
                }
                alert("볼륨 설정 명령을 전송했습니다.");
            },

            setCallNumber() {
                const num = document.getElementById('inp-call-num').value.replace(/-/g, "");
                if(!num) return alert("번호를 입력하세요.");
                // Code 21: 관제번호 설정
                const packet = this.makePacket("W", "21", `:${num}:`);
                SerialManager.write(packet);
                alert("관제번호 설정 명령을 전송했습니다.");
            },

            setServerIP() {
                const ipPort = document.getElementById('inp-server-ip').value.split(":");
                if(ipPort.length !== 2) return alert("IP:PORT 형식으로 입력하세요.");
                const ips = ipPort[0].split(".");
                const port = parseInt(ipPort[1]);
                const portH = Math.floor(port / 256);
                const portL = port % 256;
                // Map 200부터 6개 바이트 (IP1, IP2, IP3, IP4, PortH, PortL)
                this.writeMap(200, [ips[0], ips[1], ips[2], ips[3], portH, portL]);
                alert("서버 IP 설정 명령을 전송했습니다.");
            },

            requestStatus() {
                // 상태 갱신을 위해 AT 커맨드나 특정 맵 읽기 요청
                SerialManager.write("AT#KTDEVSTAT\r\n");
            },

            sendRaw() {
                const cmd = document.getElementById('inp-raw-cmd').value;
                if(cmd) SerialManager.write(cmd);
            }
        };

        // 데이터 파서 (수신 데이터 UI 업데이트)
        const DeviceParser = {
            parse(line) {
                // 디버그 메시지 파싱 (#RSRP=xx* 형태)
                if (line.includes("#RSRP=")) this.updateVal("val-rsrp", this.extract(line, "#RSRP="));
                if (line.includes("#RSSI=")) this.updateVal("val-rssi", this.extract(line, "#RSSI="));
                if (line.includes("#BAT_AVR_FLOAT=")) this.updateVal("val-battery", parseFloat(this.extract(line, "#BAT_AVR_FLOAT=")).toFixed(2) + " V");
                if (line.includes("#SIM=")) {
                    const sim = this.extract(line, "#SIM=");
                    this.updateVal("val-sim", sim === "1" ? "정상" : (sim === "0" ? "확인중" : "에러"));
                }
                if (line.includes("#NetState=")) {
                    const net = this.extract(line, "#NetState=");
                    // UI에 NetState 표시 공간이 있다면 업데이트
                }
            },
            extract(line, key) {
                try {
                    const start = line.indexOf(key) + key.length;
                    const end = line.indexOf("*", start);
                    return end !== -1 ? line.substring(start, end).trim() : line.substring(start).trim();
                } catch(e) { return "--"; }
            },
            updateVal(id, val) {
                const el = document.getElementById(id);
                if(el) {
                    el.innerText = val;
                    // 값 변경 시 깜빡임 효과 (Blink 로직 포팅)
                    el.style.color = "var(--primary)";
                    setTimeout(() => el.style.color = "var(--text-main)", 500);
                }
            }
        };
    </script>
</body>
</html>
