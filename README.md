<!DOCTYPE html>
<html lang="pt-PT">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Fortnite PC Multiplayer</title>
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

        /* Camada de Início - A "Tela Roxa" */
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
            cursor: pointer;
            text-align: center;
            transition: opacity 0.5s ease;
        }

        #ui-layer {
            position: fixed;
            inset: 0;
            pointer-events: none;
            display: none;
            z-index: 10;
        }

        #stats-container {
            position: absolute;
            bottom: 30px;
            left: 50%;
            transform: translateX(-50%);
            display: flex;
            align-items: flex-end;
            gap: 20px;
            pointer-events: none;
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
            flex-direction: column;
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

        #instructions {
            margin-top: 20px;
            font-size: 14px;
            background: rgba(0,0,0,0.3);
            padding: 20px;
            border-radius: 10px;
            line-height: 1.6;
        }

        .flash-effect {
            position: fixed;
            inset: 0;
            background: #fff;
            opacity: 0.2;
            z-index: 1000;
            pointer-events: none;
        }
    </style>
</head>
<body>
    <div id="start-overlay">
        <h1>FORTNITE PC EDITION</h1>
        <div id="instructions">
            <p><strong>CLIQUE EM QUALQUER LUGAR PARA ENTRAR</strong></p>
            <br>
            <p>W, A, S, D - Mover | ESPAÇO - Saltar</p>
            <p>CLIQUE ESQUERDO - Disparar ou Usar Picareta</p>
            <p>Q - Construir Parede (Custo: 10 Madeira)</p>
            <p>1 / 2 ou E - Trocar Item</p>
            <p>ESC - Menu / Libertar Rato</p>
        </div>
    </div>
    
    <div id="crosshair"></div>
    
    <div id="ui-layer">
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
        import { getFirestore, doc, setDoc, onSnapshot, collection, deleteDoc, updateDoc } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";

        // Configurações do Firebase
        const firebaseConfig = JSON.parse(__firebase_config);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'fortnite-pc-stable';
        
        let db, auth, currentUser;
        let scene, camera, renderer, clock, raycaster;
        let weaponGroup;
        let gameStarted = false;
        
        const localPlayer = {
            id: 'local-' + Math.random().toString(36).substr(2, 5),
            hp: 100,
            wood: 50,
            pos: { x: Math.random() * 20 - 10, y: 1.8, z: Math.random() * 20 - 10 },
            velocity: 0,
            isGrounded: true,
            currentTool: 'pickaxe'
        };

        const remotePlayers = {};
        const wallObjects = {};
        const trees = [];
        const keys = {};

        // 1. Inicializar Motor Gráfico (Não depende do Firebase)
        function initThreeJS() {
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x87ceeb);
            scene.fog = new THREE.Fog(0x87ceeb, 20, 150);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
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
                new THREE.PlaneGeometry(2000, 2000),
                new THREE.MeshPhongMaterial({ color: 0x4caf50 })
            );
            ground.rotation.x = -Math.PI / 2;
            scene.add(ground);

            generateEnvironment();
            createWeaponModel();
            setupPointerLock();
            animate();
        }

        // 2. Inicializar Firebase (Silencioso em background)
        async function initFirebase() {
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
                        localPlayer.id = user.uid;
                        setupNetworking();
                    }
                });
            } catch (e) {
                console.warn("Ligação ao Firebase falhou. Modo offline ativo.", e);
            }
        }

        function setupNetworking() {
            if (!db || !currentUser) return;

            const playersCol = collection(db, 'artifacts', appId, 'public', 'data', 'players');
            const wallsCol = collection(db, 'artifacts', appId, 'public', 'data', 'walls');

            onSnapshot(playersCol, (snap) => {
                snap.docChanges().forEach(change => {
                    const data = change.doc.data();
                    const id = change.doc.id;
                    if (id === localPlayer.id) {
                        localPlayer.hp = data.hp ?? 100;
                        updateUI();
                        if (localPlayer.hp <= 0) location.reload();
                        return;
                    }
                    if (change.type === "added" || change.type === "modified") {
                        if (!remotePlayers[id]) {
                            const m = new THREE.Mesh(new THREE.BoxGeometry(1, 2, 1), new THREE.MeshPhongMaterial({ color: 0xff4444 }));
                            m.userData = { id, type: 'player' };
                            scene.add(m);
                            remotePlayers[id] = m;
                        }
                        if (data.pos) remotePlayers[id].position.set(data.pos.x, data.pos.y, data.pos.z);
                        remotePlayers[id].userData.hp = data.hp;
                    } else if (change.type === "removed") {
                        if (remotePlayers[id]) { scene.remove(remotePlayers[id]); delete remotePlayers[id]; }
                    }
                });
            }, (err) => console.log("Firebase Players Busy"));

            onSnapshot(wallsCol, (snap) => {
                snap.docChanges().forEach(change => {
                    const id = change.doc.id;
                    const data = change.doc.data();
                    if (change.type === "added" || change.type === "modified") {
                        if (!wallObjects[id]) {
                            const wall = new THREE.Mesh(new THREE.BoxGeometry(4, 3, 0.4), new THREE.MeshPhongMaterial({ color: 0x795548 }));
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
            }, (err) => console.log("Firebase Walls Busy"));

            setInterval(() => {
                if (!db || !currentUser) return;
                setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'players', localPlayer.id), {
                    pos: { x: camera.position.x, y: camera.position.y, z: camera.position.z },
                    hp: localPlayer.hp,
                    ts: Date.now()
                }, { merge: true }).catch(() => {});
            }, 100);
        }

        function setupPointerLock() {
            const canvas = renderer.domElement;
            const overlay = document.getElementById('start-overlay');
            const ui = document.getElementById('ui-layer');
            const crosshair = document.getElementById('crosshair');

            // CORREÇÃO CRUCIAL: Remover tela roxa imediatamente ao clicar
            overlay.addEventListener('click', () => {
                canvas.requestPointerLock();
                // Forçar desaparecimento se o evento PointerLock demorar
                overlay.style.opacity = '0';
                setTimeout(() => overlay.style.display = 'none', 500);
            });

            document.addEventListener('pointerlockchange', () => {
                if (document.pointerLockElement === canvas) {
                    overlay.style.display = 'none';
                    ui.style.display = 'block';
                    crosshair.style.display = 'block';
                    gameStarted = true;
                } else {
                    overlay.style.display = 'flex';
                    overlay.style.opacity = '1';
                    ui.style.display = 'none';
                    crosshair.style.display = 'none';
                }
            });

            document.addEventListener('mousemove', (e) => {
                if (document.pointerLockElement === canvas) {
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
                if(document.pointerLockElement === canvas && e.button === 0) handleAction(); 
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
            scene.add(camera);
        }

        function generateEnvironment() {
            const trunkGeo = new THREE.CylinderGeometry(0.4, 0.5, 5);
            const leafGeo = new THREE.ConeGeometry(2, 6);
            const tMat = new THREE.MeshPhongMaterial({ color: 0x5d4037 });
            const lMat = new THREE.MeshPhongMaterial({ color: 0x2e7d32 });
            for (let i = 0; i < 60; i++) {
                const tree = new THREE.Group();
                const t = new THREE.Mesh(trunkGeo, tMat); t.position.y = 2.5;
                const l = new THREE.Mesh(leafGeo, lMat); l.position.y = 6;
                tree.add(t, l);
                tree.position.set(Math.random() * 400 - 200, 0, Math.random() * 400 - 200);
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

            if (localPlayer.currentTool === 'pickaxe') {
                trees.forEach(tree => {
                    if (camera.position.distanceTo(tree.position) < 8) {
                        localPlayer.wood += 5;
                        document.getElementById('wood-val').innerText = localPlayer.wood;
                    }
                });
            }

            if (!db) return; // Proteção modo offline

            const targets = [...Object.values(remotePlayers), ...Object.values(wallObjects)];
            const hits = raycaster.intersectObjects(targets);
            if (hits.length > 0) {
                const obj = hits[0].object;
                const dmg = localPlayer.currentTool === 'gun' ? 30 : 15;
                if (obj.userData.type === 'player') {
                    updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'players', obj.userData.id), {
                        hp: Math.max(0, (obj.userData.hp || 100) - dmg)
                    }).catch(()=>{});
                } else if (obj.userData.type === 'wall') {
                    const newHp = (obj.userData.hp || 100) - dmg;
                    const ref = doc(db, 'artifacts', appId, 'public', 'data', 'walls', obj.userData.id);
                    if (newHp <= 0) deleteDoc(ref).catch(()=>{});
                    else updateDoc(ref, { hp: newHp }).catch(()=>{});
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
                setDoc(doc(collection(db, 'artifacts', appId, 'public', 'data', 'walls')), { 
                    x: pos.x, y: 1.5, z: pos.z, ry: camera.rotation.y, hp: 100 
                }).catch(()=>{});
            } else {
                // Feedback visual local mesmo offline
                const wall = new THREE.Mesh(new THREE.BoxGeometry(4, 3, 0.4), new THREE.MeshPhongMaterial({ color: 0x795548 }));
                wall.position.set(pos.x, 1.5, pos.z);
                wall.rotation.y = camera.rotation.y;
                scene.add(wall);
            }
        }

        function updateUI() {
            document.getElementById('hp-bar-fill').style.width = localPlayer.hp + "%";
            document.getElementById('hp-val').innerText = Math.ceil(localPlayer.hp);
        }

        function animate() {
            requestAnimationFrame(animate);
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
