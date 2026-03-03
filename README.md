<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>비상벨 직접 제어 패널</title>
    <style>
        :root {
            --primary: #007bff; --primary-dark: #0056b3;
            --bg-color: #f4f7f6; --card-bg: #ffffff;
            --text-main: #333333; --text-muted: #666666;
            --border-color: #e0e0e0; --danger: #dc3545;
            --success: #10b981; --warning: #f59e0b;
        }

        * { box-sizing: border-box; margin: 0; padding: 0; -webkit-tap-highlight-color: transparent; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background-color: var(--bg-color); color: var(--text-main); }

        .header {
            position: sticky; top: 0; z-index: 100; display: flex; justify-content: center; align-items: center;
            background: var(--primary); color: white; padding: 16px; box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        .header h2 { font-size: 18px; font-weight: 600; margin: 0; }

        #mobileWarningPage {
            display: none; flex-direction: column; justify-content: center; align-items: center;
            height: 100vh; text-align: center; background: white; padding: 20px;
        }
        .warning-icon { font-size: 60px; margin-bottom: 20px; }
        .warning-title { font-size: 22px; font-weight: bold; color: var(--danger); margin-bottom: 15px; }
        .warning-desc { font-size: 16px; color: var(--text-muted); line-height: 1.5; margin-bottom: 30px; }
        .download-btn {
            background: var(--primary); color: white; padding: 14px 24px; border-radius: 8px;
            text-decoration: none; font-size: 16px; font-weight: bold; box-shadow: 0 4px 6px rgba(0,123,255,0.3);
        }

        #mainControlPage { display: none; }
        .connection-bar { background: #fff; padding: 12px 16px; display: flex; gap: 10px; align-items: center; border-bottom: 1px solid var(--border-color); }
        .status-dot { width: 12px; height: 12px; border-radius: 50%; background: var(--danger); }
        .status-dot.connected { background: var(--success); }
        
        .btn-connect { padding: 10px 16px; border: none; border-radius: 6px; font-weight: bold; color: white; cursor: pointer; transition: 0.2s; }
        .btn-pc { background: #4b5563; flex: 1; }
        .btn-app { background: var(--primary); flex: 1; font-size: 16px; padding: 14px;}
        .btn-danger { background: var(--danger) !important; }

        .card { background: var(--card-bg); border-radius: 12px; padding: 16px; margin: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); }
        button.btn-action { width: 100%; padding: 14px; background: var(--primary); color: white; border: none; border-radius: 8px; font-size: 16px; font-weight: 600; margin-bottom: 8px; cursor: pointer; }
        #logBox { margin: 16px; height: 300px; background: #1e1e1e; color: #4af626; padding: 12px; font-family: monospace; font-size: 12px; overflow-y: auto; border-radius: 8px; word-break: break-all; }
    </style>
</head>
<body>

    <div id="mobileWarningPage">
        <div class="warning-icon">📱🚫</div>
        <div class="warning-title">전용 앱 사용 안내</div>
        <div class="warning-desc">
            일반 모바일 브라우저에서는<br>USB 기기 직접 제어를 지원하지 않습니다.<br><br>
            장치 통신을 위해 <strong>[전용 앱]</strong>을 이용해 주세요.
        </div>
        <a href="#" class="download-btn" onclick="alert('앱 다운로드 준비 중')">안드로이드 앱 다운로드</a>
    </div>

    <div id="mainControlPage">
        <div class="header"><h2>비상벨 제어 패널</h2></div>

        <!-- PC 연결 UI -->
        <div id="ui-pc-connection" class="connection-bar" style="display: none;">
            <div id="pcStatusDot" class="status-dot"></div>
            <span id="pcStatusText" style="font-weight: bold; font-size: 14px; width: 60px;">연결대기</span>
            <select id="baudRate" style="padding: 8px; border-radius: 6px; border: 1px solid #ccc;">
                <option value="115200" selected>115200 bps</option>
            </select>
            <button id="btnPcConnect" class="btn-connect btn-pc" onclick="SerialManager.connectPC()">USB 포트 열기</button>
        </div>

        <!-- 앱 연결 UI -->
        <div id="ui-app-connection" class="connection-bar" style="display: none; flex-direction: column;">
            <div style="display: flex; width: 100%; justify-content: space-between; align-items: center; margin-bottom: 8px;">
                <span style="font-size: 14px; color: var(--text-muted);">안드로이드 네이티브 연결</span>
                <div style="display: flex; align-items: center; gap: 6px;">
                    <div id="appStatusDot" class="status-dot"></div>
                    <span id="appStatusText" style="font-weight: bold; font-size: 14px;">검색중</span>
                </div>
            </div>
            <button id="btnAppConnect" class="btn-connect btn-app" onclick="SerialManager.connectApp()">USB 장치 연결 (앱)</button>
        </div>

        <div class="card">
            <button class="btn-action btn-danger" onclick="SerialManager.write('#0121defID9999W13287:1:97:*')">🚨 경보 발생</button>
            <button class="btn-action" style="background: transparent; border: 1px solid var(--primary); color: var(--primary);" onclick="SerialManager.write('#0120defID9999W13287:1:4:*')">✅ 경보 해제</button>
        </div>

        <div id="logBox">시스템 대기 중...</div>
    </div>

    <script>
        // UI 분기
        document.addEventListener("DOMContentLoaded", () => {
            const isAppEnvironment = typeof window.Android !== "undefined";
            const isMobileBrowser = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent);

            if (isAppEnvironment) {
                document.getElementById("mainControlPage").style.display = "block";
                document.getElementById("ui-app-connection").style.display = "flex";
            } else if (isMobileBrowser) {
                document.getElementById("mobileWarningPage").style.display = "flex";
            } else {
                document.getElementById("mainControlPage").style.display = "block";
                document.getElementById("ui-pc-connection").style.display = "flex";
                if (!navigator.serial) document.getElementById("btnPcConnect").disabled = true;
            }
        });

        // 로그 유틸리티
        const logBox = document.getElementById("logBox");
        function appendLog(msg) {
            logBox.textContent += msg + "\n";
            if (logBox.textContent.length > 10000) logBox.textContent = logBox.textContent.slice(-8000);
            logBox.scrollTop = logBox.scrollHeight;
        }

        // ==========================================
        // ★ 그새끼 프로토콜 파싱 로직 (VB.NET -> JS 이식) ★
        // ==========================================
        
        // 맵 주소별 기능 정의 (원래 process_map_data 부분)
        // 토큰 절약을 위해 핵심만 선언, 필요시 여기에 추가만 하면 됩니다.
        const MAP_ADDR = {
            1: "EB_TYPE",
            139: "MP3_VOLUME",
            140: "CODECCTRL_SPK_VOL", // 스피커 볼륨 (예: 1~5)
            141: "CODECCTRL_MIC_VOL", // 마이크 볼륨
            // ... (나머지 필요한 번지수를 추가하세요)
        };

        function parseLegacyProtocol(rxdata) {
            // 1. 디버그/AT 명령어 처리 (원래 코드의 AT, @, $$ 등 처리부)
            if (rxdata.startsWith("@") || rxdata.startsWith("AT") || rxdata.startsWith("$$")) {
                appendLog("[DEBUG/AT] " + rxdata);
                return;
            }

            // 2. ASN-500 메인 프로토콜 파싱 (parcing_ASN500 부분)
            // 형태: #01 24 defID9999 R 13 85:1:0:*
            const startIdx = rxdata.indexOf("#");
            const endIdx = rxdata.indexOf("*");
            
            if (startIdx < 0 || endIdx < 0) return; // 불완전한 패킷 무시
            
            let frame = rxdata.substring(startIdx, endIdx + 1);
            if (frame.startsWith("#OK*") || frame.length < 15) return;

            try {
                // 문자열 절단 (VB.NET의 Substring 기반 이식)
                const trId = frame.substring(1, 3);
                const lenStr = frame.substring(3, 5);
                const deviceId = frame.substring(5, 14);
                const rw = frame.substring(14, 15).toUpperCase(); // R, W, C
                const code = frame.substring(15, 17);
                const payload = frame.substring(17, frame.length - 1); // '*' 제외

                appendLog(`[PARSED] Cmd:${code} RW:${rw} Dev:${deviceId} Payload:${payload}`);

                // Command Code 분기 (VB의 Select Case CODE)
                switch (code) {
                    case "13": // 맵데이터 연속 쓰기/읽기
                        if (payload.includes(":")) {
                            const parts = payload.split(":");
                            if (parts.length >= 3) {
                                const startMap = parseInt(parts[0], 10);
                                const count = parseInt(parts[1], 10);
                                const vals = parts[2].split(",");
                                
                                for (let i = 0; i < vals.length; i++) {
                                    if(vals[i] !== "") processMapData(startMap + i, vals[i]);
                                }
                            }
                        }
                        break;

                    case "14": // 맵데이터 개별 쓰기/읽기
                        if (payload.includes(":")) {
                            const pairs = payload.split(":");
                            pairs.forEach(pair => {
                                if (pair.includes(",")) {
                                    const [mapAddr, mapVal] = pair.split(",");
                                    processMapData(parseInt(mapAddr, 10), mapVal);
                                }
                            });
                        }
                        break;

                    case "21": // 관제번호
                        const callNum = payload.replace(/:/g, "");
                        appendLog(`[UPDATE] 관제번호 변경됨: ${callNum}`);
                        break;
                    
                    case "30": // 로그인 응답
                        appendLog(`[UPDATE] 로그인(CM) 상태 응답: ${payload}`);
                        break;

                    default:
                        // 기타 코드 처리
                        break;
                }
            } catch (e) {
                appendLog("[ERROR] 프로토콜 파싱 중 오류: " + e.message);
            }
        }

        // 맵 데이터 매핑 처리부 (VB의 process_map_data)
        function processMapData(address, value) {
            const funcName = MAP_ADDR[address] || `ADDR_${address}`;
            appendLog(`  -> [MAP] ${funcName} 설정됨 (값: ${value})`);
            
            // 향후 이 부분에 document.getElementById('...').value = value; 형태로 UI 바인딩 추가
        }


        // ==========================================
        // 3. 통합 통신 핸들러 (어댑터)
        // ==========================================
        const SerialManager = {
            port: null, writer: null, reader: null, keepReading: false,
            // [수정] VB 코드는 '\n' 또는 '*' 로 패킷을 자름. 버퍼링 방식 개선.
            rxBuffer: "", 

            async connectPC() {
                if (this.port) { await this.disconnectPC(); return; }
                try {
                    const baudRate = parseInt(document.getElementById("baudRate").value);
                    this.port = await navigator.serial.requestPort();
                    await this.port.open({ baudRate: baudRate });
                    
                    const textEncoder = new TextEncoderStream();
                    textEncoder.readable.pipeTo(this.port.writable);
                    this.writer = textEncoder.writable.getWriter();
                    
                    document.getElementById("pcStatusDot").classList.add("connected");
                    document.getElementById("pcStatusText").innerText = "연결됨";
                    document.getElementById("pcStatusText").style.color = "var(--success)";
                    const btn = document.getElementById("btnPcConnect");
                    btn.innerText = "연결 해제";
                    btn.classList.add("btn-danger");

                    appendLog(`[SYSTEM] 포트 열림 (${baudRate}bps)`);
                    
                    this.keepReading = true;
                    this.readLoopPC();
                } catch (err) {
                    appendLog("[ERROR] PC 연결 실패: " + err.message);
                }
            },

            async disconnectPC() {
                this.keepReading = false;
                if (this.reader) { await this.reader.cancel(); this.reader = null; }
                if (this.writer) { await this.writer.close(); this.writer = null; }
                if (this.port) { await this.port.close(); this.port = null; }

                document.getElementById("pcStatusDot").classList.remove("connected");
                document.getElementById("pcStatusText").innerText = "연결대기";
                document.getElementById("pcStatusText").style.color = "var(--text-main)";
                const btn = document.getElementById("btnPcConnect");
                btn.innerText = "USB 포트 열기";
                btn.classList.remove("btn-danger");
                appendLog("[SYSTEM] PC 포트 닫힘");
            },

            connectApp() {
                window.Android.connectUSB(); 
                appendLog("[SYSTEM] 네이티브 앱에 USB 연결을 요청했습니다.");
                document.getElementById("appStatusDot").classList.add("connected");
                document.getElementById("appStatusText").innerText = "권한요청중";
                document.getElementById("btnAppConnect").innerText = "연결 해제 (앱)";
                document.getElementById("btnAppConnect").classList.add("btn-danger");
            },

            async write(data) {
                // 원래 프로토콜 규칙대로 \r\n을 포함하여 쏜다
                if (typeof window.Android !== "undefined") {
                    window.Android.sendData(data);
                    appendLog("TX (App): " + data);
                } else if (this.writer) {
                    await this.writer.write(data + "\r\n");
                    appendLog("TX (PC): " + data);
                } else {
                    alert("먼저 장치를 연결해주세요.");
                }
            },

            onReceiveData(rawData) {
                // 스트림으로 들어온 데이터를 버퍼에 누적하고, '*' 또는 줄바꿈 단위로 잘라서 파싱
                this.rxBuffer += rawData;
                
                // 버퍼에 * 또는 줄바꿈이 포함되어 있으면 잘라냄
                while (this.rxBuffer.includes("*") || this.rxBuffer.includes("\n")) {
                    let splitIdx = this.rxBuffer.indexOf("*");
                    let lfIdx = this.rxBuffer.indexOf("\n");
                    
                    // 둘 다 존재하면 더 앞에 있는 것을 기준으로 자름
                    let cutIdx = -1;
                    if (splitIdx !== -1 && lfIdx !== -1) cutIdx = Math.min(splitIdx, lfIdx);
                    else if (splitIdx !== -1) cutIdx = splitIdx;
                    else cutIdx = lfIdx;

                    let completeFrame = this.rxBuffer.substring(0, cutIdx + 1).trim();
                    this.rxBuffer = this.rxBuffer.substring(cutIdx + 1);

                    if (completeFrame.length > 0) {
                        appendLog("RX: " + completeFrame);
                        parseLegacyProtocol(completeFrame); // 핵심 파서로 넘김
                    }
                }
            },

            async readLoopPC() {
                const textDecoder = new TextDecoderStream();
                const readableStreamClosed = this.port.readable.pipeTo(textDecoder.writable);
                this.reader = textDecoder.readable.getReader();
                try {
                    while (this.keepReading) {
                        const { value, done } = await this.reader.read();
                        if (done) break;
                        if (value) this.onReceiveData(value);
                    }
                } catch (error) {
                    appendLog("[ERROR] PC 읽기 오류: " + error);
                } finally {
                    this.reader.releaseLock();
                }
            }
        };

        // 안드로이드 통신 브릿지
        function receiveDataFromAndroid(data) {
            SerialManager.onReceiveData(data);
        }
    </script>
</body>
</html>
