// ==========================================
// ★ 그새끼 프로토콜 맵 주소 매핑 (VB의 Select Case 대체) ★
// ==========================================
// VB 코드에 있던 수백 개의 Case 문을 이렇게 객체 하나로 압축합니다.
// (맵 번호: "UI_요소_ID" 형태로 매핑)
const MAP_ADDR = {
    1: "TB_EB_TYPE",             // 장비 타입
    4: "tb_deviceID_Set",        // 장비 ID
    19: "tb_ledblinkmode",       // LED 블링크 모드
    51: "TextBox61",             // LTE_ENG_MODE
    139: "TB_mp_volume",         // MP3 볼륨
    140: "tb_codec_vol",         // 스피커 볼륨
    141: "tb_mic_vol",           // 마이크 볼륨
    152: "tb_disalarm_tmr_type", // 경보 해제 타이머 타입
    153: "tb_RF_ALARM_PROTOCOL_TYPE",
    164: "tb_LEDOFFMODE",
    165: "tb_led_duty",
    254: "tb_attackRateTime",
    255: "tb_recoveryRateTime",
    256: "tb_postBoostGain",
    257: "tb_preBoostGain",
    258: "tb_ratio",
    259: "tb_compensationGain",
    260: "tb_limiterLevel",
    261: "tb_noiseGateTHHold",
    271: "tb_BUZZ_TO_ALARM",
    273: "TB_USE_BUZER",
    274: "TB_USE_SYSBUZER",
    287: "TB_CODE_85_data",      // DEVICE_CMD
    291: "tb_fotaPollingMin",    // FOTA 주기
    292: "tb_periodTime"         // 서버 접속 주기 (PERIOD_SMAC_MM)
};

// ==========================================
// ★ 메인 파싱 엔진 (VB의 processRxData_DebugMode + parcing_ASN500) ★
// ==========================================
function parseLegacyProtocol(rxdata) {
    rxdata = rxdata.trim();

    // ----------------------------------------------------
    // 1. 디버그/AT 명령어 / 상태 정보 파싱 (수많은 ElseIf 대체)
    // ----------------------------------------------------
    if (rxdata.startsWith("@") || rxdata.startsWith("AT") || rxdata.startsWith("$$")) {
        appendLog("[DEBUG/AT] " + rxdata);
        return;
    }

    // VB 코드의 수많은 rxdata.IndexOf("#변수명=") 처리기
    if (rxdata.includes("=")) {
        const parts = rxdata.replace(/#/g, "").replace(/\*/g, "").split("=");
        if (parts.length >= 2) {
            const key = parts[0].trim();
            const value = parts[1].trim();

            switch (key) {
                case "RSRP": updateUI("tb_sRSRP", value); break;
                case "RSSI": updateUI("tb_sRSSI", value); break;
                case "RSRQ": updateUI("tb_sRSRQ", value); break;
                case "SINR": updateUI("tb_sSINR", value); break;
                case "lteIMEI": updateUI("tb_lteimei", value); break;
                case "lteIMSI": updateUI("tb_lteimsi", value); break;
                case "lteCCID": updateUI("tb_lteccid", value); break;
                case "DEVICE_TYPE": updateUI("tb_modelName", "DNS-" + value); break;
                case "SIM": 
                    let simState = value == "1" ? "정상" : (value == "0" ? "확인중" : "에러");
                    updateUI("tb_sim_state", simState); 
                    break;
                case "NetState":
                    let netStateStr =["not Search", "HomeNet", "not reg", "denied", "unknown", "5"][parseInt(value)] || value;
                    updateUI("tb_netState", netStateStr);
                    break;
                case "TcpState": updateUI("tb_TcpState", value); break;
                case "batVolt": updateUI("tb_bat2", parseFloat(value).toFixed(2)); break;
                case "extVolt": updateUI("tb_bat4", parseFloat(value).toFixed(2)); break;
                case "bat_charger_stat": 
                    let batStat = value == "0" ? "충전중" : (value == "1" ? "충전완료" : "배터리 이상");
                    updateUI("tb_bat_state", batStat); 
                    break;
                case "stat_Alarm":
                    let alarmStat = value == "0" ? "정상" : "경보 발생";
                    updateUI("tb_alarmStat", alarmStat);
                    break;
                case "recvRFid":
                    updateUI("tb_recvRFid", value);
                    appendLog("📡 무선벨(RFid) 감지됨: " + value);
                    break;
            }
        }
    }

    // ----------------------------------------------------
    // 2. ASN-500 패킷 통신 파싱 (MAP 데이터 R/W)
    // ----------------------------------------------------
    // 형태: #01 24 defID9999 R 13 85:1:0:*
    const startIdx = rxdata.indexOf("#");
    const endIdx = rxdata.indexOf("*");
    
    if (startIdx < 0 || endIdx < 0) return; // 불완전한 패킷 무시
    
    let frame = rxdata.substring(startIdx, endIdx + 1);
    if (frame.startsWith("#OK*") || frame.length < 15) return;

    try {
        const trId = frame.substring(1, 3);
        const lenStr = frame.substring(3, 5);
        const deviceId = frame.substring(5, 14);
        const rw = frame.substring(14, 15).toUpperCase(); // R, W, C
        const code = frame.substring(15, 17);
        const payload = frame.substring(17, frame.length - 1); // '*' 제외

        appendLog(`[CMD_PARSED] 명령:${code} 읽기/쓰기:${rw} 장비ID:${deviceId}`);

        // Command Code (VB의 Select Case CODE)
        switch (code) {
            case "13": // 맵데이터 연속 (예: 287:1:1:)
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

            case "14": // 맵데이터 개별 (예: 140,5:141,3:)
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

            case "21": // 관제번호 응답
                const callNum = payload.replace(/:/g, "");
                updateUI("tb_callNum", callNum);
                appendLog(`📞 관제번호 확인됨: ${callNum}`);
                break;
            
            case "30": // 로그인 응답
                if (payload.startsWith("log:")) {
                    let logRes = payload.split(":")[1];
                    let logMsg = logRes == "1" ? "로그인 완료" : "로그인 실패";
                    appendLog(`🔐 CM 로그인 상태: ${logMsg}`);
                }
                break;
                
            case "38": // Ready to Data Send
                appendLog(`🚀 [READY TO DATA SEND] 펌웨어/음원 전송 준비 완료`);
                break;
        }
    } catch (e) {
        appendLog("[ERROR] 프로토콜 파싱 중 오류: " + e.message);
    }
}

// 맵 데이터 화면(UI) 반영 함수
function processMapData(address, value) {
    const targetId = MAP_ADDR[address];
    if (targetId) {
        updateUI(targetId, value);
        appendLog(`  -> [MAP 반영] 주소:${address} (${targetId}) = ${value}`);
    } else {
        appendLog(`  -> [MAP 수신] 주소:${address} (등록되지 않은 맵 변수) = ${value}`);
    }
}

// UI 텍스트박스 업데이트 안전 함수 (존재하는 UI 요소만 업데이트)
function updateUI(elementId, value) {
    const el = document.getElementById(elementId);
    if (el) {
        // 체크박스 처리
        if (el.type === 'checkbox') {
            el.checked = (value == "1");
        } 
        // 텍스트/셀렉트 처리
        else {
            el.value = value;
            // 값 변경 시 살짝 배경색 깜빡이는 효과 (VB의 BlinkTB 대체)
            el.style.backgroundColor = "#d1fae5"; // Light green
            setTimeout(() => el.style.backgroundColor = "", 500);
        }
    }
}
