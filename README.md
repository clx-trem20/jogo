<!DOCTYPE html>
<html lang="pt-PT">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Fortnite Mobile Multiplayer</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getFirestore, doc, setDoc, onSnapshot, collection, deleteDoc, updateDoc, query, where } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";

        // Configuração de ambiente
        const firebaseConfig = JSON.parse(__firebase_config);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'fortnite-mobile';
        
        let db, auth;
        try {
            const app = initializeApp(firebaseConfig);
            db = getFirestore(app);
            auth = getAuth(app);
        } catch (e) {
            console.error("Erro Firebase Init:", e);
        }

        let scene, camera, renderer, clock, raycaster;
        let localPlayer = {
            id: "local-" + Math.random().toString(36).substr(2, 9),
            hp: 100,
            wood: 50,
            kills: 0,
            pos: { x: Math.random() * 80 - 40, y: 1.8, z: Math.random() * 80 - 40 },
            isGrounded: true,
            velocity: 0,
            currentTool: 'pickaxe' // 'pickaxe' ou 'gun'
        };
        
        const remotePlayers = {};
        const wallObjects = {}; // Mapear por ID para facilitar remoção
        const trees = [];
        let moveDir = { x: 0, z: 0 };
        let isStarted = false;
        let weaponGroup;

        async function init() {
            const startBtn = document.getElementById('start-overlay');
            startBtn.addEventListener('click', async () => {
                startBtn.style.display = 'none';
                if (!isStarted) {
                    isStarted = true;
                    startApp();
                }
                if (auth) {
                    try {
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
                    } catch (error) {
                        console.warn("Modo Offline", error);
                    }
                }
            });
        }

        function startApp() {
            setupThreeJS();
            generateEnvironment();
            createWeaponModel();
            setupMobileControls();
            animate();
            
            window.addEventListener('beforeunload', async () => {
                if (auth && auth.currentUser) {
                    try {
                        await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'players', localPlayer.id));
                    } catch(e) {}
                }
            });
        }

        function setupThreeJS() {
            if (renderer) return;
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x87ceeb);
            scene.fog = new THREE.Fog(0x87ceeb, 0, 150);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(localPlayer.pos.x, 1.8, localPlayer.pos.z);
            camera.rotation.order = 'YXZ';

            raycaster = new THREE.Raycaster();

            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
            document.body.appendChild(renderer.domElement);

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
        }

        function createWeaponModel() {
            weaponGroup = new THREE.Group();
            
            const bodyGeo = new THREE.BoxGeometry(0.15, 0.3, 1.2);
            const bodyMat = new THREE.MeshPhongMaterial({ color: 0x8B7355 });
            const body = new THREE.Mesh(bodyGeo, bodyMat);
            
            const barrelGeo = new THREE.CylinderGeometry(0.04, 0.04, 0.8);
            const barrelMat = new THREE.MeshPhongMaterial({ color: 0x222222 });
            const barrel = new THREE.Mesh(barrelGeo, barrelMat);
            barrel.rotation.x = Math.PI / 2;
            barrel.position.z = -0.8;
            
            const handleGeo = new THREE.BoxGeometry(0.1, 0.4, 0.2);
            const handle = new THREE.Mesh(handleGeo, bodyMat);
            handle.position.y = -0.3;
            handle.position.z = 0.2;

            weaponGroup.add(body, barrel, handle);
            weaponGroup.position.set(0.4, -0.4, -0.6);
            camera.add(weaponGroup);
            scene.add(camera);
        }

        function generateEnvironment() {
            const treeTrunkGeo = new THREE.CylinderGeometry(0.5, 0.7, 4, 8);
            const treeTrunkMat = new THREE.MeshPhongMaterial({ color: 0x4d2926 });
            const treeLeavesGeo = new THREE.ConeGeometry(3, 6, 8);
            const treeLeavesMat = new THREE.MeshPhongMaterial({ color: 0x228B22 });

            for (let i = 0; i < 50; i++) {
                const group = new THREE.Group();
                const x = Math.random() * 300 - 150;
                const z = Math.random() * 300 - 150;
                const trunk = new THREE.Mesh(treeTrunkGeo, treeTrunkMat);
                trunk.position.y = 2;
                group.add(trunk);
                const leaves = new THREE.Mesh(treeLeavesGeo, treeLeavesMat);
                leaves.position.y = 6;
                group.add(leaves);
                group.position.set(x, 0, z);
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
                        if (data.hp <= 0) respawn();
                        return;
                    }
                    if (change.type === "added" || change.type === "modified") {
                        updateRemotePlayer(change.doc.id, data);
                    } else if (change.type === "removed") {
                        removeRemotePlayer(change.doc.id);
                    }
                });
            });

            onSnapshot(wallsRef, (snapshot) => {
                snapshot.docChanges().forEach(change => {
                    const id = change.doc.id;
                    const data = change.doc.data();
                    if (change.type === "added" || change.type === "modified") {
                        if (data.hp <= 0) {
                            removeWallMesh(id);
                        } else {
                            createWallMesh(id, data);
                        }
                    } else if (change.type === "removed") {
                        removeWallMesh(id);
                    }
                });
            });

            setInterval(async () => {
                if (auth.currentUser) {
                    try {
                        await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'players', localPlayer.id), {
                            pos: { x: camera.position.x, y: camera.position.y, z: camera.position.z },
                            rot: { y: camera.rotation.y },
                            hp: localPlayer.hp,
                            lastUpdate: Date.now()
                        }, { merge: true });
                    } catch (e) {}
                }
            }, 100);
        }

        function updateRemotePlayer(id, data) {
            if (!remotePlayers[id]) {
                const geo = new THREE.BoxGeometry(1.2, 2.2, 1.2);
                const mat = new THREE.MeshPhongMaterial({ color: 0xff0000 });
                const mesh = new THREE.Mesh(geo, mat);
                mesh.userData.id = id;
                mesh.userData.type = 'player';
                scene.add(mesh);
                remotePlayers[id] = mesh;
            }
            remotePlayers[id].position.set(data.pos.x, data.pos.y, data.pos.z);
            remotePlayers[id].rotation.y = data.rot ? data.rot.y : 0;
        }

        function removeRemotePlayer(id) {
            if (remotePlayers[id]) {
                scene.remove(remotePlayers[id]);
                delete remotePlayers[id];
            }
        }

        function respawn() {
            localPlayer.hp = 100;
            camera.position.set(Math.random()*40-20, 1.8, Math.random()*40-20);
            document.getElementById('hp-val').innerText = "100";
        }

        function createWallMesh(id, data) {
            if (wallObjects[id]) return;
            const geo = new THREE.BoxGeometry(4, 3, 0.4);
            const mat = new THREE.MeshPhongMaterial({ color: 0x8B4513 });
            const wall = new THREE.Mesh(geo, mat);
            wall.position.set(data.x, data.y, data.z);
            wall.rotation.set(data.rx, data.ry, data.rz);
            wall.userData.id = id;
            wall.userData.type = 'wall';
            wall.userData.hp = data.hp || 100;
            scene.add(wall);
            wallObjects[id] = wall;
        }

        function removeWallMesh(id) {
            if (wallObjects[id]) {
                scene.remove(wallObjects[id]);
                delete wallObjects[id];
            }
        }

        function setupMobileControls() {
            const ui = document.getElementById('ui-layer');
            ui.style.display = 'block';

            const bindAction = (id, fn) => {
                const el = document.getElementById(id);
                const trigger = (e) => { e.preventDefault(); e.stopPropagation(); fn(); };
                el.addEventListener('touchstart', trigger, { passive: false });
                el.addEventListener('mousedown', trigger);
            };

            bindAction('btn-jump', () => { if(localPlayer.isGrounded) localPlayer.velocity = 0.16; });
            bindAction('btn-build', buildWall);
            bindAction('btn-shoot', handleAction);
            bindAction('btn-swap', swapTool);

            const container = document.getElementById('joystick-container');
            const knob = document.getElementById('joystick-knob');
            container.addEventListener('touchstart', (e) => {
                const move = (me) => {
                    me.preventDefault();
                    const t = me.touches[0];
                    const r = container.getBoundingClientRect();
                    let dx = t.clientX - (r.left + r.width/2);
                    let dy = t.clientY - (r.top + r.height/2);
                    const d = Math.min(Math.sqrt(dx*dx+dy*dy), 45);
                    const ang = Math.atan2(dy, dx);
                    dx = Math.cos(ang) * d; dy = Math.sin(ang) * d;
                    knob.style.transform = `translate(${dx}px, ${dy}px)`;
                    moveDir = { x: dx/45, z: dy/45 };
                };
                const end = () => {
                    knob.style.transform = `translate(0,0)`;
                    moveDir = { x: 0, z: 0 };
                    window.removeEventListener('touchmove', move);
                    window.removeEventListener('touchend', end);
                };
                window.addEventListener('touchmove', move, {passive:false});
                window.addEventListener('touchend', end);
            });

            let lastT = null;
            document.addEventListener('touchstart', (e) => {
                if(e.target.closest('.ui-active')) return;
                lastT = { x: e.touches[0].clientX, y: e.touches[0].clientY };
            });
            document.addEventListener('touchmove', (e) => {
                if(!lastT || e.target.closest('.ui-active')) return;
                const dx = e.touches[0].clientX - lastT.x;
                const dy = e.touches[0].clientY - lastT.y;
                camera.rotation.y -= dx * 0.006;
                camera.rotation.x -= dy * 0.006;
                camera.rotation.x = Math.max(-1.5, Math.min(1.5, camera.rotation.x));
                lastT = { x: e.touches[0].clientX, y: e.touches[0].clientY };
            });
        }

        function swapTool() {
            localPlayer.currentTool = localPlayer.currentTool === 'pickaxe' ? 'gun' : 'pickaxe';
            const btn = document.getElementById('btn-shoot');
            btn.innerText = localPlayer.currentTool === 'pickaxe' ? '⛏️' : '🎯';
            weaponGroup.visible = (localPlayer.currentTool === 'gun');
        }

        function handleAction() {
            if (localPlayer.currentTool === 'pickaxe') {
                shootOrChop('pickaxe');
            } else {
                shootOrChop('gun');
            }
        }

        async function shootOrChop(type) {
            const flash = document.createElement('div');
            flash.className = "flash-effect";
            document.body.appendChild(flash);
            setTimeout(() => flash.remove(), 40);

            if (type === 'gun') {
                weaponGroup.position.z += 0.1;
                setTimeout(() => weaponGroup.position.z -= 0.1, 50);
            }

            raycaster.setFromCamera({x: 0, y: 0}, camera);
            
            if (type === 'pickaxe') {
                trees.forEach(tree => {
                    if (camera.position.distanceTo(tree.position) < 8) {
                        localPlayer.wood += 5;
                        document.getElementById('wood-val').innerText = localPlayer.wood;
                    }
                });
                return;
            }

            // Alvos: Jogadores + Paredes
            const targets = [...Object.values(remotePlayers), ...Object.values(wallObjects)];
            const intersects = raycaster.intersectObjects(targets);
            
            if (intersects.length > 0) {
                const target = intersects[0].object;
                const targetId = target.userData.id;

                if (target.userData.type === 'player') {
                    if (db) {
                        try {
                            const enemyRef = doc(db, 'artifacts', appId, 'public', 'data', 'players', targetId);
                            await setDoc(enemyRef, { hp: 0 }, { merge: true });
                            localPlayer.kills++;
                        } catch(e) {}
                    }
                } 
                else if (target.userData.type === 'wall') {
                    // Cada tiro tira 10 de vida. 10 tiros = 100 HP.
                    if (db) {
                        try {
                            const wallRef = doc(db, 'artifacts', appId, 'public', 'data', 'walls', targetId);
                            const newHp = (target.userData.hp || 100) - 10;
                            
                            if (newHp <= 0) {
                                await deleteDoc(wallRef);
                            } else {
                                target.userData.hp = newHp;
                                await updateDoc(wallRef, { hp: newHp });
                                // Feedback visual de impacto na parede
                                target.material.color.set(0xffffff);
                                setTimeout(() => target.material.color.set(0x8B4513), 50);
                            }
                        } catch(e) {}
                    }
                }
            }
        }

        async function buildWall() {
            if (localPlayer.wood < 10) return;
            localPlayer.wood -= 10;
            document.getElementById('wood-val').innerText = localPlayer.wood;
            const dir = new THREE.Vector3(0,0,-1).applyQuaternion(camera.quaternion);
            const pos = new THREE.Vector3().copy(camera.position).add(dir.multiplyScalar(5));
            
            const wallData = { 
                x: pos.x, y: 1.5, z: pos.z, 
                rx: 0, ry: camera.rotation.y, rz: 0,
                hp: 100, // Vida da parede
                owner: localPlayer.id,
                timestamp: Date.now()
            };
            
            if (db) {
                await setDoc(doc(collection(db, 'artifacts', appId, 'public', 'data', 'walls')), wallData);
            } else {
                createWallMesh("local-"+Date.now(), wallData);
            }
        }

        function animate() {
            requestAnimationFrame(animate);
            if (!isStarted) return;
            
            if (moveDir.x !== 0 || moveDir.z !== 0) {
                const speed = 0.22;
                const forward = new THREE.Vector3(0, 0, -1).applyQuaternion(new THREE.Quaternion().setFromEuler(new THREE.Euler(0, camera.rotation.y, 0)));
                const right = new THREE.Vector3(1, 0, 0).applyQuaternion(new THREE.Quaternion().setFromEuler(new THREE.Euler(0, camera.rotation.y, 0)));
                camera.position.addScaledVector(forward, -moveDir.z * speed);
                camera.position.addScaledVector(right, moveDir.x * speed);
            }

            localPlayer.velocity -= 0.008;
            camera.position.y += localPlayer.velocity;
            if (camera.position.y <= 1.8) {
                camera.position.y = 1.8;
                localPlayer.velocity = 0;
                localPlayer.isGrounded = true;
            }
            
            if(weaponGroup) {
                weaponGroup.rotation.y = Math.sin(Date.now()*0.005)*0.015;
            }

            renderer.render(scene, camera);
        }

        window.onload = init;
    </script>
    <style>
        body { margin: 0; overflow: hidden; background: #000; touch-action: none; font-family: 'Arial Black', sans-serif; }
        #start-overlay { position: fixed; inset: 0; background: radial-gradient(circle, #6d28d9, #1e1b4b); color: white; display: flex; flex-direction: column; align-items: center; justify-content: center; z-index: 20000; cursor: pointer; }
        #ui-layer { position: fixed; inset: 0; pointer-events: none; z-index: 10000; display: none; }
        .ui-active { pointer-events: auto !important; }
        #joystick-container { position: absolute; bottom: 50px; left: 50px; width: 120px; height: 120px; background: rgba(255,255,255,0.15); border: 2px solid white; border-radius: 50%; }
        #joystick-knob { position: absolute; top: 35px; left: 35px; width: 50px; height: 50px; background: white; border-radius: 50%; box-shadow: 0 0 15px rgba(255,255,255,0.5); }
        #action-buttons { position: absolute; bottom: 50px; right: 50px; display: grid; grid-template-columns: 1fr 1fr; gap: 15px; }
        #action-buttons button { width: 80px; height: 80px; border-radius: 50%; border: 3px solid white; background: rgba(0,0,0,0.7); color: white; font-size: 30px; }
        #btn-shoot { grid-column: span 2; width: 100% !important; height: 90px !important; border-radius: 45px !important; background: rgba(220,38,38,0.8) !important; }
        #crosshair { position: fixed; top: 50%; left: 50%; width: 24px; height: 24px; border: 2px solid rgba(255,255,255,0.8); border-radius: 50%; transform: translate(-50%, -50%); pointer-events: none; }
        #crosshair::after { content: ''; position: absolute; top: 50%; left: 50%; width: 4px; height: 4px; background: white; border-radius: 50%; transform: translate(-50%,-50%); }
        #stats { position: fixed; top: 20px; left: 20px; color: white; background: rgba(0,0,0,0.6); padding: 15px; border-radius: 8px; border-left: 5px solid #fcd34d; font-size: 14px; }
        .flash-effect { position: fixed; inset: 0; background: white; opacity: 0.1; pointer-events: none; z-index: 15000; }
        @keyframes pulse { 0% { opacity: 0.8; } 50% { opacity: 1; transform: scale(1.05); } 100% { opacity: 0.8; } }
    </style>
</head>
<body>
    <div id="start-overlay">
        <h1 style="font-size: 4rem; text-shadow: 0 5px 15px rgba(0,0,0,0.5); margin:0;">FORTNITE</h1>
        <p style="letter-spacing: 5px; animation: pulse 1.5s infinite;">CLICA PARA JOGAR</p>
    </div>
    <div id="crosshair"></div>
    <div id="stats">
        <div style="color: #fcd34d;">MADEIRA: <span id="wood-val">50</span></div>
        <div>VIDA: <span id="hp-val" style="color: #4ade80;">100</span></div>
    </div>
    <div id="ui-layer">
        <div id="joystick-container" class="ui-active">
            <div id="joystick-knob"></div>
        </div>
        <div id="action-buttons" class="ui-active">
            <button id="btn-shoot">⛏️</button>
            <button id="btn-build">🧱</button>
            <button id="btn-swap">🔄</button>
            <button id="btn-jump">⬆️</button>
        </div>
    </div>
</body>
</html>
