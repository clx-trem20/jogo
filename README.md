<!DOCTYPE html>
<html lang="pt-PT">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Fortnite PC Multiplayer - Lobby System</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body, html {
            width: 100%;
            height: 100%;
            overflow: hidden;
            background: #000;
            font-family: 'Segoe UI', sans-serif;
            user-select: none;
        }

        canvas {
            display: block;
        }

        /* Camada de Início */
        #start-overlay {
            position: fixed;
            inset: 0;
            background: linear-gradient(135deg, #6d28d9 0%, #4c1d95 100%);
            color: white;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            z-index: 9999;
            text-align: center;
        }

        #lobby-container {
            background: rgba(0,0,0,0.4);
            padding: 30px;
            border-radius: 15px;
            margin-top: 20px;
            width: 350px;
        }

        input {
            width: 100%;
            padding: 12px;
            margin: 10px 0;
            border-radius: 8px;
            border: none;
            font-size: 16px;
            text-align: center;
        }

        .btn-play {
            background: #fcd34d;
            color: #000;
            border: none;
            padding: 15px 30px;
            font-weight: bold;
            font-size: 18px;
            border-radius: 8px;
            cursor: pointer;
            width: 100%;
            transition: transform 0.2s;
        }

        .btn-play:hover {
            transform: scale(1.05);
            background: #fbbf24;
        }

        #ui-layer {
            position: fixed;
            inset: 0;
            pointer-events: none;
            display: none;
            z-index: 10;
        }

        /* Indicador da Safe Zone */
        #safe-timer-container {
            position: absolute;
            top: 20px;
            left: 50%;
            transform: translateX(-50%);
            background: rgba(0, 0, 0, 0.7);
            padding: 10px 20px;
            border-radius: 8px;
            border: 2px solid #3b82f6;
            color: white;
            text-align: center;
        }

        #room-info {
            position: absolute;
            top: 20px;
            left: 20px;
            background: rgba(0,0,0,0.6);
            color: white;
            padding: 10px;
            border-radius: 8px;
            font-size: 14px;
        }

        #safe-clock {
            font-size: 24px;
            font-weight: bold;
        }

        #stats-container {
            position: absolute;
            bottom: 30px;
            left: 50%;
            transform: translateX(-50%);
            display: flex;
            align-items: flex-end;
            gap: 20px;
        }

        #hp-container {
            width: 250px;
            height: 30px;
            background: rgba(0,0,0,0.6);
            border: 2px solid rgba(255,255,255,0.3);
            border-radius: 4px;
            overflow: hidden;
            position: relative;
        }

        #hp-bar-fill {
            width: 100%;
            height: 100%;
            background: #4ade80;
            transition: width 0.3s ease;
        }

        #hp-text {
            position: absolute;
            inset: 0;
            display: flex;
            align-items: center;
            justify-content: center;
            color: white;
            font-weight: bold;
            text-shadow: 1px 1px 2px #000;
        }

        .resource-box {
            background: rgba(0,0,0,0.7);
            padding: 10px 20px;
            border-radius: 4px;
            color: #fff;
            font-weight: bold;
            border-bottom: 4px solid #fcd34d;
        }

        #hotbar {
            position: absolute;
            bottom: 30px;
            right: 30px;
            display: flex;
            gap: 10px;
        }

        .hotbar-slot {
            width: 60px;
            height: 60px;
            background: rgba(0,0,0,0.6);
            border: 2px solid rgba(255,255,255,0.2);
            border-radius: 8px;
            display: flex;
            align-items: center;
            justify-content: center;
            color: white;
            font-size: 20px;
        }

        .hotbar-slot.active {
            border-color: #fcd34d;
            background: rgba(252, 211, 77, 0.2);
        }

        #crosshair {
            position: fixed;
            top: 50%;
            left: 50%;
            width: 20px;
            height: 20px;
            transform: translate(-50%, -50%);
            pointer-events: none;
            z-index: 5;
            display: none;
        }

        #crosshair::before, #crosshair::after {
            content: '';
            position: absolute;
            background: rgba(255, 255, 255, 0.8);
        }

        #crosshair::before { top: 50%; left: 0; width: 100%; height: 2px; transform: translateY(-50%); }
        #crosshair::after { left: 50%; top: 0; height: 100%; width: 2px; transform: translateX(-50%); }

        .flash-effect {
            position: fixed;
            inset: 0;
            background: #fff;
            opacity: 0.2;
            z-index: 1000;
            pointer-events: none;
        }

        #storm-warning {
            position: fixed;
            top: 100px;
            left: 50%;
            transform: translateX(-50%);
            color: #ef4444;
            font-weight: bold;
            font-size: 24px;
            text-shadow: 2px 2px 4px #000;
            display: none;
        }
    </style>
