<!DOCTYPE html>
<html lang="pt-PT">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Fortnite PC Edition - Full Multiplayer</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body, html { width: 100%; height: 100%; overflow: hidden; background: #000; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; user-select: none; }
        canvas { display: block; }

        /* Overlay de Início */
        #start-overlay {
            position: fixed; inset: 0;
            background: linear-gradient(135deg, #4c1d95 0%, #1e1b4b 100%);
            color: white; display: flex; flex-direction: column;
            align-items: center; justify-content: center; z-index: 9999; text-align: center;
        }

        #lobby-container {
            background: rgba(255, 255, 255, 0.1); padding: 40px; border-radius: 20px;
            margin-top: 20px; width: 450px; backdrop-filter: blur(15px);
            border: 1px solid rgba(255,255,255,0.2); box-shadow: 0 10px 40px rgba(0,0,0,0.6);
        }

        input {
            width: 100%; padding: 15px; margin: 15px 0; border-radius: 10px;
            border: 2px solid rgba(255,255,255,0.2); font-size: 18px; text-align: center;
            background: rgba(0,0,0,0.4); color: #fcd34d; text-transform: uppercase; font-weight: bold;
        }

        .btn-play {
            background: linear-gradient(to right, #f59e0b, #d97706);
            color: white; border: none; padding: 20px; font-weight: 900;
            font-size: 22px; border-radius: 12px; cursor: pointer; width: 100%;
            text-transform: uppercase; letter-spacing: 2px; transition: transform 0.2s;
        }
        .btn-play:hover { transform: scale(1.03); filter: brightness(1.1); }
        .btn-play:disabled { background: #444; cursor: not-allowed; opacity: 0.7; }

        /* Interface de Jogo */
        #ui-layer { position: fixed; inset: 0; pointer-events: none; display: none; z-index: 10; }
        
        #room-info { 
            position: absolute; top: 25px; left: 25px; 
            background: rgba(0,0,0,0.7); color: #fff; padding: 12px 20px; 
            border-radius: 8px; border-left: 5px solid #fcd34d; font-family: monospace;
        }

        #safe-timer-container {
            position: absolute; top: 25px; left: 50%; transform: translateX(-50%);
            background: rgba(0, 0, 0, 0.8); padding: 12px 30px; border-radius: 15px;
            border: 2px solid #3b82f6; color: white; text-align: center; min-width: 200px;
        }
        #safe-clock { font-size: 32px; font-weight: 900; font-family: 'Courier New', Courier, monospace; }
        #safe-phase { font-size: 14px; color: #93c5fd; text-transform: uppercase; letter-spacing: 1px; }

        #storm-warning {
            position: fixed; top: 140px; left: 50%; transform: translateX(-50%);
            color: #ef4444; font-weight: 900; font-size: 36px;
            text-shadow: 0 0 15px rgba(0,0,0,0.9); display: none;
            animation: pulse 0.8s infinite;
        }

        /* Aviso de Interação */
        #interaction-prompt {
            position: fixed; bottom: 200px; left: 50%; transform: translateX(-50%);
            background: rgba(0,0,0,0.8); color: white; padding: 10px 20px;
            border-radius: 5px; font-weight: bold; border: 1px solid #fcd34d;
            display: none; z-index: 100;
        }

        @keyframes pulse { 0% { opacity: 1; transform: translateX(-50%) scale(1); } 50% { opacity: 0.6; transform: translateX(-50%) scale(1.05); } 100% { opacity: 1; transform: translateX(-50%) scale(1); } }

        #stats-container { 
            position: absolute; bottom: 40px; left: 50%; transform: translateX(-50%); 
            display: flex; align-items: flex-end; gap: 25px; pointer-events: auto; 
        }
        
        #hp-container { 
            width: 300px; height: 35px; background: rgba(0,0,0,0.7); 
            border: 2px solid rgba(255,255,255,0.4); border-radius: 6px; overflow: hidden; position: relative; 
        }
        #hp-bar-fill { width: 100%; height: 100%; background: linear-gradient(to right, #22c55e, #4ade80); transition: width 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275); }
        #hp-text { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; color: white; font-weight: 900; text-shadow: 1px 1px 2px #000; font-size: 18px; }
        
        .resource-box { 
            background: rgba(0,0,0,0.8); padding: 12px 25px; border-radius: 8px; 
            color: #fff; font-weight: 900; border-bottom: 5px solid #fcd34d; font-size: 20px;
        }

        #hotbar { position: absolute; bottom: 40px; right: 40px; display: flex; gap: 12px; pointer-events: auto; }
        .hotbar-slot { 
            width: 75px; height: 75px; background: rgba(0,0,0,0.7); 
            border: 3px solid #444; border-radius: 12px; display: flex; 
            align-items: center; justify-content: center; font-size: 32px; 
            cursor: pointer; transition: all 0.2s; position: relative;
        }
        .hotbar-slot::after {
            content: attr(data-key); position: absolute; top: 5px; left: 5px;
            font-size: 12px; color: #888; font-weight: bold;
        }
        .hotbar-slot.active { border-color: #fcd34d; background: rgba(252, 211, 77, 0.2); transform: translateY(-10px); box-shadow: 0 0 20px rgba(252, 211, 77, 0.4); }

        #crosshair { 
            position: fixed; top: 50%; left: 50%; width: 24px; height: 24px; 
            transform: translate(-50%, -50%); pointer-events: none; z-index: 5; display: none; 
        }
        #crosshair::before { content: ''; position: absolute; background: #fff; top: 50%; left: 0; width: 100%; height: 2px; }
        #crosshair::after { content: ''; position: absolute; background: #fff; left: 50%; top: 0; height: 100%; width: 2px; }

        #debug-log { position: fixed; top: 120px; left: 25px; color: #0f0; font-family: monospace; font-size: 11px; pointer-events: none; background: rgba(0,0,0,0.6); padding: 8px; border-radius: 4px; }
        .flash { position: fixed; inset: 0; background: white; opacity: 0.2; pointer-events: none; z-index: 100; }

        #kill-feed {
            position: fixed; top: 25px; right: 25px; width: 250px;
            display: flex; flex-direction: column; gap: 5px; pointer-events: none;
        }
        .kill-item { background: rgba(0,0,0,0.6); color: white; padding: 8px; border-radius: 4px; font-size: 13px; border-right: 4px solid #ef4444; animation: slideIn 0.3s forwards; }
        @keyframes slideIn { from { transform: translateX(100%); } to { transform: translateX(0); } }
    </style>
</head>
<body>
    <div id="start-overlay">
        <h1 style="font-size: 5rem; font-weight: 900; font-style: italic; letter-spacing: -2px; margin-bottom: 0;">FORTNITE</h1>
        <p style="color: #a78bfa; font-weight: bold; margin-bottom: 20px;">PC EDITION MULTIPLAYER</p>
        <div id="lobby-container">
            <h3 id="my-id-display" style="color: #fcd34d; font-family: monospace; margin-bottom: 20px; font-size: 20px;">A LIGAR AO SERVIDOR...</h3>
            <input type="text" id="room-input" placeholder="NOME DA SALA (EX: PROS)">
            <button class="btn-play" id="play-button" onclick="joinGame()" disabled>AGUARDE LIGAÇÃO...</button>
            <p style="font-size: 12px; color: #94a3b8; margin-top: 20px;">Controlos: WASD (Mover) | SHIFT (Correr) | ESPAÇO (Saltar) | G (Consumir) | 1, 2, 3 (Itens)</p>
        </div>
    </div>
    
    <div id="crosshair"></div>
    <div id="debug-log">Inicializando motor...</div>
    <div id="kill-feed"></div>
    <div id="interaction-prompt">Pressiona [G] para comer cogumelo 🍄</div>
    
    <div id="ui-layer">
        <div id="room-info">
            SALA: <span id="current-room-id" style="color: #fcd34d;">---</span><br>
            PAREDES: <span id="wall-count">0</span> | JOGADORES: <span id="player-count">1</span>
        </div>

        <div id="safe-timer-container">
            <div id="safe-phase">Safe Zone - Fase 1</div>
            <div id="safe-clock">05:00</div>
        </div>

        <div id="storm-warning">⚠️ ESTÁS NA TEMPESTADE! ⚠️</div>

        <div id="stats-container">
            <div class="resource-box">🪵 <span id="wood-val">50</span></div>
            <div id="hp-container">
                <div id="hp-bar-fill"></div>
                <div id="hp-text"><span id="hp-val">100</span> HP</div>
            </div>
        </div>

        <div id="hotbar">
            <div id="slot-1" class="hotbar-slot active" data-key="1" onclick="swapTool('pickaxe')">⛏️</div>
            <div id="slot-2" class="hotbar-slot" data-key="2" onclick="swapTool('gun')">🔫</div>
            <div id="slot-3" class="hotbar-slot" data-key="3" onclick="swapTool('build')">🧱</div>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getFirestore, doc, setDoc, onSnapshot, collection, deleteDoc, updateDoc, addDoc, getDoc } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";

        // CONFIGURAÇÕES GLOBAIS
        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'fortnite-full-v1';
        
        let db, auth, currentUser;
        let scene, camera, renderer, raycaster, clock;
        let weaponGroup, stormVisual;
        let currentRoomId = ""; 
        let isJoining = false;

        const localPlayer = {
            hp: 100, wood: 50,
            pos: { x: Math.random()*40-20, y: 1.8, z: Math.random()*40-20 },
            velocity: 0, isGrounded: true, currentTool: 'pickaxe',
            visualId: 'ID-' + Math.random().toString(36).substr(2, 4).toUpperCase()
        };

        const safeZone = {
            centerX: 0, centerZ: 0,
            currentRadius: 200, targetRadius: 200,
            phases: [200, 120, 60, 15, 0],
            currentPhaseIndex: 0,
            timer: 300
        };

        const wallObjects = {}; 
        const remotePlayers = {};
        const trees = [];
        const mushrooms = []; 
        const keys = {};

        function log(msg) {
            document.getElementById('debug-log').innerText = msg;
        }

        window.joinGame = async () => {
            if (isJoining || !currentUser) return;
            isJoining = true;

            const input = document.getElementById('room-input').value.trim();
            currentRoomId = input ? input.toUpperCase() : "GLOBAL";
            
            document.getElementById('play-button').innerText = "A ENTRAR...";
            
            // Iniciar Jogo
            renderer.domElement.requestPointerLock();
            document.getElementById('start-overlay').style.display = 'none';
            document.getElementById('ui-layer').style.display = 'block';
            document.getElementById('crosshair').style.display = 'block';
            document.getElementById('current-room-id').innerText = currentRoomId;
            
            startSafeCycle();
            
            if (db && currentUser) {
                await setupNetworking();
                log("Online: Sala " + currentRoomId);
            }
        };

        function initThreeJS() {
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x87ceeb);
            scene.fog = new THREE.Fog(0x87ceeb, 20, 600);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 2000);
            camera.position.set(localPlayer.pos.x, 1.8, localPlayer.pos.z);
            camera.rotation.order = 'YXZ';

            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.shadowMap.enabled = true;
            document.body.appendChild(renderer.domElement);

            raycaster = new THREE.Raycaster();
            clock = new THREE.Clock();

            // Luzes
            scene.add(new THREE.AmbientLight(0xffffff, 0.6));
            const sun = new THREE.DirectionalLight(0xffffff, 1.0);
            sun.position.set(100, 200, 100);
            sun.castShadow = true;
            scene.add(sun);

            // Chão
            const groundGeo = new THREE.PlaneGeometry(2000, 2000);
            const groundMat = new THREE.MeshStandardMaterial({ color: 0x3d9940, roughness: 0.8 });
            const ground = new THREE.Mesh(groundGeo, groundMat);
            ground.rotation.x = -Math.PI / 2;
            ground.receiveShadow = true;
            scene.add(ground);

            // Storm Visual
            const stormGeo = new THREE.TorusGeometry(200, 2, 16, 100);
            const stormMat = new THREE.MeshBasicMaterial({ color: 0x3b82f6, transparent: true, opacity: 0.6 });
            stormVisual = new THREE.Mesh(stormGeo, stormMat);
            stormVisual.rotation.x = Math.PI/2;
            stormVisual.position.y = 0.5;
            scene.add(stormVisual);

            generateTrees();
            generateMushrooms(); 
            createWeaponModel();
            setupControls();
            animate();
        }

        async function initFirebase() {
            if (!firebaseConfig) {
                log("Aviso: Configuração Firebase não encontrada.");
                document.getElementById('play-button').innerText = "JOGAR OFFLINE";
                document.getElementById('play-button').disabled = false;
                return;
            }

            try {
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);

                // Listener de Auth
                onAuthStateChanged(auth, (u) => {
                    if (u) {
                        currentUser = u;
                        const shortId = u.uid.substring(0, 8).toUpperCase();
                        document.getElementById('my-id-display').innerText = "O TEU ID: " + shortId;
                        document.getElementById('play-button').innerText = "ENTRAR NO CAMPO DE BATALHA";
                        document.getElementById('play-button').disabled = false;
                        log("Firebase Ligado.");
                    } else {
                        attemptLogin();
                    }
                });

                async function attemptLogin() {
                    try {
                        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                            await signInWithCustomToken(auth, __initial_auth_token);
                        } else {
                            await signInAnonymously(auth);
                        }
                    } catch (e) {
                        log("Erro ao ligar. Tentando novamente...");
                        setTimeout(attemptLogin, 2000);
                    }
                }

                attemptLogin();

            } catch (e) {
                log("Erro Firebase: " + e.message);
                document.getElementById('play-button').innerText = "ERRO - JOGAR OFFLINE";
                document.getElementById('play-button').disabled = false;
            }
        }

        async function setupNetworking() {
            if (!db || !currentUser) return;

            const playersCol = collection(db, 'artifacts', appId, 'public', 'data', 'players');
            const wallsCol = collection(db, 'artifacts', appId, 'public', 'data', 'walls');

            // Escutar Jogadores
            onSnapshot(playersCol, (snap) => {
                snap.docChanges().forEach(change => {
                    const data = change.doc.data();
                    const id = change.doc.id;

                    if (id === currentUser.uid) {
                        if (data.damageTaken > 0) {
                            localPlayer.hp -= data.damageTaken;
                            updateHPUI();
                            updateDoc(doc(playersCol, id), { damageTaken: 0 });
                        }
                        return;
                    }

                    if (data.roomId !== currentRoomId) {
                        if (remotePlayers[id]) { scene.remove(remotePlayers[id]); delete remotePlayers[id]; }
                        return;
                    }

                    if (change.type === "added" || change.type === "modified") {
                        if (!remotePlayers[id]) {
                            const pGeo = new THREE.CapsuleGeometry(0.5, 1.2, 4, 8);
                            const pMat = new THREE.MeshStandardMaterial({ color: 0xff4444 });
                            const p = new THREE.Mesh(pGeo, pMat);
                            scene.add(p);
                            remotePlayers[id] = p;
                        }
                        remotePlayers[id].position.set(data.pos.x, data.pos.y - 0.9, data.pos.z);
                    } else if (change.type === "removed") {
                        if (remotePlayers[id]) { scene.remove(remotePlayers[id]); delete remotePlayers[id]; }
                    }
                });
                document.getElementById('player-count').innerText = Object.keys(remotePlayers).length + 1;
            }, (err) => log("Erro Rede Jogadores"));

            // Escutar Paredes
            onSnapshot(wallsCol, (snap) => {
                snap.docChanges().forEach(change => {
                    const data = change.doc.data();
                    const id = change.doc.id;

                    if (data.roomId !== currentRoomId) return;

                    if (change.type === "added" || change.type === "modified") {
                        if (!wallObjects[id]) {
                            const wallGeo = new THREE.BoxGeometry(4, 3, 0.4);
                            const wallMat = new THREE.MeshStandardMaterial({ color: 0x8b4513, roughness: 0.9 });
                            const wall = new THREE.Mesh(wallGeo, wallMat);
                            wall.position.set(data.x, data.y, data.z);
                            wall.rotation.y = data.ry;
                            wall.userData = { id, type: 'wall', hp: data.hp };
                            scene.add(wall);
                            wallObjects[id] = wall;
                        } else {
                            wallObjects[id].userData.hp = data.hp;
                        }
                    } else if (change.type === "removed") {
                        if (wallObjects[id]) { scene.remove(wallObjects[id]); delete wallObjects[id]; }
                    }
                });
                document.getElementById('wall-count').innerText = Object.keys(wallObjects).length;
            }, (err) => log("Erro Rede Paredes"));

            // Heartbeat
            setInterval(() => {
                if(!currentUser) return;
                setDoc(doc(playersCol, currentUser.uid), {
                    pos: { x: camera.position.x, y: camera.position.y, z: camera.position.z },
                    roomId: currentRoomId,
                    visualId: localPlayer.visualId,
                    ts: Date.now(),
                    hp: localPlayer.hp
                }, { merge: true });
            }, 300);
        }

        function generateTrees() {
            for (let i = 0; i < 80; i++) {
                const trunk = new THREE.Mesh(new THREE.CylinderGeometry(0.4, 0.6, 5), new THREE.MeshStandardMaterial({ color: 0x5d4037 }));
                const leaves = new THREE.Mesh(new THREE.ConeGeometry(3, 8, 8), new THREE.MeshStandardMaterial({ color: 0x1b5e20 }));
                trunk.position.y = 2.5; leaves.position.y = 7;
                const tree = new THREE.Group();
                tree.add(trunk, leaves);
                tree.position.set(Math.random()*600-300, 0, Math.random()*600-300);
                scene.add(tree);
                trees.push(tree);
            }
        }

        function generateMushrooms() {
            for (let i = 0; i < 120; i++) {
                const mushroom = new THREE.Group();
                const stem = new THREE.Mesh(new THREE.CylinderGeometry(0.1, 0.1, 0.3, 8), new THREE.MeshStandardMaterial({ color: 0xffffff }));
                stem.position.y = 0.15;
                const cap = new THREE.Mesh(new THREE.SphereGeometry(0.3, 16, 8, 0, Math.PI * 2, 0, Math.PI / 2), new THREE.MeshStandardMaterial({ color: 0xff0000 }));
                cap.position.y = 0.3;
                mushroom.add(stem, cap);
                mushroom.position.set(Math.random() * 800 - 400, 0, Math.random() * 800 - 400);
                scene.add(mushroom);
                mushrooms.push(mushroom);
            }
        }

        function createWeaponModel() {
            weaponGroup = new THREE.Group();
            const barrel = new THREE.Mesh(new THREE.BoxGeometry(0.12, 0.15, 0.8), new THREE.MeshStandardMaterial({ color: 0x111111 }));
            const grip = new THREE.Mesh(new THREE.BoxGeometry(0.1, 0.3, 0.15), new THREE.MeshStandardMaterial({ color: 0x111111 }));
            grip.position.set(0, -0.2, 0.2);
            weaponGroup.add(barrel, grip);
            weaponGroup.position.set(0.4, -0.4, -0.6);
            weaponGroup.visible = false;
            camera.add(weaponGroup);
        }

        function startSafeCycle() {
            setInterval(() => {
                if (safeZone.timer > 0) {
                    safeZone.timer--;
                } else {
                    if (safeZone.currentPhaseIndex < safeZone.phases.length - 1) {
                        safeZone.currentPhaseIndex++;
                        safeZone.targetRadius = safeZone.phases[safeZone.currentPhaseIndex];
                        safeZone.timer = 300;
                        document.getElementById('safe-phase').innerText = `Safe Zone - Fase ${safeZone.currentPhaseIndex + 1}`;
                    }
                }
                updateUI();
                processStormDamage();
            }, 1000);
        }

        function processStormDamage() {
            const dist = Math.sqrt(Math.pow(camera.position.x - safeZone.centerX, 2) + Math.pow(camera.position.z - safeZone.centerZ, 2));
            const warning = document.getElementById('storm-warning');
            if (dist > safeZone.currentRadius) {
                localPlayer.hp -= 2;
                warning.style.display = 'block';
                updateHPUI();
            } else {
                warning.style.display = 'none';
            }
        }

        async function handleAction() {
            if (localPlayer.currentTool === 'build') { buildWall(); return; }

            const f = document.createElement('div'); f.className = 'flash';
            document.body.appendChild(f); setTimeout(()=>f.remove(), 40);

            raycaster.setFromCamera({x:0, y:0}, camera);

            if (localPlayer.currentTool === 'pickaxe') {
                trees.forEach(t => {
                    if (camera.position.distanceTo(t.position) < 7) {
                        localPlayer.wood += 5;
                        document.getElementById('wood-val').innerText = localPlayer.wood;
                        t.scale.set(0.9, 1.1, 0.9);
                        setTimeout(() => t.scale.set(1, 1, 1), 100);
                    }
                });
            }

            const hits = raycaster.intersectObjects([...Object.values(wallObjects), ...Object.values(remotePlayers)]);

            if (hits.length > 0) {
                const target = hits[0].object;
                const damage = (localPlayer.currentTool === 'gun') ? 20 : 10; 

                const oldColor = target.material.color.getHex();
                target.material.color.setHex(0xffffff);
                setTimeout(() => target.material.color.setHex(oldColor), 60);

                if (target.userData.type === 'wall' && db) {
                    const id = target.userData.id;
                    const wallDocRef = doc(db, 'artifacts', appId, 'public', 'data', 'walls', id);
                    const snap = await getDoc(wallDocRef);
                    if(snap.exists()) {
                        const currentHp = snap.data().hp - damage;
                        if (currentHp <= 0) { await deleteDoc(wallDocRef); } 
                        else { await updateDoc(wallDocRef, { hp: currentHp }); }
                    }
                } else if (db) {
                    for (const [id, mesh] of Object.entries(remotePlayers)) {
                        if (mesh === target) {
                            const pDoc = doc(db, 'artifacts', appId, 'public', 'data', 'players', id);
                            const snap = await getDoc(pDoc);
                            const currentDmg = (snap.data().damageTaken || 0) + damage;
                            await updateDoc(pDoc, { damageTaken: currentDmg });
                            break;
                        }
                    }
                }
            }
        }

        async function buildWall() {
            if (localPlayer.wood < 10) return;
            const dir = new THREE.Vector3(0, 0, -5).applyQuaternion(camera.quaternion);
            const pos = new THREE.Vector3().copy(camera.position).add(dir);
            
            if (db) {
                try {
                    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'walls'), {
                        x: pos.x, y: 1.5, z: pos.z, ry: camera.rotation.y,
                        roomId: currentRoomId, hp: 100, ts: Date.now()
                    });
                    localPlayer.wood -= 10;
                    document.getElementById('wood-val').innerText = localPlayer.wood;
                } catch(e) { console.error("Erro Construção:", e); }
            } else {
                const wallGeo = new THREE.BoxGeometry(4, 3, 0.4);
                const wallMat = new THREE.MeshStandardMaterial({ color: 0x8b4513 });
                const wall = new THREE.Mesh(wallGeo, wallMat);
                wall.position.set(pos.x, 1.5, pos.z);
                wall.rotation.y = camera.rotation.y;
                scene.add(wall);
                localPlayer.wood -= 10;
            }
        }

        function setupControls() {
            document.addEventListener('mousemove', (e) => {
                if (document.pointerLockElement === renderer.domElement) {
                    camera.rotation.y -= e.movementX * 0.0025;
                    camera.rotation.x -= e.movementY * 0.0025;
                    camera.rotation.x = Math.max(-1.5, Math.min(1.5, camera.rotation.x));
                }
            });
            window.addEventListener('keydown', (e) => { 
                keys[e.code] = true; 
                if(e.code === 'Digit1') swapTool('pickaxe');
                if(e.code === 'Digit2') swapTool('gun');
                if(e.code === 'Digit3' || e.code === 'KeyQ') swapTool('build');
            });
            window.addEventListener('keyup', (e) => keys[e.code] = false);
            window.addEventListener('mousedown', () => { if(document.pointerLockElement === renderer.domElement) handleAction(); });
        }

        window.swapTool = (tool) => {
            localPlayer.currentTool = tool;
            weaponGroup.visible = (tool === 'gun');
            document.querySelectorAll('.hotbar-slot').forEach(s => s.classList.remove('active'));
            if(tool==='pickaxe') document.getElementById('slot-1').classList.add('active');
            if(tool==='gun') document.getElementById('slot-2').classList.add('active');
            if(tool==='build') document.getElementById('slot-3').classList.add('active');
        };

        function updateUI() {
            const mins = Math.floor(safeZone.timer / 60);
            const secs = safeZone.timer % 60;
            document.getElementById('safe-clock').innerText = `${mins.toString().padStart(2,'0')}:${secs.toString().padStart(2,'0')}`;
        }

        function updateHPUI() {
            const hp = Math.max(0, localPlayer.hp);
            document.getElementById('hp-bar-fill').style.width = Math.min(100, hp) + "%";
            document.getElementById('hp-text').innerText = `${Math.ceil(hp)} HP`;
            if (hp <= 0) { 
                alert("Foste eliminado! Regressando ao lobby..."); 
                location.reload(); 
            }
        }

        function checkCollision(newPos) {
            const pBox = new THREE.Box3().setFromCenterAndSize(newPos, new THREE.Vector3(1, 2, 1));
            for (const id in wallObjects) {
                const wBox = new THREE.Box3().setFromObject(wallObjects[id]);
                if (pBox.intersectsBox(wBox)) return true;
            }
            return false;
        }

        function animate() {
            requestAnimationFrame(animate);

            if (safeZone.currentRadius > safeZone.targetRadius) {
                safeZone.currentRadius -= 0.04;
                stormVisual.scale.set(safeZone.currentRadius / 200, safeZone.currentRadius / 200, 1);
            }

            if (document.pointerLockElement === renderer.domElement) {
                const speed = keys['ShiftLeft'] ? 0.35 : 0.15;
                const rot = camera.rotation.y;
                let dx = 0, dz = 0;

                if (keys['KeyW']) { dx -= Math.sin(rot)*speed; dz -= Math.cos(rot)*speed; }
                if (keys['KeyS']) { dx += Math.sin(rot)*speed; dz += Math.cos(rot)*speed; }
                if (keys['KeyA']) { dx -= Math.cos(rot)*speed; dz += Math.sin(rot)*speed; }
                if (keys['KeyD']) { dx += Math.cos(rot)*speed; dz -= Math.sin(rot)*speed; }
                
                const nextPos = camera.position.clone();
                nextPos.x += dx; nextPos.z += dz;
                if (!checkCollision(nextPos)) {
                    camera.position.x = nextPos.x;
                    camera.position.z = nextPos.z;
                }

                if (keys['Space'] && localPlayer.isGrounded) {
                    localPlayer.velocity = 0.22;
                    localPlayer.isGrounded = false;
                }

                let nearMushroom = false;
                const prompt = document.getElementById('interaction-prompt');

                for (let i = mushrooms.length - 1; i >= 0; i--) {
                    const mush = mushrooms[i];
                    const dist = camera.position.distanceTo(mush.position);
                    if (dist < 2.5) { 
                        nearMushroom = true;
                        if (keys['KeyG'] && localPlayer.hp < 100) {
                            localPlayer.hp = Math.min(100, localPlayer.hp + 15);
                            updateHPUI();
                            scene.remove(mush);
                            mushrooms.splice(i, 1);
                        }
                    }
                }
                prompt.style.display = nearMushroom ? 'block' : 'none';
            }

            localPlayer.velocity -= 0.01;
            camera.position.y += localPlayer.velocity;
            if (camera.position.y <= 1.8) {
                camera.position.y = 1.8;
                localPlayer.velocity = 0;
                localPlayer.isGrounded = true;
            }

            renderer.render(scene, camera);
        }

        window.onload = () => { 
            initThreeJS(); 
            initFirebase(); 
        };
        window.onresize = () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        };
    </script>
</body>
</html>
