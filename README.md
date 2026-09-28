[ブロック崩しindex.html](https://github.com/user-attachments/files/32747382/index.html)
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Block Breaker Deluxe</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Outfit:wght@400;600;800&display=swap');
        
        body {
            font-family: 'Outfit', sans-serif;
            background: #0f172a;
            color: #f8fafc;
            touch-action: manipulation;
            overflow: hidden;
        }

        #game-container {
            position: relative;
            box-shadow: 0 0 50px rgba(56, 189, 248, 0.15);
        }

        canvas {
            background: radial-gradient(circle at center, #1e293b 0%, #0f172a 100%);
            display: block;
        }

        .glass-panel {
            background: rgba(30, 41, 59, 0.7);
            backdrop-filter: blur(12px);
            border: 1px solid rgba(255, 255, 255, 0.1);
        }

        .neon-text-blue {
            text-shadow: 0 0 10px rgba(56, 189, 248, 0.6);
        }

        .btn-action {
            background: linear-gradient(135deg, #3b82f6 0%, #1d4ed8 100%);
            transition: all 0.2s ease;
        }

        .btn-action:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 20px rgba(59, 130, 246, 0.4);
        }

        .btn-action:active {
            transform: translateY(1px);
        }
    </style>
</head>
<body class="flex flex-col items-center justify-center min-h-screen p-2 sm:p-4 select-none">

    <div class="w-full max-w-2xl flex flex-col items-center">
        <!-- Top Stats Bar -->
        <div class="w-full glass-panel rounded-t-2xl p-4 flex justify-between items-center mb-1">
            <div class="flex items-center gap-6">
                <div>
                    <div class="text-xs uppercase tracking-wider text-slate-400 font-semibold">Score</div>
                    <div id="score" class="text-2xl sm:text-3xl font-extrabold text-cyan-400 neon-text-blue">0</div>
                </div>
                <div>
                    <div class="text-xs uppercase tracking-wider text-slate-400 font-semibold">High Score</div>
                    <div id="high-score" class="text-xl sm:text-2xl font-bold text-slate-300">0</div>
                </div>
            </div>

            <div class="flex items-center gap-6">
                <div class="text-right">
                    <div class="text-xs uppercase tracking-wider text-slate-400 font-semibold">STAGE</div>
                    <div id="stage" class="text-2xl sm:text-3xl font-extrabold text-amber-400">1</div>
                </div>
                <div class="text-right">
                    <div class="text-xs uppercase tracking-wider text-slate-400 font-semibold">LIVES</div>
                    <div id="lives" class="text-2xl sm:text-3xl font-extrabold text-rose-500 flex gap-1">
                        ❤️❤️❤️
                    </div>
                </div>
            </div>
        </div>

        <!-- Main Game Area -->
        <div id="game-container" class="relative w-full rounded-b-2xl overflow-hidden border border-slate-700/50">
            <canvas id="gameCanvas" class="w-full h-auto"></canvas>

            <!-- Overlay Screen (Start / Game Over / Victory) -->
            <div id="overlay" class="absolute inset-0 glass-panel flex flex-col items-center justify-center p-6 text-center z-10">
                <h1 id="overlay-title" class="text-4xl sm:text-5xl font-black mb-3 text-transparent bg-clip-text bg-gradient-to-r from-cyan-400 to-blue-600 tracking-wider">
                    BLOCK BREAKER
                </h1>
                <p id="overlay-msg" class="text-slate-300 mb-8 max-w-xs text-sm sm:text-base">
                    画面ドラッグ/マウス移動で操作。<br>パドルでボールを打ち返してブロックを全て破壊しよう！
                </p>
                <button id="start-btn" class="btn-action text-white font-bold px-8 py-4 rounded-xl text-lg shadow-lg flex items-center gap-3">
                    <i class="fa-solid fa-play"></i> GAME START
                </button>
            </div>
        </div>

        <!-- Controls / Audio Toggle -->
        <div class="w-full mt-3 flex justify-between items-center text-xs text-slate-400 px-2">
            <div class="flex gap-4">
                <span><i class="fa-solid fa-arrows-left-right mr-1"></i> マウス / タッチ対応</span>
                <span><i class="fa-solid fa-bolt mr-1"></i> パワーアップ有</span>
            </div>
            <button id="sound-btn" class="hover:text-white transition flex items-center gap-1">
                <i id="sound-icon" class="fa-solid fa-volume-high text-cyan-400"></i>
                <span id="sound-status">SOUND ON</span>
            </button>
        </div>
    </div>

    <script>
        // Web Audio API Sound Synthesizer (No external audio files required)
        class SoundSystem {
            constructor() {
                this.ctx = null;
                this.muted = false;
            }

            init() {
                if (!this.ctx) {
                    this.ctx = new (window.AudioContext || window.webkitAudioContext)();
                }
            }

            playHit(freq = 400, duration = 0.05, type = 'sine') {
                if (this.muted || !this.ctx) return;
                try {
                    const osc = this.ctx.createOscillator();
                    const gain = this.ctx.createGain();
                    osc.type = type;
                    osc.frequency.setValueAtTime(freq, this.ctx.currentTime);
                    gain.gain.setValueAtTime(0.3, this.ctx.currentTime);
                    gain.gain.exponentialRampToValueAtTime(0.01, this.ctx.currentTime + duration);
                    osc.connect(gain);
                    gain.connect(this.ctx.destination);
                    osc.start();
                    osc.stop(this.ctx.currentTime + duration);
                } catch (e) {}
            }

            playBrickHit(hp) {
                const freqs = [300, 450, 600, 750, 900];
                this.playHit(freqs[Math.min(hp, freqs.length - 1)], 0.08, 'triangle');
            }

            playPowerup() {
                if (this.muted || !this.ctx) return;
                const now = this.ctx.currentTime;
                const osc = this.ctx.createOscillator();
                const gain = this.ctx.createGain();
                osc.type = 'sine';
                osc.frequency.setValueAtTime(400, now);
                osc.frequency.exponentialRampToValueAtTime(800, now + 0.15);
                gain.gain.setValueAtTime(0.3, now);
                gain.gain.linearRampToValueAtTime(0.01, now + 0.15);
                osc.connect(gain);
                gain.connect(this.ctx.destination);
                osc.start();
                osc.stop(now + 0.15);
            }

            playLaser() {
                if (this.muted || !this.ctx) return;
                const now = this.ctx.currentTime;
                const osc = this.ctx.createOscillator();
                const gain = this.ctx.createGain();
                osc.type = 'sawtooth';
                osc.frequency.setValueAtTime(800, now);
                osc.frequency.exponentialRampToValueAtTime(150, now + 0.1);
                gain.gain.setValueAtTime(0.2, now);
                gain.gain.linearRampToValueAtTime(0.01, now + 0.1);
                osc.connect(gain);
                gain.connect(this.ctx.destination);
                osc.start();
                osc.stop(now + 0.1);
            }

            playLose() {
                if (this.muted || !this.ctx) return;
                const now = this.ctx.currentTime;
                const osc = this.ctx.createOscillator();
                const gain = this.ctx.createGain();
                osc.type = 'sawtooth';
                osc.frequency.setValueAtTime(300, now);
                osc.frequency.linearRampToValueAtTime(100, now + 0.4);
                gain.gain.setValueAtTime(0.4, now);
                gain.gain.linearRampToValueAtTime(0.01, now + 0.4);
                osc.connect(gain);
                gain.connect(this.ctx.destination);
                osc.start();
                osc.stop(now + 0.4);
            }
        }

        const sounds = new SoundSystem();

        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        // Internal Game Resolution
        const GAME_WIDTH = 600;
        const GAME_HEIGHT = 700;
        canvas.width = GAME_WIDTH;
        canvas.height = GAME_HEIGHT;

        // Game Entities
        let paddle, balls, bricks, powerups, lasers, particles;
        let score = 0;
        let highScore = localStorage.getItem('blockbreaker_high') || 0;
        let lives = 3;
        let stage = 1;
        let gameState = 'START'; // START, PLAYING, GAMEOVER, CLEAR
        let laserTimer = 0;

        document.getElementById('high-score').textContent = highScore;

        // Colors palette for bricks
        const BRICK_COLORS = [
            { bg: '#ef4444', border: '#f87171', pts: 50 },  // Red
            { bg: '#f97316', border: '#fb923c', pts: 40 },  // Orange
            { bg: '#eab308', border: '#fde047', pts: 30 },  // Yellow
            { bg: '#22c55e', border: '#4ade80', pts: 20 },  // Green
            { bg: '#06b6d4', border: '#38bdf8', pts: 10 }   // Cyan
        ];

        class Paddle {
            constructor() {
                this.width = 100;
                this.height = 14;
                this.x = (GAME_WIDTH - this.width) / 2;
                this.y = GAME_HEIGHT - 40;
                this.speed = 0;
                this.targetX = this.x;
                this.hasLaser = false;
            }

            update() {
                // Smooth movement towards mouse/touch target
                this.x += (this.targetX - this.x) * 0.3;
                // Clamp within screen bounds
                if (this.x < 0) this.x = 0;
                if (this.x + this.width > GAME_WIDTH) this.x = GAME_WIDTH - this.width;
            }

            draw() {
                // Gradient styling
                const grad = ctx.createLinearGradient(this.x, this.y, this.x, this.y + this.height);
                grad.addColorStop(0, '#38bdf8');
                grad.addColorStop(1, '#0284c7');

                ctx.save();
                ctx.fillStyle = grad;
                ctx.shadowColor = '#38bdf8';
                ctx.shadowBlur = this.hasLaser ? 15 : 8;
                
                ctx.beginPath();
                ctx.roundRect(this.x, this.y, this.width, this.height, 8);
                ctx.fill();

                // Draw lasers cannons if laser active
                if (this.hasLaser) {
                    ctx.fillStyle = '#ef4444';
                    ctx.fillRect(this.x, this.y - 6, 6, 8);
                    ctx.fillRect(this.x + this.width - 6, this.y - 6, 6, 8);
                }

                ctx.restore();
            }
        }

        class Ball {
            constructor(x, y, vx, vy) {
                this.x = x || GAME_WIDTH / 2;
                this.y = y || GAME_HEIGHT - 60;
                this.radius = 7;
                this.speed = 6.5;
                this.vx = vx || (Math.random() - 0.5) * 4;
                this.vy = vy || -this.speed;
                this.stuck = false;
            }

            update() {
                if (this.stuck) {
                    this.x = paddle.x + paddle.width / 2;
                    this.y = paddle.y - this.radius;
                    return;
                }

                this.x += this.vx;
                this.y += this.vy;

                // Wall Collisions
                if (this.x - this.radius <= 0) {
                    this.x = this.radius;
                    this.vx *= -1;
                    sounds.playHit(200);
                } else if (this.x + this.radius >= GAME_WIDTH) {
                    this.x = GAME_WIDTH - this.radius;
                    this.vx *= -1;
                    sounds.playHit(200);
                }

                if (this.y - this.radius <= 0) {
                    this.y = this.radius;
                    this.vy *= -1;
                    sounds.playHit(200);
                }
            }

            draw() {
                ctx.save();
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
                ctx.fillStyle = '#ffffff';
                ctx.shadowColor = '#e0f2fe';
                ctx.shadowBlur = 10;
                ctx.fill();
                ctx.restore();
            }
        }

        class Brick {
            constructor(x, y, width, height, type) {
                this.x = x;
                this.y = y;
                this.width = width;
                this.height = height;
                this.type = type; // 0 to 4
                this.hp = type === 0 ? 2 : 1; // Top red row takes 2 hits
                this.maxHp = this.hp;
                this.alive = true;
            }

            draw() {
                if (!this.alive) return;
                const style = BRICK_COLORS[this.type];

                ctx.save();
                ctx.fillStyle = style.bg;
                ctx.strokeStyle = style.border;
                ctx.lineWidth = 2;

                if (this.hp < this.maxHp) {
                    ctx.globalAlpha = 0.6; // Visual indicator for damaged brick
                }

                ctx.beginPath();
                ctx.roundRect(this.x, this.y, this.width, this.height, 4);
                ctx.fill();
                ctx.stroke();

                // Inner highlight
                ctx.fillStyle = 'rgba(255,255,255,0.2)';
                ctx.fillRect(this.x + 2, this.y + 2, this.width - 4, this.height / 3);

                ctx.restore();
            }
        }

        class Powerup {
            constructor(x, y, type) {
                this.x = x;
                this.y = y;
                this.width = 24;
                this.height = 24;
                this.vy = 2;
                this.type = type; // 'ENLARGE', 'MULTIBALL', 'LASER', 'SLOW'
                this.alive = true;
            }

            update() {
                this.y += this.vy;
                if (this.y > GAME_HEIGHT) this.alive = false;
            }

            draw() {
                if (!this.alive) return;
                ctx.save();
                
                let icon = '⚡';
                let color = '#eab308';
                if (this.type === 'ENLARGE') { icon = '↔️'; color = '#3b82f6'; }
                if (this.type === 'MULTIBALL') { icon = '⚽'; color = '#10b981'; }
                if (this.type === 'LASER') { icon = '🔫'; color = '#ef4444'; }
                if (this.type === 'SLOW') { icon = '🐢'; color = '#a855f7'; }

                ctx.fillStyle = color;
                ctx.beginPath();
                ctx.arc(this.x + 12, this.y + 12, 12, 0, Math.PI * 2);
                ctx.fill();

                ctx.font = '12px sans-serif';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.fillText(icon, this.x + 12, this.y + 12);

                ctx.restore();
            }
        }

        class Particle {
            constructor(x, y, color) {
                this.x = x;
                this.y = y;
                this.vx = (Math.random() - 0.5) * 6;
                this.vy = (Math.random() - 0.5) * 6;
                this.radius = Math.random() * 3 + 1;
                this.color = color;
                this.alpha = 1;
                this.decay = Math.random() * 0.03 + 0.015;
            }

            update() {
                this.x += this.vx;
                this.y += this.vy;
                this.alpha -= this.decay;
            }

            draw() {
                ctx.save();
                ctx.globalAlpha = Math.max(0, this.alpha);
                ctx.fillStyle = this.color;
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
                ctx.fill();
                ctx.restore();
            }
        }

        class Laser {
            constructor(x, y) {
                this.x = x;
                this.y = y;
                this.vy = -12;
                this.width = 4;
                this.height = 12;
                this.alive = true;
            }

            update() {
                this.y += this.vy;
                if (this.y < 0) this.alive = false;
            }

            draw() {
                if (!this.alive) return;
                ctx.save();
                ctx.fillStyle = '#ef4444';
                ctx.shadowColor = '#f87171';
                ctx.shadowBlur = 8;
                ctx.fillRect(this.x - this.width / 2, this.y, this.width, this.height);
                ctx.restore();
            }
        }

        function initStage(stageNum) {
            paddle = new Paddle();
            balls = [new Ball()];
            powerups = [];
            lasers = [];
            particles = [];
            laserTimer = 0;

            // Generate Bricks
            bricks = [];
            const rows = Math.min(5 + stageNum, 8);
            const cols = 8;
            const padding = 6;
            const offsetTop = 60;
            const offsetLeft = 15;
            const brickWidth = (GAME_WIDTH - (offsetLeft * 2) - (padding * (cols - 1))) / cols;
            const brickHeight = 22;

            for (let r = 0; r < rows; r++) {
                for (let c = 0; c < cols; c++) {
                    const type = r % BRICK_COLORS.length;
                    const x = offsetLeft + c * (brickWidth + padding);
                    const y = offsetTop + r * (brickHeight + padding);
                    bricks.push(new Brick(x, y, brickWidth, brickHeight, type));
                }
            }

            updateUI();
        }

        function createParticles(x, y, color, count = 8) {
            for (let i = 0; i < count; i++) {
                particles.push(new Particle(x, y, color));
            }
        }

        function spawnPowerup(x, y) {
            if (Math.random() < 0.25) { // 25% chance
                const types = ['ENLARGE', 'MULTIBALL', 'LASER', 'SLOW'];
                const type = types[Math.floor(Math.random() * types.length)];
                powerups.push(new Powerup(x, y, type));
            }
        }

        function handleCollisions() {
            // Ball - Paddle Collision
            balls.forEach(ball => {
                if (ball.y + ball.radius >= paddle.y &&
                    ball.y - ball.radius <= paddle.y + paddle.height &&
                    ball.x >= paddle.x && ball.x <= paddle.x + paddle.width && ball.vy > 0) {
                    
                    sounds.playHit(300);
                    ball.vy = -Math.abs(ball.vy);

                    // Angle calculation based on hit location
                    const hitPoint = (ball.x - (paddle.x + paddle.width / 2)) / (paddle.width / 2);
                    ball.vx = hitPoint * (ball.speed * 0.8);
                }

                // Ball - Brick Collision
                bricks.forEach(brick => {
                    if (!brick.alive) return;

                    if (ball.x + ball.radius > brick.x &&
                        ball.x - ball.radius < brick.x + brick.width &&
                        ball.y + ball.radius > brick.y &&
                        ball.y - ball.radius < brick.y + brick.height) {

                        brick.hp--;
                        sounds.playBrickHit(brick.type);
                        createParticles(ball.x, ball.y, BRICK_COLORS[brick.type].bg, 6);

                        if (brick.hp <= 0) {
                            brick.alive = false;
                            score += BRICK_COLORS[brick.type].pts;
                            spawnPowerup(brick.x + brick.width / 2, brick.y + brick.height / 2);
                        }

                        // Bounce logic
                        const prevX = ball.x - ball.vx;
                        const prevY = ball.y - ball.vy;

                        if (prevX <= brick.x || prevX >= brick.x + brick.width) {
                            ball.vx *= -1;
                        } else {
                            ball.vy *= -1;
                        }

                        updateUI();
                    }
                });
            });

            // Laser - Brick Collision
            lasers.forEach(laser => {
                if (!laser.alive) return;
                bricks.forEach(brick => {
                    if (!brick.alive) return;
                    if (laser.x >= brick.x && laser.x <= brick.x + brick.width &&
                        laser.y >= brick.y && laser.y <= brick.y + brick.height) {
                        
                        laser.alive = false;
                        brick.hp--;
                        sounds.playBrickHit(brick.type);
                        createParticles(laser.x, laser.y, '#ef4444', 4);

                        if (brick.hp <= 0) {
                            brick.alive = false;
                            score += BRICK_COLORS[brick.type].pts;
                            spawnPowerup(brick.x + brick.width / 2, brick.y + brick.height / 2);
                        }
                        updateUI();
                    }
                });
            });

            // Paddle - Powerup Collision
            powerups.forEach(p => {
                if (!p.alive) return;
                if (p.x + p.width >= paddle.x && p.x <= paddle.x + paddle.width &&
                    p.y + p.height >= paddle.y && p.y <= paddle.y + paddle.height) {
                    
                    p.alive = false;
                    sounds.playPowerup();

                    if (p.type === 'ENLARGE') {
                        paddle.width = Math.min(paddle.width + 30, 180);
                    } else if (p.type === 'MULTIBALL') {
                        const baseBall = balls[0] || new Ball();
                        balls.push(new Ball(baseBall.x, baseBall.y, -3, -5));
                        balls.push(new Ball(baseBall.x, baseBall.y, 3, -5));
                    } else if (p.type === 'LASER') {
                        paddle.hasLaser = true;
                        laserTimer = 400; // frames
                    } else if (p.type === 'SLOW') {
                        balls.forEach(b => {
                            b.vx *= 0.7;
                            b.vy *= 0.7;
                        });
                    }
                }
            });
        }

        function update() {
            if (gameState !== 'PLAYING') return;

            paddle.update();

            // Laser Firing Logic
            if (paddle.hasLaser) {
                laserTimer--;
                if (laserTimer % 20 === 0) {
                    lasers.push(new Laser(paddle.x + 4, paddle.y));
                    lasers.push(new Laser(paddle.x + paddle.width - 4, paddle.y));
                    sounds.playLaser();
                }
                if (laserTimer <= 0) paddle.hasLaser = false;
            }

            // Update Lasers
            lasers.forEach(l => l.update());
            lasers = lasers.filter(l => l.alive);

            // Update Balls
            balls.forEach(b => b.update());

            // Check Ball Out of Bounds
            balls = balls.filter(b => b.y - b.radius < GAME_HEIGHT);

            if (balls.length === 0) {
                lives--;
                sounds.playLose();
                updateUI();

                if (lives <= 0) {
                    endGame(false);
                } else {
                    balls.push(new Ball());
                }
            }

            // Update Powerups & Particles
            powerups.forEach(p => p.update());
            powerups = powerups.filter(p => p.alive);

            particles.forEach(pt => pt.update());
            particles = particles.filter(pt => pt.alpha > 0);

            handleCollisions();

            // Check Stage Clear
            if (bricks.every(b => !b.alive)) {
                stage++;
                sounds.playPowerup();
                if (stage > 5) {
                    endGame(true);
                } else {
                    initStage(stage);
                }
            }
        }

        function draw() {
            ctx.clearRect(0, 0, GAME_WIDTH, GAME_HEIGHT);

            // Draw game components
            bricks.forEach(b => b.draw());
            powerups.forEach(p => p.draw());
            lasers.forEach(l => l.draw());
            particles.forEach(pt => pt.draw());
            paddle.draw();
            balls.forEach(b => b.draw());
        }

        function gameLoop() {
            update();
            draw();
            requestAnimationFrame(gameLoop);
        }

        function updateUI() {
            document.getElementById('score').textContent = score;
            document.getElementById('stage').textContent = stage;
            document.getElementById('lives').textContent = '❤️'.repeat(Math.max(0, lives));

            if (score > highScore) {
                highScore = score;
                localStorage.setItem('blockbreaker_high', highScore);
                document.getElementById('high-score').textContent = highScore;
            }
        }

        function endGame(isWin) {
            gameState = isWin ? 'CLEAR' : 'GAMEOVER';
            const overlay = document.getElementById('overlay');
            const title = document.getElementById('overlay-title');
            const msg = document.getElementById('overlay-msg');
            const btn = document.getElementById('start-btn');

            overlay.classList.remove('hidden');

            if (isWin) {
                title.textContent = 'STAGE CLEAR!';
                title.className = 'text-4xl sm:text-5xl font-black mb-3 text-emerald-400 tracking-wider';
                msg.innerHTML = `おめでとうございます！全ステージ制覇！<br>Final Score: <b class="text-cyan-400">${score}</b>`;
                btn.innerHTML = '<i class="fa-solid fa-rotate-right"></i> PLAY AGAIN';
            } else {
                title.textContent = 'GAME OVER';
                title.className = 'text-4xl sm:text-5xl font-black mb-3 text-rose-500 tracking-wider';
                msg.innerHTML = `スコア: <b class="text-cyan-400">${score}</b><br>諦めずにもう一度挑戦しよう！`;
                btn.innerHTML = '<i class="fa-solid fa-rotate-right"></i> RETRY';
            }
        }

        function startGame() {
            sounds.init();
            score = 0;
            lives = 3;
            stage = 1;
            gameState = 'PLAYING';
            initStage(stage);
            document.getElementById('overlay').classList.add('hidden');
        }

        // Input Handling
        function handleMove(e) {
            const rect = canvas.getBoundingClientRect();
            const clientX = e.touches ? e.touches[0].clientX : e.clientX;
            const scale = GAME_WIDTH / rect.width;
            const canvasX = (clientX - rect.left) * scale;
            if (paddle) {
                paddle.targetX = canvasX - paddle.width / 2;
            }
        }

        window.addEventListener('mousemove', handleMove);
        window.addEventListener('touchmove', handleMove, { passive: true });

        document.getElementById('start-btn').addEventListener('click', startGame);

        // Sound Toggle
        document.getElementById('sound-btn').addEventListener('click', () => {
            sounds.muted = !sounds.muted;
            document.getElementById('sound-status').textContent = sounds.muted ? 'SOUND OFF' : 'SOUND ON';
            document.getElementById('sound-icon').className = sounds.muted ? 'fa-solid fa-volume-xmark text-slate-500' : 'fa-solid fa-volume-high text-cyan-400';
        });

        // Initialize Loop on load
        window.onload = function() {
            initStage(1);
            gameLoop();
        };
    </script>
</body>
</html>
