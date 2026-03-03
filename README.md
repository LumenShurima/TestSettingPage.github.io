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

        /* 공통 헤더 */
        .header {
            position: sticky; top: 0; z-index: 100; display: flex; justify-content: center; align-items: center;
            background: var(--primary); color: white; padding: 16px; box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        .header h2 { font-size: 18px; font-weight: 600; margin: 0; }

        /* =========================================
           1. 일반 모바일 브라우저 차단 (안내 페이지)
           ========================================= */
        #mobileWarningPage {
            display: none; /* 기본 숨김 */
            flex-direction: column; justify-content: center; align-items: center;
            height: 100vh; text-align: center; background: white; padding: 20px;
        }
        .warning-icon { font-size: 60px; margin-bottom: 20px; }
        .warning-title { font-size: 22px; font-weight: bold; color: var(--danger); margin-bottom: 15px; }
        .warning-desc { font-size: 16px; color: var(--text-muted); line-height: 1.5; margin-bottom: 30px; }
        .download-btn {
            background: var(--primary); color: white; padding: 14px 24px; border-radius: 8px;
            text-decoration: none; font-size: 16px; font-weight: bold; box-shadow: 0 4px 6px rgba(0,123,255,0.3);
        }

        /* =========================================
           2. 메인 제어 페이지 (PC & 앱 공통)
           ========================================= */
        #mainControlPage { display: none; /* 기본 숨김 */ }

        /* 연결 상태 바 (UI 분기용) */
        .connection-bar { background: #fff; padding: 12px 16px; display: flex; gap: 10px; align-items: center; border-bottom: 1px solid var(--border-color); }
        .status-dot { width: 12px; height: 12px; border-radius: 50%; background: var(--danger); }
        .status-dot.connected { background: var(--success); }
        
        .btn-connect { padding: 10px 16px; border: none; border-radius: 6px; font-weight: bold; color: white; cursor: pointer; transition: 0.2s; }
        .btn-pc { background: #4b5563; flex: 1; }
        .btn-app { background: var(--primary); flex: 1; font-size: 16px; padding: 14px;}
        .btn-danger { background: var(--danger) !important; }

        /* 공통 컨트롤 UI (카드, 버튼 등) */
        .card { background: var(--card-bg); border-radius: 12px; padding: 16px; margin: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.05); }
        button.btn-action { width: 100%; padding: 14px; background: var(--primary); color: white; border: none; border-radius: 8px; font-size: 16px; font-weight: 600; margin-bottom: 8px; cursor: pointer; }
        #logBox { margin: 16px; height: 200px; background: #1e1e1e; color: #4af626; padding: 12px; font-family: monospace; font-size: 12px; overflow-y: auto; border-radius: 8px; }
    </style>
</head>
<body>

    <!-- [화면 A] 일반 모바일 브라우저 접속 시 띄울 경고 화면 -->
    <div id="mobileWarningPage">
        <div class="warning-icon">📱🚫</div>
        <div class="warning-title">전용 앱 사용 안내</div>
        <div class="warning-desc">
            일반 모바일 브라우저(크롬, 삼성인터넷 등)에서는<br>USB 기기 직접 제어 기능을 지원하지 않습니다.<br><br>
            장치 통신을 위해 <strong>[비상벨 제어 전용 앱]</strong>을<br>이용해 주시기 바랍니다.
        </div>
        <a href="#" class="download-btn" onclick="alert('앱 다운로드 링크 연결 예정')">안드로이드 앱 다운로드</a>
    </div>

    <!-- [화면 B] PC 또는 전용 앱 접속 시 띄울 메인 화면 -->
    <div id="mainControlPage">
        <div class="header"><h2>비상벨 제어 패널</h2></div>

        <!-- PC 전용 연결 UI -->
        <div id="ui-pc-connection" class="connection-bar" style="display: none;">
            <div id="pcStatusDot" class="status-dot"></div>
            <span id="pcStatusText" style="font-weight: bold; font-size: 14px; width: 60px;">연결대기</span>
            <select id="baudRate" style="padding: 8px; border-radius: 6px; border: 1px solid #ccc;">
                <option value="9600">9600 bps</option>
                <option value="115200" selected>115200 bps</option>
            </select>
            <button id="btnPcConnect" class="btn-connect btn-pc" onclick="SerialManager.connectPC()">USB 포트 열기</button>
        </div>

        <!-- 전용 앱 전용 연결 UI -->
        <div id="ui-app-connection" class="connection-bar" style="display: none; flex-direction: column;">
            <div style="display: flex; width: 100%; justify-content: space-between; align-items: center; margin-bottom: 8px;">
                <span style="font-size: 14px; color: var(--text-muted);">안드로이드 네이티브 통신 모드</span>
                <div style="display: flex; align-items: center; gap: 6px;">
                    <div id="appStatusDot" class="status-dot"></div>
                    <span id="appStatusText" style="font-weight: bold; font-size: 14px;">장치 검색중</span>
                </div>
            </div>
            <button id="btnAppConnect" class="btn-connect btn-app" onclick="SerialManager.connectApp()">비상벨 장치 연결 (앱)</button>
        </div>

        <!-- 공통 장치 제어 UI -->
        <div class="card">
            <button class="btn-action btn-danger" onclick="SerialManager.write('#0121defID9999W13287:1:97:*')">🚨 경보 발생</button>
            <button class="btn-action" style="background: transparent; border: 1px solid var(--primary); color: var(--primary);" onclick="SerialManager.write('#0120defID9999W13287:1:4:*')">✅ 경보 해제</button>
        </div>

        <!-- 로그 화면 -->
        <div id="logBox">시스템 대기 중...</div>
    </div>

    <script>
        // ==========================================
        // 1. 환경 감지 및 UI 분기 로직 (페이지 로드 시 실행)
        // ==========================================
        document.addEventListener("DOMContentLoaded", () => {
            const isAppEnvironment = typeof window.Android !== "undefined";
            // 모바일 기기인지 체크하는 정규식
            const isMobileBrowser = /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent);

            if (isAppEnvironment) {
                // [1] 전용 앱 환경: 메인 화면 표시 + 앱 전용 UI 표시
                document.getElementById("mainControlPage").style.display = "block";
                document.getElementById("ui-app-connection").style.display = "flex";
                appendLog("[SYSTEM] 안드로이드 전용 앱 모드로 초기화되었습니다.");
            } 
            else if (isMobileBrowser) {
                // [2] 일반 모바일 브라우저 환경: 메인 화면 숨기고 경고 화면만 표시
                document.getElementById("mobileWarningPage").style.display = "flex";
            } 
            else {
                // [3] PC 브라우저 환경: 메인 화면 표시 + PC 전용 UI 표시
                document.getElementById("mainControlPage").style.display = "block";
                document.getElementById("ui-pc-connection").style.display = "flex";
                
                if (!navigator.serial) {
                    appendLog("[WARNING] 이 PC 브라우저는 Web Serial API를 지원하지 않습니다.");
                    document.getElementById("btnPcConnect").disabled = true;
                } else {
                    appendLog("[SYSTEM] PC Web Serial 통신 모드로 초기화되었습니다.");
                }
            }
        });

        // ==========================================
        // 2. 통합 통신 핸들러 (어댑터)
        // ==========================================
        const logBox = document.getElementById("logBox");
        function appendLog(msg) {
            logBox.textContent += msg + "\n";
            logBox.scrollTop = logBox.scrollHeight;
        }

        const SerialManager = {
            port: null, writer: null, reader: null, keepReading: false,

            // [PC 전용 연결 로직]
            async connectPC() {
                if (this.port) {
                    await this.disconnectPC();
                    return;
                }
                try {
                    const baudRate = parseInt(document.getElementById("baudRate").value);
                    this.port = await navigator.serial.requestPort();
                    await this.port.open({ baudRate: baudRate });
                    
                    const textEncoder = new TextEncoderStream();
                    textEncoder.readable.pipeTo(this.port.writable);
                    this.writer = textEncoder.writable.getWriter();
                    
                    // UI 업데이트
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

            // [PC 전용 해제 로직]
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

            // [앱 전용 연결 로직]
            connectApp() {
                window.Android.connectUSB(); // 네이티브 권한 팝업 등 호출
                appendLog("[SYSTEM] 네이티브 앱에 USB 연결을 요청했습니다.");
                
                // 앱에서 연결 성공 콜백이 왔다고 가정할 때 (실제로는 안드로이드에서 호출해줘야 함)
                document.getElementById("appStatusDot").classList.add("connected");
                document.getElementById("appStatusText").innerText = "장치 연결됨";
                document.getElementById("appStatusText").style.color = "var(--success)";
                document.getElementById("btnAppConnect").innerText = "연결 해제 (앱)";
                document.getElementById("btnAppConnect").classList.add("btn-danger");
            },

            // [공통 송신 로직] - 버튼에서 이걸 호출
            async write(data) {
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

            // [공통 수신 처리 로직]
            onReceiveData(line) {
                appendLog("RX: " + line);
                // 여기에 이사님 프로토콜 파싱 로직 추가 (parseFrame 등)
            },

            // [PC 전용 수신 루프]
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
                    appendLog("[ERROR] PC 읽기 오류: " + error);
                } finally {
                    this.reader.releaseLock();
                }
            }
        };

        // ★ 안드로이드 앱에서 수신한 데이터를 웹으로 밀어넣을 때 사용하는 전역 함수 ★
        function receiveDataFromAndroid(data) {
            SerialManager.onReceiveData(data);
        }
    </script>
</body>
</html>