</head>
<body>
    <div id="start-overlay">
        <h1>FORTNITE PC EDITION</h1>
        <div id="lobby-container">
            <p>O TEU ID: <strong id="my-id-display">...</strong></p>
            <input type="text" id="room-input" placeholder="ID DO AMIGO OU NOME DA SALA">
            <button class="btn-play" onclick="joinGame()">ENTRAR NO MAPA</button>
            <p style="font-size: 12px; margin-top: 10px; opacity: 0.8;">Deixa em branco para jogar sozinho ou espera amigos entrarem no teu ID.</p>
        </div>
    </div>
    
    <div id="crosshair"></div>
    
    <div id="ui-layer">
        <div id="room-info">SALA: <span id="current-room-id">---</span></div>
        <div id="safe-timer-container">
            <div id="safe-phase">Safe Zone - Fase 1/3</div>
            <div id="safe-clock">05:00</div>
        </div>

        <div id="storm-warning">ESTÁS FORA DA SAFE ZONE!</div>

        <div id="stats-container">
            <div class="stat-group">
                <div id="hp-container">
                    <div id="hp-bar-fill"></div>
                    <div id="hp-text"><span id="hp-val">100</span> HP</div>
                </div>
            </div>
            <div class="resource-box">
                🪵 <span id="wood-val">50</span>
            </div>
        </div>

        <div id="hotbar">
            <div id="slot-1" class="hotbar-slot active">⛏️</div>
            <div id="slot-2" class="hotbar-slot">🔫</div>
            <div id="slot-3" class="hotbar-slot">🧱</div>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getFirestore, doc, setDoc, onSnapshot, collection, deleteDoc, updateDoc, addDoc, query, where } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";

        let firebaseConfig = {};
        try { firebaseConfig = JSON.parse(__firebase_config); } catch(e) { console.error("Config erro"); }
        
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'fortnite-pc-stable';
        
        let db, auth, currentUser;
        let scene, camera, renderer, clock, raycaster;
        let weaponGroup, stormCircle;
        let currentRoomId = "";
        
        const localPlayer = {
            id: 'ID-' + Math.random().toString(36).substr(2, 6).toUpperCase(),
            hp: 100,
            wood: 50,
            pos: { x: Math.random() * 40 - 20, y: 1.8, z: Math.random() * 40 - 20 },
            velocity: 0,
            isGrounded: true,
            currentTool: 'pickaxe'
        };

        const safeZone = {
            currentRadius: 150,
            targetRadius: 150,
            phases: [150, 100, 50, 10], 
            currentPhase: 0,
            timer: 300, 
            damage: 2,
            centerX: 0,
            centerZ: 0
        };

        const remotePlayers = {};
        const wallObjects = {}; 
        const trees = [];
        const keys = {};

        // Exibir ID local no menu
        document.getElementById('my-id-display').innerText = localPlayer.id;

        window.joinGame = () => {
            const input = document.getElementById('room-input').value.trim();
            currentRoomId = input || localPlayer.id;
            document.getElementById('current-room-id').innerText = currentRoomId;
            
            renderer.domElement.requestPointerLock();
            document.getElementById('start-overlay').style.display = 'none';
            document.getElementById('ui-layer').style.display = 'block';
            document.getElementById('crosshair').style.display = 'block';
            
            startSafeTimer();
            setupNetworking();
        };

        function initThreeJS() {
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x87ceeb);
            scene.fog = new THREE.Fog(0x87ceeb, 20, 250);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 2000);
            camera.position.set(localPlayer.pos.x, 1.8, localPlayer.pos.z);
            camera.rotation.order = 'YXZ';

            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(window.devicePixelRatio);
            document.body.appendChild(renderer.domElement);

            raycaster = new THREE.Raycaster();
            clock = new THREE.Clock();

            scene.add(new THREE.AmbientLight(0xffffff, 0.6));
            const sun = new THREE.DirectionalLight(0xffffff, 1.0);
            sun.position.set(50, 100, 50);
            scene.add(sun);

            const ground = new THREE.Mesh(
                new THREE.PlaneGeometry(5000, 5000),
                new THREE.MeshPhongMaterial({ color: 0x4caf50 })
            );
            ground.rotation.x = -Math.PI / 2;
            scene.add(ground);

            const stormGeo = new THREE.RingGeometry(148, 150, 64);
            const stormMat = new THREE.MeshBasicMaterial({ color: 0x3b82f6, side: THREE.DoubleSide, transparent: true, opacity: 0.5 });
            stormCircle = new THREE.Mesh(stormGeo, stormMat);
            stormCircle.rotation.x = -Math.PI / 2;
            stormCircle.position.y = 0.1;
            scene.add(stormCircle);

            generateEnvironment();
            createWeaponModel();
            setupControls();
            animate();
        }

        function startSafeTimer() {
            setInterval(() => {
                if (safeZone.timer > 0) {
                    safeZone.timer--;
                } else {
                    if (safeZone.currentPhase < 3) {
                        safeZone.currentPhase++;
                        safeZone.targetRadius = safeZone.phases[safeZone.currentPhase];
                        safeZone.timer = 300; 
                        document.getElementById('safe-phase').innerText = `Safe Zone - Fase ${Math.min(safeZone.currentPhase + 1, 3)}/3`;
                    }
                }
                updateClockUI();
                checkStormDamage();
            }, 1000);
        }

        function updateClockUI() {
            const mins = Math.floor(safeZone.timer / 60);
            const secs = safeZone.timer % 60;
            document.getElementById('safe-clock').innerText = `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
        }

        function checkStormDamage() {
            const dist = Math.sqrt(
                Math.pow(camera.position.x - safeZone.centerX, 2) + 
                Math.pow(camera.position.z - safeZone.centerZ, 2)
            );

            const warning = document.getElementById('storm-warning');
            if (dist > safeZone.currentRadius) {
                localPlayer.hp -= safeZone.damage;
                warning.style.display = 'block';
                updateUI();
                if (localPlayer.hp <= 0) location.reload();
            } else {
                warning.style.display = 'none';
            }
        }

        async function initFirebase() {
            if (typeof __firebase_config === 'undefined') return;
            try {
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);

                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
                
                onAuthStateChanged(auth, (user) => {
                    if (user) {
                        currentUser = user;
                        // Mantemos o ID visual amigável para o Lobby, mas usamos o UID para Firebase
                    }
                });
            } catch (e) { console.warn("Modo Offline."); }
        }

        function setupNetworking() {
            if (!db || !currentUser) return;

            // Filtramos as coleções pelo roomId para que grupos diferentes não se vejam
            const playersCol = collection(db, 'artifacts', appId, 'public', 'data', 'players');
            const wallsCol = collection(db, 'artifacts', appId, 'public', 'data', 'walls');

            // Ouvir apenas jogadores da mesma sala
            onSnapshot(playersCol, (snap) => {
                snap.docChanges().forEach(change => {
                    const data = change.doc.data();
                    const id = change.doc.id;
                    
                    if (data.roomId !== currentRoomId) return;
                    if (id === currentUser.uid) {
                        localPlayer.hp = data.hp ?? 100;
                        updateUI();
                        return;
                    }

                    if (change.type === "added" || change.type === "modified") {
                        if (!remotePlayers[id]) {
                            const m = new THREE.Mesh(new THREE.BoxGeometry(1, 2, 1), new THREE.MeshPhongMaterial({ color: 0xff4444 }));
                            scene.add(m);
                            remotePlayers[id] = m;
                        }
                        if (data.pos) remotePlayers[id].position.set(data.pos.x, data.pos.y, data.pos.z);
                        remotePlayers[id].userData = { id, type: 'player', hp: data.hp };
                    } else if (change.type === "removed") {
                        if (remotePlayers[id]) { scene.remove(remotePlayers[id]); delete remotePlayers[id]; }
                    }
                });
            });

            // Ouvir apenas paredes da mesma sala
            onSnapshot(wallsCol, (snap) => {
                snap.docChanges().forEach(change => {
                    const id = change.doc.id;
                    const data = change.doc.data();

                    if (data.roomId !== currentRoomId) return;

                    if (change.type === "added" || change.type === "modified") {
                        if (!wallObjects[id]) {
                            const wall = new THREE.Mesh(new THREE.BoxGeometry(4, 3, 0.4), new THREE.MeshPhongMaterial({ color: 0x795548 }));
                            scene.add(wall);
                            wallObjects[id] = wall;
                        }
                        wallObjects[id].position.set(data.x, data.y, data.z);
                        wallObjects[id].rotation.y = data.ry;
                        wallObjects[id].userData = { id, type: 'wall', hp: data.hp };
                    } else if (change.type === "removed") {
                        if (wallObjects[id]) { scene.remove(wallObjects[id]); delete wallObjects[id]; }
                    }
                });
            });

            // Enviar posição com Room ID
            setInterval(() => {
                if (!db || !currentUser) return;
                setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'players', currentUser.uid), {
                    pos: { x: camera.position.x, y: camera.position.y, z: camera.position.z },
                    hp: localPlayer.hp,
                    roomId: currentRoomId,
                    ts: Date.now()
                }, { merge: true }).catch(()=>{});
            }, 100);
        }

        function setupControls() {
            document.addEventListener('pointerlockchange', () => {
                if (document.pointerLockElement !== renderer.domElement) {
                    document.getElementById('start-overlay').style.display = 'flex';
                }
            });

            document.addEventListener('mousemove', (e) => {
                if (document.pointerLockElement === renderer.domElement) {
                    camera.rotation.y -= e.movementX * 0.0025;
                    camera.rotation.x -= e.movementY * 0.0025;
                    camera.rotation.x = Math.max(-1.5, Math.min(1.5, camera.rotation.x));
                }
            });

            window.addEventListener('keydown', (e) => { 
                keys[e.code] = true; 
                if(e.code === 'KeyQ') buildWall(); 
                if(e.code === 'Digit1' || e.code === 'Digit2' || e.code === 'KeyE') swapTool(); 
            });
            window.addEventListener('keyup', (e) => keys[e.code] = false);
            window.addEventListener('mousedown', (e) => { 
                if(document.pointerLockElement === renderer.domElement && e.button === 0) handleAction(); 
            });
        }

        function createWeaponModel() {
            weaponGroup = new THREE.Group();
            const barrel = new THREE.Mesh(new THREE.CylinderGeometry(0.02, 0.02, 0.6), new THREE.MeshPhongMaterial({ color: 0x333 }));
            barrel.rotation.x = Math.PI/2; barrel.position.z = -0.3;
            weaponGroup.add(barrel);
            weaponGroup.position.set(0.35, -0.3, -0.4);
            weaponGroup.visible = false;
            camera.add(weaponGroup);
        }

        function generateEnvironment() {
            const trunkGeo = new THREE.CylinderGeometry(0.4, 0.5, 5);
            const leafGeo = new THREE.ConeGeometry(2, 6);
            const tMat = new THREE.MeshPhongMaterial({ color: 0x5d4037 });
            const lMat = new THREE.MeshPhongMaterial({ color: 0x2e7d32 });
            for (let i = 0; i < 150; i++) {
                const tree = new THREE.Group();
                const t = new THREE.Mesh(trunkGeo, tMat); t.position.y = 2.5;
                const l = new THREE.Mesh(leafGeo, lMat); l.position.y = 6;
                tree.add(t, l);
                tree.position.set(Math.random() * 800 - 400, 0, Math.random() * 800 - 400);
                scene.add(tree);
                trees.push(tree);
            }
        }

        function swapTool() {
            localPlayer.currentTool = localPlayer.currentTool === 'pickaxe' ? 'gun' : 'pickaxe';
            weaponGroup.visible = (localPlayer.currentTool === 'gun');
            document.getElementById('slot-1').classList.toggle('active', localPlayer.currentTool === 'pickaxe');
            document.getElementById('slot-2').classList.toggle('active', localPlayer.currentTool === 'gun');
        }

        async function handleAction() {
            const flash = document.createElement('div'); flash.className = 'flash-effect';
            document.body.appendChild(flash); setTimeout(() => flash.remove(), 50);

            raycaster.setFromCamera({x:0, y:0}, camera);
            raycaster.far = localPlayer.currentTool === 'pickaxe' ? 8 : 200;

            if (localPlayer.currentTool === 'pickaxe') {
                trees.forEach(tree => {
                    if (camera.position.distanceTo(tree.position) < 8) {
                        localPlayer.wood += 5;
                        document.getElementById('wood-val').innerText = localPlayer.wood;
                    }
                });
            }

            if (!db) return;

            const targets = [...Object.values(remotePlayers), ...Object.values(wallObjects)];
            const hits = raycaster.intersectObjects(targets);
            
            if (hits.length > 0) {
                const obj = hits[0].object;
                const userData = obj.userData;
                if (!userData || !userData.id) return;

                const dmg = localPlayer.currentTool === 'gun' ? 30 : 25;

                if (userData.type === 'player') {
                    updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'players', userData.id), {
                        hp: Math.max(0, (userData.hp || 100) - dmg)
                    }).catch(()=>{});
                } else if (userData.type === 'wall') {
                    const wallRef = doc(db, 'artifacts', appId, 'public', 'data', 'walls', userData.id);
                    const newHp = (userData.hp || 100) - dmg;
                    if (newHp <= 0) deleteDoc(wallRef).catch(()=>{});
                    else updateDoc(wallRef, { hp: newHp }).catch(()=>{});
                }
            }
        }

        async function buildWall() {
            if (localPlayer.wood < 10) return;
            localPlayer.wood -= 10;
            document.getElementById('wood-val').innerText = localPlayer.wood;
            
            const dir = new THREE.Vector3(0, 0, -5).applyQuaternion(camera.quaternion);
            const pos = new THREE.Vector3().copy(camera.position).add(dir);
            
            if (db && currentUser) {
                addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'walls'), { 
                    x: pos.x, y: 1.5, z: pos.z, ry: camera.rotation.y, 
                    hp: 100, roomId: currentRoomId, owner: currentUser.uid
                }).catch(()=>{});
            }
        }

        function updateUI() {
            document.getElementById('hp-bar-fill').style.width = localPlayer.hp + "%";
            document.getElementById('hp-val').innerText = Math.max(0, Math.ceil(localPlayer.hp));
        }

        function animate() {
            requestAnimationFrame(animate);
            if (safeZone.currentRadius > safeZone.targetRadius) {
                safeZone.currentRadius -= 0.05;
                const s = safeZone.currentRadius / 150;
                stormCircle.scale.set(s, s, 1);
            }

            if (document.pointerLockElement === renderer.domElement) {
                const speed = 0.15;
                const dirY = camera.rotation.y;
                const fwd = new THREE.Vector3(-Math.sin(dirY), 0, -Math.cos(dirY));
                const rgt = new THREE.Vector3(Math.cos(dirY), 0, -Math.sin(dirY));
                if (keys['KeyW']) camera.position.addScaledVector(fwd, speed);
                if (keys['KeyS']) camera.position.addScaledVector(fwd, -speed);
                if (keys['KeyA']) camera.position.addScaledVector(rgt, -speed);
                if (keys['KeyD']) camera.position.addScaledVector(rgt, speed);
                if (keys['Space'] && localPlayer.isGrounded) { localPlayer.velocity = 0.22; localPlayer.isGrounded = false; }
            }
            localPlayer.velocity -= 0.01;
            camera.position.y += localPlayer.velocity;
            if (camera.position.y <= 1.8) { camera.position.y = 1.8; localPlayer.velocity = 0; localPlayer.isGrounded = true; }
            renderer.render(scene, camera);
        }

        window.onload = () => {
            initThreeJS();
            initFirebase();
        };

        window.onresize = () => {
            if (!camera) return;
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        };
    </script>
</body>
</html>
