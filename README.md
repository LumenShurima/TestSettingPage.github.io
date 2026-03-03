// 통신을 전담할 통합 핸들러 (어댑터)
const SerialManager = {
    // 현재 환경이 안드로이드 앱 내부인지 확인 (window.Android 객체 존재 여부)
    isAppEnvironment: typeof window.Android !== "undefined",
    
    // Web Serial용 변수 (PC용)
    port: null,
    writer: null,
    reader: null,
    keepReading: false,

    // 1. 연결 함수
    async connect() {
        if (this.isAppEnvironment) {
            // [모바일 앱] 안드로이드 네이티브 연결 함수 호출
            window.Android.connectUSB();
            appendLog("[SYSTEM] 안드로이드 앱을 통한 USB 연결 요청...");
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
                
                // UI 업데이트 및 초기 데이터 요청 (기존 로직)
                document.getElementById("statusDot").classList.add("connected");
                document.getElementById("statusText").innerText = "연결됨";
                requestInitialData();
            } catch (err) {
                appendLog("[ERROR] PC 연결 실패: " + err.message);
            }
        } else {
            alert("이 브라우저는 시리얼 통신을 지원하지 않습니다.");
        }
    },

    // 2. 송신 함수 (TX)
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

    // 3. 수신 처리 함수 (RX) - PC와 앱 모두 이 함수를 거침!
    onReceiveData(line) {
        appendLog("RX: " + line);
        
        // 이사님의 프로토콜 파싱 함수 호출
        parseRfidBlock(line);
        const frame = parseFrame(line);
        if (frame) BranchProcess(frame);
    },

    // [내부용] PC 웹 브라우저 수신 루프
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
                            // 공통 수신 함수로 넘김
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
