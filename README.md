<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>經典乒乓球遊戲 - GitHub 版</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #1a1a1a;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            touch-action: none;
        }
        canvas {
            display: block;
            touch-action: none;
        }
        .ui-overlay {
            position: absolute;
            top: 20px;
            left: 50%;
            transform: translateX(-50%);
            color: white;
            text-align: center;
            pointer-events: none;
            user-select: none;
            width: 100%;
        }
    </style>
</head>
<body>

    <div class="ui-overlay">
        <h1 class="text-2xl font-bold mb-2">PING PONG</h1>
        <div class="text-4xl font-mono" id="score">0 : 0</div>
        <div class="text-sm mt-2 opacity-70" id="hint">移動滑鼠或觸控螢幕來控制 | 擊球會加速</div>
    </div>

    <canvas id="gameCanvas"></canvas>

    <script>
        /**
         * 經典乒乓球遊戲 - 開源版
         * 適用於 GitHub Pages 部署
         */

        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreElement = document.getElementById('score');

        // 遊戲核心配置
        const state = {
            playerScore: 0,
            aiScore: 0,
            paddleWidth: 15,
            paddleHeight: 100,
            ballRadius: 8,
            gameRunning: true,
            initialBallSpeed: 7
        };

        const player = {
            x: 0,
            y: 0
        };

        const ai = {
            x: 0,
            y: 0,
            speed: 0.08 // AI 反應靈敏度 (0-1)
        };

        const ball = {
            x: 0,
            y: 0,
            dx: 0,
            dy: 0,
            speed: state.initialBallSpeed
        };

        // 調整畫布大小並重新初始化位置
        function resize() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;

            player.x = 20;
            player.y = canvas.height / 2 - state.paddleHeight / 2;

            ai.x = canvas.width - 20 - state.paddleWidth;
            ai.y = canvas.height / 2 - state.paddleHeight / 2;

            resetBall();
        }

        // 重置球的位置與速度
        function resetBall() {
            ball.x = canvas.width / 2;
            ball.y = canvas.height / 2;
            ball.speed = state.initialBallSpeed;
            
            // 隨機反彈角度
            const angle = (Math.random() - 0.5) * Math.PI / 2; 
            const direction = Math.random() > 0.5 ? 1 : -1;
            ball.dx = direction * ball.speed * Math.cos(angle);
            ball.dy = ball.speed * Math.sin(angle);
        }

        // 監聽視窗縮放
        window.addEventListener('resize', resize);
        
        // 處理輸入 (滑鼠與觸控)
        function handleInput(e) {
            let clientY;
            if (e.type === 'mousemove') {
                clientY = e.clientY;
            } else if (e.type === 'touchmove') {
                clientY = e.touches[0].clientY;
                e.preventDefault(); // 防止行動端滾動
            }
            
            if (clientY !== undefined) {
                player.y = clientY - state.paddleHeight / 2;
                
                // 邊界限制
                if (player.y < 0) player.y = 0;
                if (player.y > canvas.height - state.paddleHeight) player.y = canvas.height - state.paddleHeight;
            }
        }

        window.addEventListener('mousemove', handleInput);
        window.addEventListener('touchmove', handleInput, { passive: false });

        // 核心邏輯更新
        function update() {
            // 球移動
            ball.x += ball.dx;
            ball.y += ball.dy;

            // 牆壁碰撞 (上/下)
            if (ball.y - state.ballRadius < 0 || ball.y + state.ballRadius > canvas.height) {
                ball.dy *= -1;
            }

            // AI 簡單跟隨邏輯
            const aiCenter = ai.y + state.paddleHeight / 2;
            if (aiCenter < ball.y - 35) {
                ai.y += (ball.y - aiCenter) * ai.speed;
            } else if (aiCenter > ball.y + 35) {
                ai.y -= (aiCenter - ball.y) * ai.speed;
            }

            // AI 邊界檢查
            if (ai.y < 0) ai.y = 0;
            if (ai.y > canvas.height - state.paddleHeight) ai.y = canvas.height - state.paddleHeight;

            // 碰撞檢測 (判斷球是在左半邊還是右半邊)
            let paddle = (ball.x < canvas.width / 2) ? player : ai;

            if (checkCollision(ball, paddle)) {
                // 計算碰撞點相對位置以改變角度
                let collidePoint = (ball.y - (paddle.y + state.paddleHeight / 2));
                collidePoint = collidePoint / (state.paddleHeight / 2);
                let angleRad = (Math.PI / 4) * collidePoint;

                let direction = (ball.x < canvas.width / 2) ? 1 : -1;
                ball.dx = direction * ball.speed * Math.cos(angleRad);
                ball.dy = ball.speed * Math.sin(angleRad);
                
                // 每次碰撞加速增加難度
                ball.speed += 0.3;
            }

            // 得分判定
            if (ball.x - state.ballRadius < 0) {
                state.aiScore++;
                updateScoreUI();
                resetBall();
            } else if (ball.x + state.ballRadius > canvas.width) {
                state.playerScore++;
                updateScoreUI();
                resetBall();
            }
        }

        // 矩形碰撞檢測
        function checkCollision(b, p) {
            return b.x + state.ballRadius > p.x && 
                   b.x - state.ballRadius < p.x + state.paddleWidth && 
                   b.y + state.ballRadius > p.y && 
                   b.y - state.ballRadius < p.y + state.paddleHeight;
        }

        // 更新 UI 分數
        function updateScoreUI() {
            scoreElement.innerText = `${state.playerScore} : ${state.aiScore}`;
        }

        // 渲染畫面
        function draw() {
            // 清除畫布
            ctx.fillStyle = '#1a1a1a';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // 繪製球網線
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.1)';
            ctx.setLineDash([10, 10]);
            ctx.beginPath();
            ctx.moveTo(canvas.width / 2, 0);
            ctx.lineTo(canvas.width / 2, canvas.height);
            ctx.stroke();
            ctx.setLineDash([]);

            // 繪製玩家 (綠色)
            ctx.fillStyle = '#4ade80'; 
            drawRoundedRect(player.x, player.y, state.paddleWidth, state.paddleHeight, 5);

            // 繪製 AI (紅色)
            ctx.fillStyle = '#f87171';
            drawRoundedRect(ai.x, ai.y, state.paddleWidth, state.paddleHeight, 5);

            // 繪製球
            ctx.fillStyle = '#ffffff';
            ctx.beginPath();
            ctx.arc(ball.x, ball.y, state.ballRadius, 0, Math.PI * 2);
            ctx.fill();
        }

        // 輔助函式：繪製圓角矩形
        function drawRoundedRect(x, y, w, h, r) {
            ctx.beginPath();
            if (ctx.roundRect) {
                ctx.roundRect(x, y, w, h, r);
            } else {
                // 舊版瀏覽器兼容
                ctx.rect(x, y, w, h);
            }
            ctx.fill();
        }

        // 遊戲主循環
        function gameLoop() {
            update();
            draw();
            requestAnimationFrame(gameLoop);
        }

        // 初始化
        window.onload = () => {
            resize();
            gameLoop();
        };
    </script>
</body>
</html>
