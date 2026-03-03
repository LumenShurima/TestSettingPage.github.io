<html lang="ko">
<head>
    <meta charset="UTF-8">
    <!-- 모바일 화면 비율 고정 및 확대 방지 (UI 깨짐 1차 방지) -->
    <meta name="viewport"
      content="width=device-width, initial-scale=1">
    <title>비상벨 통합 제어 시스템</title>
    <style>
        :root {
            --primary: #0f172a; --primary-light: #1e293b; --accent: #3b82f6; --accent-hover: #2563eb;
            --bg-color: #f1f5f9; --card-bg: #ffffff; --text-main: #334155; --text-muted: #64748b;
            --border: #e2e8f0; --danger: #ef4444; --success: #10b981;
        }

        /* -------------------------------------------
           전역 초기화 및 100% 꽉 찬 레이아웃 (스크롤 오류 해결)
        ------------------------------------------- */
        html, body {
            width: 100%; height: 100%; margin: 0; padding: 0;
            background-color: var(--bg-color); color: var(--text-main);
            font-family: 'Pretendard', -apple-system, sans-serif;
            -webkit-tap-highlight-color: transparent; box-sizing: border-box;
            overflow-x: hidden;
        }
        *, *::before, *::after { box-sizing: inherit; }

        /* 전체 앱 컨테이너 */
        .app-container {
            display: flex; flex-direction: column;
            width: 100vw; max-width: 100%; height: 100%; overflow: hidden;
        }

        /* -------------------------------------------
           상단바 (Topbar) - 반응형 Flex 적용
        ------------------------------------------- */
        .topbar {
            background: #fff; padding: 12px 16px; border-bottom: 1px solid var(--border);
            display: flex; justify-content: space-between; align-items: center;
            flex-wrap: wrap; gap: 10px; z-index: 10; flex-shrink: 0;
        }
        .topbar-left, .topbar-right { display: flex; align-items: center; gap: 10px; }
        .menu-btn {
            background: transparent; border: none; font-size: 24px; color: var(--primary);
            cursor: pointer; padding: 4px; display: flex; align-items: center;
        }
        .status-group { display: flex; align-items: center; gap: 8px; }
        .status-dot { width: 12px; height: 12px; border-radius: 50%; background: var(--danger); box-shadow: 0 0 0 3px rgba(239,68,68,0.2); flex-shrink: 0; }
        .status-dot.connected { background: var(--success); box-shadow: 0 0 0 3px rgba(16,185,129,0.2); }
        .target-input { padding: 4px; font-size: 12px; width: 80px; border: 1px solid #ccc; border-radius: 4px; background: #fff; }

        /* 모바일 좁은 화면에서 상단바 줄바꿈 처리 */
        @media (max-width: 480px) {
            .topbar { flex-direction: column; align-items: stretch; }
            .topbar-left, .topbar-right { justify-content: space-between; width: 100%; }
        }

        /* -------------------------------------------
           팝업 사이드바 (드로어 메뉴)
        ------------------------------------------- */
        .overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.5); z-index: 999; display: none; opacity: 0; transition: 0.3s; }
        .sidebar {
            position: fixed; top: 0; left: -260px; width: 260px; height: 100%;
            background: var(--primary); color: white; display: flex; flex-direction: column;
            z-index: 1000; transition: left 0.3s cubic-bezier(0.4, 0, 0.2, 1); box-shadow: 4px 0 10px rgba(0,0,0,0.2);
        }
        .sidebar.open { left: 0; }
        .sidebar-header { padding: 20px; background: #020617; font-weight: 800; font-size: 18px; display: flex; justify-content: space-between; align-items: center; }
        .btn-close-menu { background: transparent; border: none; color: white; font-size: 24px; cursor: pointer; padding: 0 8px; }
        .nav-menu { list-style: none; flex: 1; overflow-y: auto; padding: 10px 0; margin: 0; }
        .nav-item { padding: 16px 20px; cursor: pointer; font-size: 15px; font-weight: 500; border-left: 4px solid transparent; border-bottom: 1px solid rgba(255,255,255,0.05); transition: 0.2s; }
        .nav-item:hover, .nav-item.active { background: var(--primary-light); border-left-color: var(--accent); color: #60a5fa; }

        /* -------------------------------------------
           메인 스크롤 콘텐츠 영역 (핵심 레이아웃)
        ------------------------------------------- */
        .content-scroll {
            flex: 1; 
            min-height: 0; /* 🔥 [핵심] 컨테이너가 무한정 늘어나는 걸 막아 스크롤을 강제로 활성화합니다! */
            padding: 16px; 
            overflow-y: auto; 
            -webkit-overflow-scrolling: touch;
            display: block; /* 모바일에서 Flex 중첩으로 인한 스크롤 꼬임 방지 */
        }
        .page { 
            display: none; 
            flex-direction: column; 
            animation: fadeIn 0.3s ease; 
            padding-bottom: 40px; /* 맨 아래쪽 버튼이 잘 안 눌리는 것을 방지하기 위한 여백 */
        }
        .page.active { 
            display: flex; 
        }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(5px); } to { opacity: 1; transform: translateY(0); } }

        /* 카드 및 버튼 UI */
        .card { background: var(--card-bg); border-radius: 10px; padding: 16px; margin-bottom: 16px; border: 1px solid var(--border); box-shadow: 0 2px 4px rgba(0,0,0,0.02); flex-shrink: 0; }
        .card-title { font-size: 15px; font-weight: 700; margin-bottom: 16px; color: var(--primary); border-bottom: 2px solid var(--bg-color); padding-bottom: 8px; }
        
        button { cursor: pointer; border: none; border-radius: 6px; font-weight: 600; font-size: 14px; padding: 10px 16px; transition: 0.2s; color: white; white-space: nowrap; }
        .btn-accent { background: var(--accent); } .btn-accent:hover { background: var(--accent-hover); }
        .btn-danger { background: var(--danger); }
        .btn-dark { background: var(--primary-light); }
        .btn-outline { background: transparent; border: 1px solid var(--border); color: var(--text-main); }
        .btn-outline:hover { background: var(--bg-color); }

        input, select { padding: 10px; border: 1px solid var(--border); border-radius: 6px; font-size: 14px; outline: none; width: 100%; min-width: 0; background: #fff; }
        input:focus, select:focus { border-color: var(--accent); }

        /* -------------------------------------------
           반응형 그리드 및 폼
        ------------------------------------------- */
        .grid-cards { display: grid; gap: 16px; grid-template-columns: 1fr; }
        .grid-buttons { display: grid; gap: 8px; grid-template-columns: repeat(2, 1fr); }
        .grid-widgets { display: grid; gap: 10px; grid-template-columns: repeat(2, 1fr); }
        
        @media (min-width: 768px) {
            .grid-cards { grid-template-columns: 1fr 1fr; }
            .grid-buttons { grid-template-columns: repeat(4, 1fr); }
            .grid-widgets { grid-template-columns: repeat(4, 1fr); }
        }

        .form-row { display: flex; align-items: center; gap: 8px; margin-bottom: 12px; flex-wrap: wrap; }
        .form-row label { width: 130px; font-size: 13px; font-weight: 600; color: var(--text-muted); flex-shrink: 0; }
        .form-inputs { display: flex; flex: 1; gap: 8px; min-width: 200px; }
        
        @media (max-width: 600px) {
            .form-row { flex-direction: column; align-items: stretch; }
            .form-row label { width: 100%; margin-bottom: 4px; }
            .form-inputs { flex-direction: column; }
        }

        /* 상태 위젯 */
        .widget { background: var(--bg-color); padding: 12px; border-radius: 8px; text-align: center; border: 1px solid var(--border); }
        .widget .label { font-size: 11px; color: var(--text-muted); display: block; margin-bottom: 4px; }
        .widget .value { font-size: 16px; font-weight: 800; color: var(--accent); }

        /* 스위치 스타일 */
        .switch-row { display: flex; justify-content: space-between; align-items: center; padding: 12px 0; border-bottom: 1px solid var(--border); }
        .switch-row:last-child { border-bottom: none; }
        .switch-label { font-size: 14px; font-weight: 600; color: var(--text-main); }
        .switch { position: relative; width: 46px; height: 26px; display: inline-block; flex-shrink: 0;}
        .switch-input { opacity: 0; width: 0; height: 0; position: absolute; }
        .switch-track { position: absolute; inset: 0; background: #cbd5e1; border-radius: 26px; transition: 0.3s; cursor: pointer; }
        .switch-thumb { position: absolute; top: 2px; left: 2px; width: 22px; height: 22px; background: white; border-radius: 50%; transition: 0.3s; box-shadow: 0 2px 4px rgba(0,0,0,0.2); pointer-events: none;}
        .switch-input:checked ~ .switch-track { background: var(--success); }
        .switch-input:checked ~ .switch-thumb { transform: translateX(20px); }

        /* 터미널 화면 100% 대응 */
        #logBox { flex: 1; background: #0f172a; color: #a7f3d0; padding: 12px; font-family: 'Consolas', monospace; font-size: 12px; overflow-y: auto; border-radius: 8px; line-height: 1.5; word-break: break-all; min-height: 300px;}
        .log-tx { color: #93c5fd; } .log-rx { color: #fde047; } .log-sys { color: #94a3b8; }
    </style>
</head>
<body>

<div class="app-container">

    <!-- 사이드바 오버레이 (바탕 클릭시 닫힘) -->
    <div id="sidebarOverlay" class="overlay" onclick="toggleMenu()"></div>

    <!-- 좌측 사이드바 (팝업 형태) -->
    <div id="sidebar" class="sidebar">
        <div class="sidebar-header">
            <span>DNS SETUP</span>
            <button class="btn-close-menu" onclick="toggleMenu()">✕</button>
        </div>
        <ul class="nav-menu">
            <li class="nav-item active" onclick="nav('page-dash', this)">대시보드</li>
            <li class="nav-item" onclick="nav('page-net', this)">네트워크 설정</li>
            <li class="nav-item" onclick="nav('page-audio', this)">오디오 / 볼륨</li>
            <li class="nav-item" onclick="nav('page-rf', this)">무선벨(RF) 등록</li>
            <li class="nav-item" onclick="nav('page-setting', this)">상세 동작 설정</li>
            <li class="nav-item" onclick="nav('page-term', this)">터미널 로그</li>
        </ul>
    </div>

    <!-- 상단바 -->
    <div class="topbar">
        <div class="topbar-left">
            <button class="menu-btn" onclick="toggleMenu()">☰</button>
            <div class="status-group">
                <div id="statusDot" class="status-dot"></div>
                <div>
                    <div id="statusText" style="font-weight: 700; font-size: 14px;">연결 대기중</div>
                    <div style="font-size: 11px; color: var(--text-muted); display:flex; align-items:center; gap:4px; margin-top:2px;">
                        ID: <input type="text" id="targetId" class="target-input" value="defID9999">
                    </div>
                </div>
            </div>
        </div>
        <div class="topbar-right">
            <select id="baudRate" style="width: 90px; padding: 8px;">
                <option value="115200" selected>115200</option>
                <option value="9600">9600</option>
            </select>
            <button id="btnConnect" class="btn-dark" onclick="SerialManager.toggleConnect()">USB 연결</button>
        </div>
    </div>

    <!-- 메인 스크롤 영역 -->
    <div class="content-scroll">
        
        <!-- 1. 대시보드 -->
        <div id="page-dash" class="page active">
            <div class="card">
                <div class="card-title">⚡ 빠른 제어</div>
                <div class="grid-buttons">
                    <button class="btn-danger" onclick="DeviceControl.triggerAlarm()">🚨 경보 발생</button>
                    <button class="btn-outline" onclick="DeviceControl.clearAlarm()">✅ 경보 해제</button>
                    <button class="btn-dark" onclick="DeviceControl.resetDevice()">🔄 장치 리셋</button>
                    <button class="btn-dark" onclick="DeviceControl.resetModem()">📶 모뎀 리셋</button>
                </div>
            </div>

            <div class="card">
                <div class="card-title">📊 상태 모니터링</div>
                <div class="grid-widgets">
                    <div class="widget"><span class="label">LTE RSRP</span><span class="value" id="val-rsrp">--</span></div>
                    <div class="widget"><span class="label">LTE RSSI</span><span class="value" id="val-rssi">--</span></div>
                    <div class="widget"><span class="label">배터리 전압</span><span class="value" id="val-battery">-- V</span></div>
                    <div class="widget"><span class="label">USIM 상태</span><span class="value" id="val-sim">--</span></div>
                    <div class="widget"><span class="label">기울기(X)</span><span class="value" id="val-tilt-x">--</span></div>
                    <div class="widget"><span class="label">기울기(Y)</span><span class="value" id="val-tilt-y">--</span></div>
                    <div class="widget"><span class="label">충전 상태</span><span class="value" id="val-charger">--</span></div>
                    <div class="widget"><span class="label">펌웨어</span><span class="value" id="val-fw">--</span></div>
                </div>
                <button class="btn-outline" style="width: 100%; margin-top: 16px;" onclick="DeviceControl.requestStatus()">상태 새로고침</button>
            </div>
        </div>

        <!-- 2. 네트워크 설정 -->
        <div id="page-net" class="page">
            <div class="card">
                <div class="card-title">🌐 서버 IP 설정</div>
                <div class="form-row">
                    <label>메인 서버 (관제)</label>
                    <div class="form-inputs">
                        <input type="text" id="inp-main-ip" placeholder="192.168.0.1:12100">
                        <button class="btn-outline" onclick="DeviceControl.readMap(200, 6)">읽기</button>
                        <button class="btn-accent" onclick="DeviceControl.setServerIP(200, 'inp-main-ip')">쓰기</button>
                    </div>
                </div>
                <div class="form-row">
                    <label>제조사 서버 (DMS)</label>
                    <div class="form-inputs">
                        <input type="text" id="inp-dms-ip" placeholder="192.168.0.2:12100">
                        <button class="btn-outline" onclick="DeviceControl.readMap(212, 6)">읽기</button>
                        <button class="btn-accent" onclick="DeviceControl.setServerIP(212, 'inp-dms-ip')">쓰기</button>
                    </div>
                </div>
            </div>
            <div class="card">
                <div class="card-title">📞 관제 번호 설정</div>
                <div class="form-row">
                    <label>메인 관제 번호</label>
                    <div class="form-inputs">
                        <input type="text" id="inp-call-1" placeholder="01012345678">
                        <button class="btn-accent" onclick="DeviceControl.setCallNum('21', 'inp-call-1')">쓰기</button>
                    </div>
                </div>
                <div class="form-row">
                    <label>보조 관제 번호</label>
                    <div class="form-inputs">
                        <input type="text" id="inp-call-2" placeholder="01012345678">
                        <button class="btn-accent" onclick="DeviceControl.setCallNumMap(357, 'inp-call-2')">쓰기</button>
                    </div>
                </div>
            </div>
        </div>

        <!-- 3. 오디오/볼륨 -->
        <div id="page-audio" class="page">
            <div class="card">
                <div class="card-title">🔊 기본 볼륨</div>
                <div class="form-row">
                    <label>스피커 볼륨 (0~7)</label>
                    <div class="form-inputs">
                        <input type="number" id="inp-vol-spk" min="0" max="7" value="5">
                        <button class="btn-accent" onclick="DeviceControl.writeMap(85,[document.getElementById('inp-vol-spk').value])">적용</button>
                    </div>
                </div>
                <div class="form-row">
                    <label>마이크 볼륨 (0~63)</label>
                    <div class="form-inputs">
                        <input type="number" id="inp-vol-mic" min="0" max="63" value="40">
                        <button class="btn-accent" onclick="DeviceControl.writeMap(88,[document.getElementById('inp-vol-mic').value])">적용</button>
                    </div>
                </div>
                <div class="form-row">
                    <label>MP3 볼륨 (0~30)</label>
                    <div class="form-inputs">
                        <input type="number" id="inp-vol-mp3" min="0" max="30" value="15">
                        <button class="btn-accent" onclick="DeviceControl.writeMap(139,[document.getElementById('inp-vol-mp3').value])">적용</button>
                    </div>
                </div>
            </div>
        </div>

        <!-- 4. 무선벨(RF) -->
        <div id="page-rf" class="page">
            <div class="card">
                <div class="card-title">📡 무선벨 등록 (Code 25)</div>
                <div class="form-row">
                    <label>저장 번지 (1~32)</label>
                    <div class="form-inputs"><input type="number" id="inp-rf-idx" min="1" max="32" value="1"></div>
                </div>
                <div class="form-row">
                    <label>무선벨 ID (4자리)</label>
                    <div class="form-inputs"><input type="text" id="inp-rf-id" maxlength="4" placeholder="A001" value="A001"></div>
                </div>
                <div class="form-row">
                    <label>연속 등록 갯수</label>
                    <div class="form-inputs"><input type="number" id="inp-rf-cnt" min="1" max="10" value="1"></div>
                </div>
                <div style="display:flex; gap:8px; margin-top: 16px;">
                    <button class="btn-outline" style="flex:1;" onclick="DeviceControl.readRF()">읽기</button>
                    <button class="btn-danger" style="flex:1;" onclick="DeviceControl.clearRF()">전체 삭제</button>
                    <button class="btn-accent" style="flex:1;" onclick="DeviceControl.registerRF()">등록 (쓰기)</button>
                </div>
            </div>
        </div>

        <!-- 5. 상세 동작 설정 -->
        <div id="page-setting" class="page">
            <div class="grid-cards">
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

                <div class="card">
                    <div class="card-title">시스템 부가 설정</div>
                    <div class="switch-row"><span class="switch-label">앰프(Amp) 사용</span><label class="switch"><input type="checkbox" id="sys_amp" class="switch-input-sys" data-map="84"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
                    <div class="switch-row"><span class="switch-label">부저(Buzzer) 사용</span><label class="switch"><input type="checkbox" id="sys_buzzer" class="switch-input-sys" data-map="273"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
                    <div class="switch-row"><span class="switch-label">ALC 사용 (자동 볼륨)</span><label class="switch"><input type="checkbox" id="sys_alc" class="switch-input-sys" data-map="427"><span class="switch-track"></span><span class="switch-thumb"></span></label></div>
                </div>
            </div>
        </div>

        <!-- 6. 터미널 -->
        <div id="page-term" class="page">
            <div class="card" style="padding: 10px; margin-bottom: 10px; flex-shrink: 0;">
                <div class="form-row" style="margin: 0;">
                    <div class="form-inputs">
                        <input type="text" id="inp-raw-cmd" placeholder="패킷 입력 (#0119defID9999R14292,0::*)">
                        <button class="btn-accent" onclick="DeviceControl.sendRaw()">전송</button>
                        <button class="btn-outline" onclick="document.getElementById('logBox').innerHTML=''">Clear</button>
                    </div>
                </div>
            </div>
            <div id="logBox"></div>
        </div>

    </div>
</div>

<script>
    // 전역 변수 설정
    const SBA = Array(15).fill(false);
    const SWITCH_ID = {
        0: "touch_alram", 1: "touch_transmit", 2: "emerg_transmit", 3: "touch_sms", 4: "emerg_sms",
        5: "emerg_alram", 6: "touch_light", 7: "emerg_light", 8: "touch_mp3", 9: "emerg_mp3",
        10: "rf_alram", 11: "rf_transmit", 12: "rf_sms", 13: "rf_light", 14: "rf_mp3"
    };

    // ==========================================
    // 1. UI 네비게이션 및 메뉴 토글
    // ==========================================
    function toggleMenu() {
        const sidebar = document.getElementById('sidebar');
        const overlay = document.getElementById('sidebarOverlay');
        sidebar.classList.toggle('open');
        
        if (sidebar.classList.contains('open')) {
            overlay.style.display = 'block';
            setTimeout(() => overlay.style.opacity = '1', 10);
        } else {
            overlay.style.opacity = '0';
            setTimeout(() => overlay.style.display = 'none', 300);
        }
    }

    function nav(pageId, el) {
        document.querySelectorAll('.nav-item').forEach(n => n.classList.remove('active'));
        document.querySelectorAll('.page').forEach(p => p.classList.remove('active'));
        el.classList.add('active');
        document.getElementById(pageId).classList.add('active');
        
        // 메뉴 선택 시 드로어 닫기
        if (document.getElementById('sidebar').classList.contains('open')) {
            toggleMenu();
        }
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
    // 2. 통신 매니저 (Web Serial + Android)
    // ==========================================
    const SerialManager = {
        port: null, writer: null, reader: null, keepReading: false, isConnected: false,

        async toggleConnect() {
            if (this.isConnected) {
                await this.disconnect();
            } else {
                if (typeof window.Android !== "undefined") {
                    window.Android.connectUSB();
                    this.setUI(true);
                    setTimeout(() => DeviceControl.requestInitialData(), 1000);
                } else if (navigator.serial) {
                    await this.connectPC();
                } else {
                    alert("Web Serial API를 지원하지 않는 브라우저입니다.");
                }
            }
        },

        async connectPC() {
            try {
                const baudRate = parseInt(document.getElementById("baudRate").value);
                this.port = await navigator.serial.requestPort();
                await this.port.open({ baudRate: baudRate });
                
                const textEncoder = new TextEncoderStream();
                textEncoder.readable.pipeTo(this.port.writable);
                this.writer = textEncoder.writable.getWriter();
                
                this.setUI(true);
                appendLog(`포트 열림 (${baudRate}bps)`, "sys");
                
                setTimeout(() => DeviceControl.requestInitialData(), 500);

                this.keepReading = true;
                this.readLoopPC();
            } catch (err) {
                appendLog(`연결 실패: ${err.message}`, "sys");
            }
        },

        async disconnect() {
            this.keepReading = false;
            if (this.reader) { await this.reader.cancel(); this.reader = null; }
            if (this.writer) { await this.writer.close(); this.writer = null; }
            if (this.port) { await this.port.close(); this.port = null; }
            if (typeof window.Android !== "undefined") window.Android.disconnectUSB();
            
            this.setUI(false);
            appendLog("포트 닫힘", "sys");
        },

        setUI(connected) {
            this.isConnected = connected;
            document.getElementById("statusDot").className = `status-dot ${connected ? 'connected' : ''}`;
            document.getElementById("statusText").innerText = connected ? "연결됨" : "연결 대기중";
            document.getElementById("statusText").style.color = connected ? "var(--success)" : "var(--text-main)";
            const btn = document.getElementById("btnConnect");
            btn.innerText = connected ? "연결 해제" : "USB 연결";
            btn.className = connected ? "btn-danger" : "btn-dark";
        },

        async write(data) {
            if (!this.isConnected) return alert("장치를 먼저 연결해주세요.");
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
                appendLog(`읽기 오류: ${error}`, "sys");
            } finally {
                this.reader.releaseLock();
            }
        }
    };

    window.receiveDataFromAndroid = function(data) { SerialManager.onReceiveData(data); };

    // ==========================================
    // 3. 비즈니스 로직 (프로토콜 및 제어)
    // ==========================================
    const DeviceControl = {
        makePacket(rw, code, dataStr) {
            const tid = document.getElementById('targetId').value || "defID9999";
            const trId = "01";
            const payload = tid.padStart(9, '0') + rw + code + dataStr;
            const lengthStr = payload.length.toString().padStart(2, '0');
            return `#${trId}${lengthStr}${payload}*`;
        },

        writeMap(address, values) {
            const dataStr = `${address}:${values.length}:${values.join(',')}:`;
            SerialManager.write(this.makePacket("W", "13", dataStr));
        },

        readMap(address, count) {
            const dataStr = `${address}:${count}::`;
            SerialManager.write(this.makePacket("R", "13", dataStr));
        },

        requestInitialData() {
            SerialManager.write(this.makePacket("R", "14", "80,1:91,1:104,1:84,1:273,1:427,1:"));
            SerialManager.write(this.makePacket("R", "13", "28:9:0,0,0,0,0,0,0,0,0:"));
            SerialManager.write(this.makePacket("R", "13", "154:4:0,0,0,0:"));
        },

        triggerAlarm() { this.writeMap(287, [97]); },
        clearAlarm() { this.writeMap(287, [4]); },
        resetDevice() { if(confirm("장치 리셋?")) this.writeMap(287, [1]); },
        resetModem() { if(confirm("모뎀 리셋?")) this.writeMap(287,[2]); },
        requestStatus() { SerialManager.write("AT#KTDEVSTAT"); },

        setServerIP(mapAddr, inputId) {
            const ipPort = document.getElementById(inputId).value.split(":");
            if(ipPort.length !== 2) return alert("IP:PORT 형식 확인 (예: 192.168.0.1:12100)");
            const ips = ipPort[0].split(".");
            const port = parseInt(ipPort[1]);
            this.writeMap(mapAddr,[ips[0], ips[1], ips[2], ips[3], Math.floor(port/256), port%256]);
        },

        setCallNum(code, inputId) {
            const num = document.getElementById(inputId).value.replace(/-/g, "");
            SerialManager.write(this.makePacket("W", code, `:${num}:`));
        },
        
        setCallNumMap(mapAddr, inputId) {
            const num = document.getElementById(inputId).value.replace(/-/g, "").padEnd(11, '\0');
            let arr =[];
            for(let i=0; i<11; i++) arr.push(num.charCodeAt(i));
            this.writeMap(mapAddr, arr);
        },

        registerRF() {
            const idx = document.getElementById('inp-rf-idx').value.padStart(2, '0');
            const id = document.getElementById('inp-rf-id').value;
            const cnt = document.getElementById('inp-rf-cnt').value;
            let ids =[];
            for(let i=0; i<cnt; i++) ids.push(id);
            SerialManager.write(this.makePacket("W", "25", `:${idx}:${ids.join(',')}:`));
        },
        readRF() {
            const idx = document.getElementById('inp-rf-idx').value.padStart(2, '0');
            SerialManager.write(this.makePacket("R", "25", `:${idx}::`));
        },
        clearRF() {
            if(confirm("모든 무선벨을 삭제합니까?")) this.writeMap(287,[92]);
        },

        sendRaw() {
            const cmd = document.getElementById('inp-raw-cmd').value;
            if(cmd) SerialManager.write(cmd);
        }
    };

    // ==========================================
    // 4. 수신 데이터 파서 및 프로토콜 파싱
    // ==========================================
    const DeviceParser = {
          parse(line) {
            // 1) AT 상태 문자열 파싱 (기존 유지 + 소폭 개선)
            if (line.includes("#RSRP=")) this.update("val-rsrp", this.ext(line, "#RSRP="));
            if (line.includes("#RSSI=")) this.update("val-rssi", this.ext(line, "#RSSI="));
            if (line.includes("#BAT_AVR_FLOAT=")) {
              const v = parseFloat(this.ext(line, "#BAT_AVR_FLOAT="));
              this.update("val-battery", Number.isFinite(v) ? v.toFixed(2) + " V" : "-- V");
            }
            if (line.includes("#SIM=")) {
              const sim = this.ext(line, "#SIM=");
              this.update("val-sim", sim === "1" ? "정상" : (sim === "0" ? "확인중" : "에러"));
            }
            if (line.includes("#tilt_x=")) this.update("val-tilt-x", this.ext(line, "#tilt_x="));
            if (line.includes("#tilt_y=")) this.update("val-tilt-y", this.ext(line, "#tilt_y="));
            if (line.includes("#CHARGER=")) this.update("val-charger", this.ext(line, "#CHARGER="));
            if (line.includes("#PKG_VER=")) this.update("val-fw", this.ext(line, "#PKG_VER=").split("").join("."));
        
            // 2) 프로토콜 프레임 파싱 (새 구현)
            this.parseFrames(line);
          },
        
          // AT류는 '*'가 있기도/없기도 해서 기존 ext 유지
          ext(line, key) {
            try {
              const start = line.indexOf(key) + key.length;
              const end = line.indexOf("*", start);
              return end !== -1 ? line.substring(start, end).trim() : line.substring(start).trim();
            } catch (e) {
              return "--";
            }
          },
        
          update(id, val) {
            const el = document.getElementById(id);
            if (!el) return;
            const prev = el.style.color;
            el.innerText = val;
            el.style.color = "var(--accent)";
            // 500ms 후 원래색(없으면 기본색)으로 복귀
            setTimeout(() => { el.style.color = prev || "var(--text-main)"; }, 500);
          },
        
          // ------------------------------
          // 프로토콜 프레임 파싱 (길이 기반)
          // 규칙: # + TrID(2) + Len(2) + payload(Len) + *
          // payload: TID(9) + RW(1) + CODE(2) + DATA(variable)
          // ------------------------------
          parseFrames(line) {
            let i = 0;
            while (true) {
              const s = line.indexOf("#", i);
              if (s < 0) break;
              const e = line.indexOf("*", s + 1);
              if (e < 0) break;
        
              const frame = line.slice(s, e + 1); // '*' 포함
              i = e + 1;
        
              // #OK 같은 잡 프레임 제외
              if (frame.startsWith("#OK")) continue;
        
              const parsed = this.parseFrame(frame);
              if (!parsed) continue;
        
              this.branchProcess(parsed.codeNum, parsed.data, parsed);
            }
          },
        
          parseFrame(frame) {
            // 최소 길이: # + 2 + 2 + payload(>=12) + *
            // payload 최소: tid(9)+rw(1)+code(2)=12
            if (frame.length < 1 + 2 + 2 + 12 + 1) return null;
            if (frame[0] !== "#" || frame[frame.length - 1] !== "*") return null;
        
            const trId = frame.substr(1, 2);
            const lenStr = frame.substr(3, 2);
            const payloadLen = Number(lenStr);
        
            if (!Number.isFinite(payloadLen) || payloadLen < 12) return null;
        
            // 전체 프레임 길이 검증: '#'(1)+trId(2)+len(2)+payload(payloadLen)+'*'(1) = payloadLen + 6
            const expectedTotal = payloadLen + 6;
            if (frame.length !== expectedTotal) {
              // 길이 불일치면 잡음/부분수신 가능성 큼 -> 무시
              return null;
            }
        
            const payload = frame.substr(5, payloadLen); // 5부터 payloadLen
            const tid = payload.substr(0, 9);
            const rw = payload.substr(9, 1);
            const codeStr = payload.substr(10, 2);
            const codeNum = Number(codeStr);
            const data = payload.substr(12); // 나머지
        
            if (!Number.isFinite(codeNum)) return null;
            if (rw !== "R" && rw !== "W" && rw !== "r" && rw !== "w") return null;
        
            return { trId, payloadLen, tid, rw, codeStr, codeNum, data, raw: frame };
          },
        
          // ------------------------------
          // CODE별 처리
          // ------------------------------
          branchProcess(cmdNum, data, meta) {
            // data는 trailing ':'가 붙는 경우가 많아서 여기서 공통 정리
            const cleaned = (data || "").endsWith(":") ? data.slice(0, -1) : (data || "");
        
            switch (cmdNum) {
              case 13:
                this.handleCode13(cleaned);
                break;
        
              case 14:
                this.handleCode14(cleaned);
                break;
        
              default:
                // 다른 코드면 무시 (필요시 확장)
                break;
            }
          },
        
          // CODE 13: "addr:count:csv"
          handleCode13(data) {
            // 예: "28:9:0,0,0,0,0,0,0,0,0"
            const parts = data.split(":");
            if (parts.length < 3) return;
        
            const addr = Number(parts[0]);
            const count = Number(parts[1]);
        
            // 값 영역은 parts[2]가 기본. 뒤에 ':'가 더 붙는 변형이면 join으로 흡수
            const valuesCsv = parts.slice(2).join(":");
            const values = valuesCsv.length ? valuesCsv.split(",") : [];
            // 값은 문자열로 들어오니 Number 변환은 쓰는 쪽에서 필요할 때만
            // count 검증(엄격히 하려면): values.length === count
        
            if (addr === 28) {
              // "28:9:(n1..n9)" 매핑
              // 기존 코드의 인덱스 유지
              const v = values.map(x => Number(x));
              document.getElementById("touch_transmit").checked = SBA[1] = !!v[0];
              document.getElementById("emerg_transmit").checked = SBA[2] = !!v[1];
              document.getElementById("touch_sms").checked = SBA[3] = !!v[2];
              document.getElementById("emerg_sms").checked = SBA[4] = !!v[3];
              document.getElementById("touch_light").checked = SBA[6] = !!v[5];
              document.getElementById("emerg_light").checked = SBA[7] = !!v[6];
              document.getElementById("touch_mp3").checked = SBA[8] = !!v[7];
              document.getElementById("emerg_mp3").checked = SBA[9] = !!v[8];
            } else if (addr === 154) {
              const v = values.map(x => Number(x));
              document.getElementById("rf_transmit").checked = SBA[11] = !!v[0];
              document.getElementById("rf_sms").checked = SBA[12] = !!v[1];
              document.getElementById("rf_light").checked = SBA[13] = !!v[2];
              document.getElementById("rf_mp3").checked = SBA[14] = !!v[3];
            }
          },
        
          // CODE 14: "a,b:a,b:a,b" (콜론으로 레코드 구분)
          handleCode14(data) {
            // 예: "80,1:91,1:104,1:84,1:273,1:427,1"
            const segments = data.split(":").filter(s => s.length);
            for (const seg of segments) {
              if (!seg.includes(",")) continue;
              const [aStr, bStr] = seg.split(",");
              const a = Number(aStr);
              const b = Number(bStr);
              if (!Number.isFinite(a) || !Number.isFinite(b)) continue;
        
              if (a === 80)  document.getElementById("touch_alram").checked = SBA[0] = !!b;
              if (a === 91)  document.getElementById("emerg_alram").checked = SBA[5] = !!b;
              if (a === 104) document.getElementById("rf_alram").checked = SBA[10] = !!b;
        
              if (a === 84)  document.getElementById("sys_amp").checked = !!b;
              if (a === 273) document.getElementById("sys_buzzer").checked = !!b;
              if (a === 427) document.getElementById("sys_alc").checked = !!b;
            }
          }
        };
    
    // ==========================================
    // 5. 스위치 UI 이벤트 바인딩
    // ==========================================
    window.onload = () => {
        document.querySelectorAll(".switch-input").forEach(input => {
            input.addEventListener("change", function() {
                const idx = Number(Object.keys(SWITCH_ID).find(k => SWITCH_ID[k] === this.id));
                if(!isNaN(idx)) SBA[idx] = this.checked;
                const n = SBA.map(v => v ? 1 : 0);
                
                SerialManager.write(DeviceControl.makePacket("W", "14", `80,${n[0]}:91,${n[5]}:104,${n[10]}:`));
                SerialManager.write(DeviceControl.makePacket("W", "13", `28:9:${n[1]},${n[2]},${n[3]},${n[4]},1,${n[6]},${n[7]},${n[8]},${n[9]}:`));
                SerialManager.write(DeviceControl.makePacket("W", "13", `154:4:${n[11]},${n[12]},${n[13]},${n[14]}:`));
            });
        });

        document.querySelectorAll(".switch-input-sys").forEach(input => {
            input.addEventListener("change", function() {
                const mapAddr = this.getAttribute("data-map");
                const val = this.checked ? 1 : 0;
                SerialManager.write(DeviceControl.makePacket("W", "14", `${mapAddr},${val}:`));
            });
        });
    };
</script>
</body>
</html>
