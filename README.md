<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>P2P Hockey: Team Selection</title>
    <script src="https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js"></script>
    <style>
        body { background: #1a1a1a; color: #fff; font-family: 'Segoe UI', sans-serif; text-align: center; margin: 0; }
        canvas { background: #eef2f7; border: 4px solid #334155; border-radius: 8px; display: none; margin: 20px auto; }
        
        .ui-container { background: #2d3748; padding: 20px; border-radius: 12px; display: inline-block; margin-top: 50px; min-width: 400px; box-shadow: 0 10px 30px rgba(0,0,0,0.5); }
        .lobby { display: flex; justify-content: space-around; margin: 20px 0; }
        .team-box { background: #1a202c; padding: 10px; border-radius: 8px; width: 180px; min-height: 150px; }
        .team-red { border-top: 5px solid #ef4444; }
        .team-blue { border-top: 5px solid #3b82f6; }
        
        button { padding: 10px 20px; font-size: 16px; border-radius: 6px; border: none; cursor: pointer; font-weight: bold; transition: 0.2s; margin: 5px; }
        .btn-join-red { background: #ef4444; color: white; }
        .btn-join-blue { background: #3b82f6; color: white; }
        .btn-start { background: #10b981; color: white; display: none; width: 100%; font-size: 20px; }
        input { padding: 10px; border-radius: 6px; border: 1px solid #4a5568; background: #1a202c; color: white; width: 200px; }
        
        .score-board { font-size: 24px; font-weight: bold; margin-bottom: 10px; display: none; }
        .status-msg { color: #a0aec0; font-size: 14px; margin-top: 10px; }
    </style>
</head>
<body>

    <div id="setupMenu" class="ui-container">
        <h2>🏒 Хоккейное Лобби</h2>
        
        <div id="preConnect">
            <button id="btnCreate" style="background: #38bdf8;">Создать игру</button>
            <p>ИЛИ</p>
            <input type="text" id="joinIdInput" placeholder="ID Хоста">
            <button id="btnJoin" style="background: #94a3b8;">Войти</button>
        </div>

        <div id="lobbyView" style="display:none;">
            <p id="displayID" style="color: #38bdf8; font-weight:bold;"></p>
            <div class="lobby">
                <div class="team-box team-red">
                    <h3 style="color:#ef4444">КРАСНЫЕ</h3>
                    <div id="listRed"></div>
                    <button class="btn-join-red" onclick="requestTeam('red')">Зайти</button>
                </div>
                <div class="team-box team-blue">
                    <h3 style="color:#3b82f6">СИНИЕ</h3>
                    <div id="listBlue"></div>
                    <button class="btn-join-blue" onclick="requestTeam('blue')">Зайти</button>
                </div>
            </div>
            <button id="btnStartGame" class="btn-start">НАЧАТЬ МАТЧ</button>
            <div class="status-msg" id="statusText">Ожидание игроков...</div>
        </div>
    </div>

    <div id="gameUI">
        <div class="score-board" id="scoreBoard">
            <span style="color:#ef4444" id="scoreRed">0</span> : <span style="color:#3b82f6" id="scoreBlue">0</span>
        </div>
        <canvas id="hockeyRink" width="1000" height="600"></canvas>
    </div>

<script>
const canvas = document.getElementById('hockeyRink');
const ctx = canvas.getContext('2d');

// --- ГЛОБАЛЬНЫЕ ПЕРЕМЕННЫЕ ---
let peer, myId, connToHost;
let connections = []; // Для хоста
let isHost = false;
let gameStarted = false;

const keys = { w: false, a: false, s: false, d: false, x: false, c: false, v: false };
const mouse = { x: 0, y: 0 };

let gameState = {
    players: {}, // id: {x, y, team, name, color...}
    puck: { x: 500, y: 300, vx: 0, vy: 0, radius: 10, holderId: null, cooldown: 0, z: 0, vz: 0 },
    score: { red: 0, blue: 0 }
};

// --- СЕТЕВАЯ ЧАСТЬ ---

// Хост: Создание
document.getElementById('btnCreate').addEventListener('click', () => {
    peer = new Peer();
    peer.on('open', (id) => {
        myId = id; isHost = true;
        initLobbyUI(id);
        addPlayerToState(id, "Хост (Вы)");
    });
    peer.on('connection', (conn) => {
        connections.push(conn);
        conn.on('data', (data) => handleNetworkData(conn, data));
        conn.on('close', () => removePlayer(conn.peer));
    });
});

// Клиент: Подключение
document.getElementById('btnJoin').addEventListener('click', () => {
    const targetId = document.getElementById('joinIdInput').value;
    if(!targetId) return;
    peer = new Peer();
    peer.on('open', (id) => {
        myId = id;
        connToHost = peer.connect(targetId);
        connToHost.on('open', () => {
            initLobbyUI(targetId);
            connToHost.send({ type: 'JOIN_LOBBY', name: "Игрок " + id.slice(0,4) });
        });
        connToHost.on('data', (data) => handleNetworkData(null, data));
    });
});

function handleNetworkData(conn, msg) {
    if (isHost) {
        if (msg.type === 'JOIN_LOBBY') addPlayerToState(conn.peer, msg.name);
        if (msg.type === 'CHANGE_TEAM') changePlayerTeam(conn.peer, msg.team);
        if (msg.type === 'INPUT') { if(gameState.players[conn.peer]) gameState.players[conn.peer].inputs = msg.data; }
    } else {
        if (msg.type === 'SYNC_LOBBY' || msg.type === 'SYNC_GAME') {
            gameState = msg.state;
            updateLobbyDisplay();
            if(msg.type === 'SYNC_GAME' && !gameStarted) startClientGame();
        }
    }
}

function initLobbyUI(id) {
    document.getElementById('preConnect').style.display = 'none';
    document.getElementById('lobbyView').style.display = 'block';
    document.getElementById('displayID').innerText = "ID КОМНАТЫ: " + id;
    if (isHost) document.getElementById('btnStartGame').style.display = 'block';
}

function addPlayerToState(id, name) {
    gameState.players[id] = {
        id, name, team: 'red', // по умолчанию за красных
        x: 0, y: 0, vx: 0, vy: 0, angle: 0,
        hasPuck: false, possessionTimer: 0,
        inputs: { keys: {}, mouse: {x:0,y:0} }
    };
    broadcastLobby();
}

function changePlayerTeam(id, team) {
    if (gameState.players[id]) {
        gameState.players[id].team = team;
        broadcastLobby();
    }
}

function requestTeam(team) {
    if (isHost) changePlayerTeam(myId, team);
    else connToHost.send({ type: 'CHANGE_TEAM', team: team });
}

function broadcastLobby() {
    if (!isHost) return;
    updateLobbyDisplay();
    connections.forEach(c => c.send({ type: 'SYNC_LOBBY', state: gameState }));
}

function updateLobbyDisplay() {
    const listRed = document.getElementById('listRed');
    const listBlue = document.getElementById('listBlue');
    listRed.innerHTML = ""; listBlue.innerHTML = "";
    
    for (let id in gameState.players) {
        const p = gameState.players[id];
        const item = document.createElement('div');
        item.innerText = p.name + (id === myId ? " (Вы)" : "");
        item.style.padding = "5px";
        if (p.team === 'red') listRed.appendChild(item);
        else listBlue.appendChild(item);
    }
}

// --- ИГРОВОЙ ПРОЦЕСС ---

document.getElementById('btnStartGame').addEventListener('click', () => {
    if (!isHost) return;
    gameStarted = true;
    resetPositions();
    startGameLoop();
});

function startClientGame() {
    gameStarted = true;
    document.getElementById('setupMenu').style.display = 'none';
    document.getElementById('scoreBoard').style.display = 'block';
    canvas.style.display = 'block';
    startGameLoop();
}

function resetPositions() {
    // Начальные позиции: красные слева, синие справа
    for (let id in gameState.players) {
        const p = gameState.players[id];
        p.vx = 0; p.vy = 0; p.hasPuck = false; p.possessionTimer = 0;
        if (p.team === 'red') { p.x = 200; p.y = 300; }
        else { p.x = 800; p.y = 300; }
    }
    gameState.puck = { x: 500, y: 300, vx: 0, vy: 0, radius: 10, holderId: null, cooldown: 60, z: 0, vz: 0 };
}

// Слушатели управления
window.addEventListener('keydown', (e) => { if(e.key.toLowerCase() in keys) keys[e.key.toLowerCase()] = true; sendInput(); });
window.addEventListener('keyup', (e) => { if(e.key.toLowerCase() in keys) keys[e.key.toLowerCase()] = false; sendInput(); });
canvas.addEventListener('mousemove', (e) => {
    const rect = canvas.getBoundingClientRect();
    mouse.x = e.clientX - rect.left; mouse.y = e.clientY - rect.top;
    sendInput();
});

function sendInput() {
    if (!gameStarted) return;
    const data = { keys: {...keys}, mouse: {...mouse} };
    if (isHost) gameState.players[myId].inputs = data;
    else if (connToHost) connToHost.send({ type: 'INPUT', data: data });
}

// --- ФИЗИКА (ТОЛЬКО ХОСТ) ---

function updatePhysics() {
    if (!isHost || !gameStarted) return;

    const puck = gameState.puck;
    
    for (let id in gameState.players) {
        const p = gameState.players[id];
        const inp = p.inputs;
        
        // Передвижение
        if (inp.keys.w) p.vy -= 0.4; if (inp.keys.s) p.vy += 0.4;
        if (inp.keys.a) p.vx -= 0.4; if (inp.keys.d) p.vx += 0.4;
        
        p.vx *= 0.96; p.vy *= 0.96;
        p.x += p.vx; p.y += p.vy;
        p.angle = Math.atan2(inp.mouse.y - p.y, inp.mouse.x - p.x);

        // Коллизии с бортами
        if (p.x < 20) p.x = 20; if (p.x > 980) p.x = 980;
        if (p.y < 20) p.y = 20; if (p.y > 580) p.y = 580;

        // Владение шайбой
        if (p.hasPuck) {
            p.possessionTimer++;
            if (p.possessionTimer > 180) releasePuck(p);
            
            // Удары
            if (inp.keys.x) shoot(p, 14, 0); // Wrist
            if (inp.keys.c) shoot(p, 24, 0); // Slap
            if (inp.keys.v) shoot(p, 10, 7); // Flip
        } else if (!puck.holderId && puck.cooldown <= 0 && puck.z < 15) {
            const dist = Math.hypot(puck.x - p.x, puck.y - p.y);
            if (dist < 35) { p.hasPuck = true; puck.holderId = id; p.possessionTimer = 0; }
        }
    }

    // Физика шайбы
    if (puck.holderId) {
        const h = gameState.players[puck.holderId];
        puck.x = h.x + Math.cos(h.angle) * 25;
        puck.y = h.y + Math.sin(h.angle) * 25;
    } else {
        puck.x += puck.vx; puck.y += puck.vy;
        puck.vx *= 0.993; puck.vy *= 0.993;
        if (puck.z > 0 || puck.vz !== 0) { puck.z += puck.vz; puck.vz -= 0.35; if(puck.z < 0) {puck.z=0; puck.vz=0;} }
        
        // Отскоки и Голы
        checkGoal(puck);
        if (puck.x < 10 || puck.x > 990) puck.vx *= -0.8;
        if (puck.y < 10 || puck.y > 590) puck.vy *= -0.8;
    }
    if (puck.cooldown > 0) puck.cooldown--;

    connections.forEach(c => c.send({ type: 'SYNC_GAME', state: gameState }));
}

function shoot(p, pwr, vz) {
    const puck = gameState.puck;
    p.hasPuck = false; puck.holderId = null; puck.cooldown = 30;
    puck.vx = Math.cos(p.angle) * pwr + p.vx * 0.5;
    puck.vy = Math.sin(p.angle) * pwr + p.vy * 0.5;
    puck.vz = vz;
}

function releasePuck(p) {
    const puck = gameState.puck;
    p.hasPuck = false; puck.holderId = null; puck.cooldown = 40;
    puck.vx = Math.cos(p.angle) * 4; puck.vy = Math.sin(p.angle) * 4;
}

function checkGoal(puck) {
    // Ворота Синих (справа) - забивают Красные
    if (puck.x > 980 && puck.y > 230 && puck.y < 370) {
        gameState.score.red++; resetPositions();
    }
    // Ворота Красных (слева) - забивают Синие
    if (puck.x < 20 && puck.y > 230 && puck.y < 370) {
        gameState.score.blue++; resetPositions();
    }
}

// --- РЕНДЕРИНГ ---

function draw() {
    if (!gameStarted) return;
    document.getElementById('setupMenu').style.display = 'none';
    document.getElementById('gameUI').style.display = 'block';
    document.getElementById('scoreRed').innerText = gameState.score.red;
    document.getElementById('scoreBlue').innerText = gameState.score.blue;

    ctx.clearRect(0, 0, 1000, 600);
    drawRink();

    for (let id in gameState.players) {
        const p = gameState.players[id];
        ctx.save();
        ctx.translate(p.x, p.y);
        ctx.rotate(p.angle);
        // Клюшка
        ctx.fillStyle = p.hasPuck ? '#facc15' : '#475569';
        ctx.fillRect(18, -4, 15, 8);
        // Игрок
        ctx.fillStyle = (p.team === 'red') ? '#ef4444' : '#3b82f6';
        ctx.beginPath(); ctx.arc(0, 0, 20, 0, Math.PI*2); ctx.fill();
        ctx.strokeStyle = (id === myId) ? 'gold' : 'white'; ctx.lineWidth = 3; ctx.stroke();
        ctx.restore();
    }

    const pk = gameState.puck;
    ctx.fillStyle = '#000';
    ctx.beginPath(); ctx.arc(pk.x, pk.y - pk.z, pk.radius + pk.z*0.3, 0, Math.PI*2); ctx.fill();
}

function drawRink() {
    ctx.strokeStyle = '#cbd5e1'; ctx.lineWidth = 2;
    ctx.strokeRect(0, 0, 1000, 600);
    // Ворота
    ctx.fillStyle = '#ef4444'; ctx.fillRect(0, 230, 15, 140);
    ctx.fillStyle = '#3b82f6'; ctx.fillRect(985, 230, 15, 140);
    // Линии
    ctx.beginPath(); ctx.moveTo(500, 0); ctx.lineTo(500, 600); ctx.stroke();
    ctx.beginPath(); ctx.arc(500, 300, 80, 0, Math.PI*2); ctx.stroke();
}

function startGameLoop() {
    function frame() {
        updatePhysics();
        draw();
        requestAnimationFrame(frame);
    }
    requestAnimationFrame(frame);
}
</script>
</body>
</html>
