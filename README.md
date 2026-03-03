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
