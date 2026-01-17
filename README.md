<!DOCTYPE html>
<html lang="pt-PT">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Fortnite Mobile Multiplayer</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getFirestore, doc, setDoc, onSnapshot, collection, deleteDoc, updateDoc } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";

        // Configuração Global
        const firebaseConfig = JSON.parse(__firebase_config);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'fortnite-mobile';
        
        let db, auth;
        let scene, camera, renderer, clock, raycaster;
        let isStarted = false;
        let weaponGroup;
        
        const localPlayer = {
            id: "temp-" + Math.random().toString(36).substr(2, 9),
            hp: 100,
            maxHp: 100,
            wood: 50,
            pos: { x: Math.random() * 40 - 20, y: 1.8, z: Math.random() * 40 - 20 },
            velocity: 0,
            isGrounded: true,
            currentTool: 'pickaxe'
        };

        const remotePlayers = {};
        const wallObjects = {};
        const trees = [];
        let moveDir = { x: 0, z: 0 };

        async function initGame() {
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x87ceeb);
            scene.fog = new THREE.Fog(0x87ceeb, 0, 150);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(localPlayer.pos.x, 1.8, localPlayer.pos.z);
            camera.rotation.order = 'YXZ';

            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(window.devicePixelRatio);
            document.body.appendChild(renderer.domElement);

            raycaster = new THREE.Raycaster();
            clock = new THREE.Clock();

            const sun = new THREE.DirectionalLight(0xffffff, 1.2);
            sun.position.set(50, 100, 50);
            scene.add(sun);
            scene.add(new THREE.AmbientLight(0x404040, 1.0));

            const ground = new THREE.Mesh(
                new THREE.PlaneGeometry(1000, 1000),
                new THREE.MeshPhongMaterial({ color: 0x348C31 })
            );
            ground.rotation.x = -Math.PI / 2;
            scene.add(ground);

            generateEnvironment();
            createWeaponModel();
            setupMobileControls();

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
                        localPlayer.id = user.uid;
                        setupNetworking();
                    }
                });
            } catch (e) {
                console.warn("Modo Offline Ativo:", e);
            }

            animate();
        }

        function createWeaponModel() {
            weaponGroup = new THREE.Group();
            const body = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.3, 1.2), new THREE.MeshPhongMaterial({ color: 0x8B7355 }));
            const barrel = new THREE.Mesh(new THREE.CylinderGeometry(0.04, 0.04, 0.8), new THREE.MeshPhongMaterial({ color: 0x222222 }));
            barrel.rotation.x = Math.PI / 2;
            barrel.position.z = -0.8;
            weaponGroup.add(body, barrel);
            weaponGroup.position.set(0.4, -0.4, -0.6);
            weaponGroup.visible = false;
            camera.add(weaponGroup);
            scene.add(camera);
        }

        function generateEnvironment() {
            const trunkGeo = new THREE.CylinderGeometry(0.5, 0.7, 4, 8);
            const leavesGeo = new THREE.ConeGeometry(3, 6, 8);
            const trunkMat = new THREE.MeshPhongMaterial({ color: 0x4d2926 });
            const leavesMat = new THREE.MeshPhongMaterial({ color: 0x228B22 });

            for (let i = 0; i < 40; i++) {
                const group = new THREE.Group();
                const trunk = new THREE.Mesh(trunkGeo, trunkMat);
                trunk.position.y = 2;
                const leaves = new THREE.Mesh(leavesGeo, leavesMat);
                leaves.position.y = 6;
                group.add(trunk, leaves);
                group.position.set(Math.random() * 200 - 100, 0, Math.random() * 200 - 100);
                scene.add(group);
                trees.push(group);
            }
        }

        function setupNetworking() {
            if (!db) return;
            const playersRef = collection(db, 'artifacts', appId, 'public', 'data', 'players');
            const wallsRef = collection(db, 'artifacts', appId, 'public', 'data', 'walls');

            onSnapshot(playersRef, (snapshot) => {
                snapshot.docChanges().forEach(change => {
                    const data = change.doc.data();
                    if (change.doc.id === localPlayer.id) {
                        // Se o HP no servidor mudar, atualiza localmente
                        if (data.hp !== undefined && data.hp !== localPlayer.hp) {
                            localPlayer.hp = data.hp;
                            updateHpUI();
                        }
                        return;
                    }
                    if (change.type === "added" || change.type === "modified") {
                        if (!remotePlayers[change.doc.id]) {
                            const mesh = new THREE.Mesh(new THREE.BoxGeometry(1.2, 2.2, 1.2), new THREE.MeshPhongMaterial({ color: 0xff0000 }));
                            mesh.userData.id = change.doc.id;
                            mesh.userData.type = 'player';
                            scene.add(mesh);
                            remotePlayers[change.doc.id] = mesh;
                        }
                        remotePlayers[change.doc.id].position.set(data.pos.x, data.pos.y, data.pos.z);
                    } else if (change.type === "removed") {
                        scene.remove(remotePlayers[change.doc.id]);
                        delete remotePlayers[change.doc.id];
                    }
                });
            });

            onSnapshot(wallsRef, (snapshot) => {
                snapshot.docChanges().forEach(change => {
                    const id = change.doc.id;
                    const data = change.doc.data();
                    if (change.type === "added" || change.type === "modified") {
                        if (data.hp <= 0) {
                            if (wallObjects[id]) { scene.remove(wallObjects[id]); delete wallObjects[id]; }
                        } else {
                            if (wallObjects[id]) {
                                wallObjects[id].userData.hp = data.hp;
                            } else {
                                const wall = new THREE.Mesh(new THREE.BoxGeometry(4, 3, 0.4), new THREE.MeshPhongMaterial({ color: 0x8B4513 }));
                                wall.position.set(data.x, data.y, data.z);
                                wall.rotation.y = data.ry;
                                wall.userData = { id: id, type: 'wall', hp: data.hp };
                                scene.add(wall);
                                wallObjects[id] = wall;
                            }
                        }
                    } else if (change.type === "removed") {
                        if (wallObjects[id]) { scene.remove(wallObjects[id]); delete wallObjects[id]; }
                    }
                });
            });

            setInterval(async () => {
                if (auth?.currentUser) {
                    await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'players', localPlayer.id), {
                        pos: { x: camera.position.x, y: camera.position.y, z: camera.position.z },
                        hp: localPlayer.hp,
                        lastUpdate: Date.now()
                    }, { merge: true });
                }
            }, 150);
        }

        function updateHpUI() {
            const hpBar = document.getElementById('hp-bar-fill');
            const hpVal = document.getElementById('hp-val');
            const pct = Math.max(0, (localPlayer.hp / localPlayer.maxHp) * 100);
            hpBar.style.width = pct + "%";
            hpVal.innerText = Math.ceil(localPlayer.hp);
            
            if (pct < 30) hpBar.style.background = "#ef4444";
            else if (pct < 60) hpBar.style.background = "#f59e0b";
            else hpBar.style.background = "#4ade80";
        }

        function setupMobileControls() {
            const bind = (id, fn) => {
                const el = document.getElementById(id);
                el.addEventListener('touchstart', (e) => { e.preventDefault(); fn(); });
                el.addEventListener('mousedown', (e) => { e.preventDefault(); fn(); });
            };

            bind('btn-jump', () => { if(localPlayer.isGrounded) localPlayer.velocity = 0.15; });
            bind('btn-swap', () => {
                localPlayer.currentTool = localPlayer.currentTool === 'pickaxe' ? 'gun' : 'pickaxe';
                document.getElementById('btn-shoot').innerText = localPlayer.currentTool === 'pickaxe' ? '⛏️' : '🎯';
                weaponGroup.visible = (localPlayer.currentTool === 'gun');
            });
            bind('btn-build', buildWall);
            bind('btn-shoot', handleAction);

            const joy = document.getElementById('joystick-container');
            const knob = document.getElementById('joystick-knob');
            joy.addEventListener('touchmove', (e) => {
                e.preventDefault();
                const t = e.touches[0];
                const r = joy.getBoundingClientRect();
                const dx = t.clientX - (r.left + r.width/2);
                const dy = t.clientY - (r.top + r.height/2);
                const dist = Math.min(Math.sqrt(dx*dx+dy*dy), 40);
                const angle = Math.atan2(dy, dx);
                knob.style.transform = `translate(${Math.cos(angle)*dist}px, ${Math.sin(angle)*dist}px)`;
                moveDir = { x: (Math.cos(angle)*dist)/40, z: (Math.sin(angle)*dist)/40 };
            });
            joy.addEventListener('touchend', () => {
                knob.style.transform = `translate(0,0)`;
                moveDir = { x: 0, z: 0 };
            });

            let lastTouch = null;
            document.addEventListener('touchstart', (e) => { if(!e.target.closest('.ui-active')) lastTouch = { x: e.touches[0].clientX, y: e.touches[0].clientY }; });
            document.addEventListener('touchmove', (e) => {
                if(!lastTouch || e.target.closest('.ui-active')) return;
                const dx = e.touches[0].clientX - lastTouch.x;
                const dy = e.touches[0].clientY - lastTouch.y;
                camera.rotation.y -= dx * 0.005;
                camera.rotation.x -= dy * 0.005;
                camera.rotation.x = Math.max(-1.4, Math.min(1.4, camera.rotation.x));
                lastTouch = { x: e.touches[0].clientX, y: e.touches[0].clientY };
            });
        }

        async function handleAction() {
            const flash = document.createElement('div');
            flash.className = 'flash-effect';
            document.body.appendChild(flash);
            setTimeout(() => flash.remove(), 50);

            raycaster.setFromCamera({x:0, y:0}, camera);
            
            if (localPlayer.currentTool === 'pickaxe') {
                trees.forEach(tree => {
                    if (camera.position.distanceTo(tree.position) < 7) {
                        localPlayer.wood += 10;
                        document.getElementById('wood-val').innerText = localPlayer.wood;
                    }
                });
            } else {
                const targetMeshes = [...Object.values(remotePlayers), ...Object.values(wallObjects)];
                const hits = raycaster.intersectObjects(targetMeshes);
                
                if (hits.length > 0) {
                    const obj = hits[0].object;
                    const data = obj.userData;

                    if (data.type === 'wall' && db) {
                        const wallRef = doc(db, 'artifacts', appId, 'public', 'data', 'walls', data.id);
                        const currentHp = data.hp || 100;
                        const newHp = currentHp - 10;

                        obj.material.color.set(0xffffff);
                        setTimeout(() => { if(obj.material) obj.material.color.set(0x8B4513); }, 50);

                        if (newHp <= 0) {
                            await deleteDoc(wallRef);
                        } else {
                            await updateDoc(wallRef, { hp: newHp });
                        }
                    } else if (data.type === 'player' && db) {
                        // Causar dano a outro jogador
                        const playerRef = doc(db, 'artifacts', appId, 'public', 'data', 'players', data.id);
                        // Aqui o servidor ou o cliente atingido processaria o dano. 
                        // Simplificando: enviamos uma redução de HP direta.
                        await updateDoc(playerRef, { hp: Math.max(0, (remotePlayers[data.id].userData.hp || 100) - 20) });
                    }
                }
            }
        }

        async function buildWall() {
            if (localPlayer.wood < 10) return;
            localPlayer.wood -= 10;
            document.getElementById('wood-val').innerText = localPlayer.wood;
            
            const dir = new THREE.Vector3(0,0,-5).applyQuaternion(camera.quaternion);
            const pos = new THREE.Vector3().copy(camera.position).add(dir);
            
            if (db) {
                const wallsCol = collection(db, 'artifacts', appId, 'public', 'data', 'walls');
                await setDoc(doc(wallsCol), {
                    x: pos.x, y: 1.5, z: pos.z, ry: camera.rotation.y, hp: 100
                });
            }
        }

        function animate() {
            requestAnimationFrame(animate);
            if (moveDir.x || moveDir.z) {
                const forward = new THREE.Vector3(0,0,-1).applyQuaternion(new THREE.Quaternion().setFromEuler(new THREE.Euler(0, camera.rotation.y, 0)));
                const right = new THREE.Vector3(1,0,0).applyQuaternion(new THREE.Quaternion().setFromEuler(new THREE.Euler(0, camera.rotation.y, 0)));
                camera.position.addScaledVector(forward, -moveDir.z * 0.2);
                camera.position.addScaledVector(right, moveDir.x * 0.2);
            }
            localPlayer.velocity -= 0.008;
            camera.position.y += localPlayer.velocity;
            if (camera.position.y <= 1.8) { camera.position.y = 1.8; localPlayer.velocity = 0; localPlayer.isGrounded = true; }
            renderer.render(scene, camera);
        }

        window.addEventListener('load', () => {
            const btn = document.getElementById('start-overlay');
            btn.addEventListener('click', () => {
                btn.style.display = 'none';
                document.getElementById('ui-layer').style.display = 'block';
                initGame();
                updateHpUI();
            });
        });
    </script>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Segoe UI', Roboto, Helvetica, Arial, sans-serif; user-select: none; }
        #start-overlay { position: fixed; inset: 0; background: #6d28d9; color: white; display: flex; flex-direction: column; align-items: center; justify-content: center; z-index: 999; cursor: pointer; }
        #ui-layer { position: fixed; inset: 0; pointer-events: none; display: none; }
        .ui-active { pointer-events: auto; }
        
        /* Stats & HP */
        #stats-container { position: fixed; top: 20px; left: 20px; display: flex; flex-direction: column; gap: 10px; pointer-events: none; }
        .stat-box { background: rgba(0,0,0,0.6); padding: 8px 15px; border-radius: 5px; color: #fff; font-weight: bold; border-left: 4px solid #fcd34d; }
        #hp-container { width: 200px; height: 25px; background: rgba(0,0,0,0.6); border-radius: 5px; overflow: hidden; position: relative; border: 2px solid rgba(255,255,255,0.2); }
        #hp-bar-fill { width: 100%; height: 100%; background: #4ade80; transition: width 0.3s ease, background 0.3s ease; }
        #hp-text { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; color: white; font-size: 12px; font-weight: bold; text-shadow: 1px 1px 2px #000; }

        #joystick-container { position: absolute; bottom: 40px; left: 40px; width: 100px; height: 100px; background: rgba(255,255,255,0.2); border-radius: 50%; border: 2px solid #fff; }
        #joystick-knob { position: absolute; top: 30px; left: 30px; width: 40px; height: 40px; background: #fff; border-radius: 50%; }
        #action-buttons { position: absolute; bottom: 40px; right: 40px; display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
        #action-buttons button { width: 70px; height: 70px; border-radius: 50%; border: 2px solid #fff; background: rgba(0,0,0,0.5); color: #fff; font-size: 24px; outline: none; }
        #btn-shoot { grid-column: span 2; width: 100% !important; height: 80px !important; border-radius: 40px !important; background: #ef4444 !important; }
        #crosshair { position: fixed; top: 50%; left: 50%; width: 20px; height: 20px; border: 2px solid #fff; border-radius: 50%; transform: translate(-50%,-50%); pointer-events: none; }
        #crosshair::before { content: ''; position: absolute; top: 50%; left: 50%; width: 2px; height: 2px; background: white; transform: translate(-50%,-50%); }
        .flash-effect { position: fixed; inset: 0; background: #fff; opacity: 0.2; z-index: 1000; pointer-events: none; }
    </style>
</head>
<body>
    <div id="start-overlay">
        <h1>FORTNITE MOBILE</h1>
        <p>TOQUE PARA INICIAR</p>
    </div>
    <div id="crosshair"></div>
    
    <div id="stats-container">
        <div id="hp-container">
            <div id="hp-bar-fill"></div>
            <div id="hp-text">HP: <span id="hp-val">100</span> / 100</div>
        </div>
        <div class="stat-box">MADEIRA: <span id="wood-val">50</span></div>
    </div>

    <div id="ui-layer">
        <div id="joystick-container" class="ui-active"><div id="joystick-knob"></div></div>
        <div id="action-buttons" class="ui-active">
            <button id="btn-shoot">⛏️</button>
            <button id="btn-build">🧱</button>
            <button id="btn-swap">🔄</button>
            <button id="btn-jump">⬆️</button>
        </div>
    </div>
</body>
</html>
