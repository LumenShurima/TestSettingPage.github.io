<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>통신 핸들러 어댑터 테스트</title>
    <style>
        body { font-family: sans-serif; padding: 20px; background: #f4f4f4; }
        .card { background: #fff; padding: 20px; border-radius: 8px; margin-bottom: 20px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
        button { padding: 10px 15px; margin: 5px; background: #007bff; color: white; border: none; border-radius: 4px; cursor: pointer; }
        button:hover { background: #0056b3; }
        button.danger { background: #dc3545; }
        #logBox { width: 100%; height: 300px; background: #222; color: #4af626; padding: 10px; font-family: monospace; overflow-y: auto; border-radius: 4px; }
        .status-dot { display: inline-block; width: 12px; height: 12px; border-radius: 50%; background: red; margin-right: 5px; }
        .status-dot.connected { background: #10b981; }
    </style>
</head>
<body>

    <div class="card">
        <h3>🔗 장치 연결 컨트롤</h3>
        <div style="margin-bottom: 15px; display: flex; align-items: center;">
            <div id="statusDot" class="status-dot"></div>
            <span id="statusText" style="font-weight: bold; margin-right: 15px;">연결대기</span>
            
            <select id="baudRate" style="padding: 8px;">
                <option value="9600">9600 bps</option>
                <option value="115200" selected>115200 bps</option>
            </select>
        </div>
        
        <button onclick="SerialManager.connect()">장치 연결 (자동 분기)</button>
        <button onclick="SerialManager.write('#0121defID9999W13287:1:97:*')">패킷 전송 테스트</button>
    </div>

    <div class="card" style="border-left: 5px solid #ffc107;">
        <h3>🛠️ 안드로이드 앱 환경 시뮬레이터 (테스트용)</h3>
        <p style="font-size: 13px; color: #666;">PC 브라우저에서도 앱 환경인 것처럼 강제로 속여서 테스트합니다.</p>
        <button onclick="mockAndroidEnvironment()" style="background: #ffc107; color: #000;">1. 안드로이드 환경으로 변환</button>
        <button class="danger" onclick="simulateAndroidReceive()">2. 앱에서 데이터 수신 (RX) 흉내내기</button>
    </div>

    <div id="logBox">통신 대기 중...</div>

    <script>
        // ==========================================
        // 1. 더미(Dummy) 함수들 (에러 방지용)
        // ==========================================
        const logBox = document.getElementById("logBox");
        function appendLog(msg) {
            logBox.textContent += msg + "\n";
            logBox.scrollTop = logBox.scrollHeight;
        }

        function requestInitialData() { appendLog("[SYSTEM] 초기 데이터 요청 완료"); }
        function parseRfidBlock(line) { /* 더미 파싱 */ }
        function parseFrame(line) { 
            // 더미 파싱 반환
            if(line.includes("#") && line.includes("*")) return { raw: line }; 
            return null; 
        }
        function BranchProcess(frame) { appendLog("   -> 파싱 완료 및 UI 업데이트 처리됨"); }


        // ==========================================
        // 2. 작성하신 통합 핸들러 (어댑터) - 그대로 가져옴
        // ==========================================
        const SerialManager = {
            // getter를 사용하여 환경을 실시간으로 체크하도록 살짝 수정 (테스트 편의상)
            get isAppEnvironment() { return typeof window.Android !== "undefined"; },
            
            port: null,
            writer: null,
            reader: null,
            keepReading: false,

            async connect() {
                if (this.isAppEnvironment) {
                    // [모바일 앱] 안드로이드 네이티브 연결 함수 호출
                    window.Android.connectUSB();
                    appendLog("[SYSTEM] 안드로이드 앱을 통한 USB 연결 요청...");
                    // 모바일용 연결 성공 UI 업데이트 (시뮬레이션)
                    document.getElementById("statusDot").classList.add("connected");
                    document.getElementById("statusText").innerText = "앱에 연결됨";
                } else if (navigator.serial) {
                    // [PC 웹] Web Serial API 호출
                    try {
                        const baudRate = parseInt(document.getElementById("baudRate").value);
                        this.port = await navigator.serial.requestPort();
                        await this.port.open({ baudRate: baudRate });
                        
                        const textEncoder = new TextEncoderStream();
                        textEncoder.readable.pipeTo(this.port.writable);
                        this.writer = textEncoder.writable.getWriter();
                        
                        appendLog(`[SYSTEM] PC Web Serial 연결 성공 (${baudRate}bps)`);
                        
                        this.keepReading = true;
                        this.readLoopPC(); // PC용 수신 루프 시작
                        
                        // UI 업데이트 및 초기 데이터 요청
                        document.getElementById("statusDot").classList.add("connected");
                        document.getElementById("statusText").innerText = "PC 연결됨";
                        requestInitialData();
                    } catch (err) {
                        appendLog("[ERROR] PC 연결 실패: " + err.message);
                    }
                } else {
                    alert("이 브라우저는 시리얼 통신을 지원하지 않습니다.");
                }
            },

            async write(data) {
                if (this.isAppEnvironment) {
                    // [모바일 앱] 안드로이드 네이티브로 데이터 넘김
                    window.Android.sendData(data);
                    appendLog("TX (App): " + data);
                } else if (this.writer) {
                    // [PC 웹] Web Serial로 데이터 쏨
                    await this.writer.write(data + "\r\n");
                    appendLog("TX (PC): " + data);
                } else {
                    alert("먼저 장치를 연결해주세요.");
                }
            },

            onReceiveData(line) {
                appendLog("RX: " + line);
                parseRfidBlock(line);
                const frame = parseFrame(line);
                if (frame) BranchProcess(frame);
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
                                if (line.trim()) {
                                    this.onReceiveData(line.trim());
                                }
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

        // ★ 안드로이드 앱에서 데이터를 받았을 때 호출할 전역 함수 ★
        function receiveDataFromAndroid(data) {
            SerialManager.onReceiveData(data);
        }

        // ==========================================
        // 3. 테스트용 시뮬레이터 함수
        // ==========================================
        
        // 브라우저에 가상의 window.Android 객체를 주입하여 속입니다.
        function mockAndroidEnvironment() {
            window.Android = {
                connectUSB: function() {
                    appendLog("   [MOCK-NATIVE] 안드로이드 권한 팝업 호출됨...");
                },
                sendData: function(data) {
                    appendLog("   [MOCK-NATIVE] USB로 데이터 발송 완료: " + data);
                }
            };
            appendLog("\n⚠️ 현재 환경이 [안드로이드 앱]으로 변경되었습니다.");
            appendLog("이제부터 '장치 연결'이나 '패킷 전송'을 누르면 앱 통신 로직을 탑니다.\n");
        }

        // 앱이 장치로부터 데이터를 읽어서 웹뷰로 넘겨주는 상황을 흉내냅니다.
        function simulateAndroidReceive() {
            if(!window.Android) {
                alert("먼저 '안드로이드 환경으로 변환' 버튼을 눌러주세요.");
                return;
            }
            // 안드로이드 네이티브가 이 함수를 호출했다고 가정
            const dummyPacket = "#0124defID9999R21:01012345678:*";
            receiveDataFromAndroid(dummyPacket);
        }
    </script>
</body>
</html>
