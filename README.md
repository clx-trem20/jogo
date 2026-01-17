<!DOCTYPE html>
<html lang="pt-PT">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Fortnite Mobile Multiplayer</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-app.js";
        import { getFirestore, doc, setDoc, onSnapshot, collection, deleteDoc } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-firestore.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.1.0/firebase-auth.js";

        const firebaseConfig = JSON.parse(__firebase_config);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'fortnite-mobile';
        
        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);
        const auth = getAuth(app);

        let scene, camera, renderer, clock;
        let localPlayer = {
            id: null,
            hp: 100,
            wood: 50,
            pos: { x: Math.random() * 80 - 40, y: 1.8, z: Math.random() * 80 - 40 },
            isGrounded: true,
            velocity: 0
        };
        
        const remotePlayers = {};
        const wallObjects = {};
        const trees = [];
        let moveDir = { x: 0, z: 0 };
        let isStarted = false;

        async function init() {
            const startBtn = document.getElementById('start-overlay');
            startBtn.addEventListener('click', async () => {
                startBtn.style.display = 'none';
                isStarted = true;
                
                try {
                    if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                        await signInWithCustomToken(auth, __initial_auth_token);
                    } else {
                        await signInAnonymously(auth);
                    }

                    onAuthStateChanged(auth, (user) => {
                        if (user && !localPlayer.id) {
                            localPlayer.id = user.uid;
                            setTimeout(startApp, 500);
                        }
                    });
                } catch (error) {
                    console.error("Erro na autenticação:", error);
                }
            });
        }

        function startApp() {
            setupThreeJS();
            generateEnvironment();
            setupMobileControls();
            setupNetworking();
            animate();
            
            window.addEventListener('beforeunload', async () => {
                if (auth.currentUser && localPlayer.id) {
                    try {
                        const playerDoc = doc(db, 'artifacts', appId, 'public', 'data', 'players', localPlayer.id);
                        await deleteDoc(playerDoc);
                    } catch(e) {}
                }
            });
        }

        function setupThreeJS() {
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x87ceeb);
            scene.fog = new THREE.Fog(0x87ceeb, 0, 150);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(localPlayer.pos.x, 1.8, localPlayer.pos.z);
            camera.rotation.order = 'YXZ';

            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
            document.body.appendChild(renderer.domElement);

            clock = new THREE.Clock();

            const sun = new THREE.DirectionalLight(0xffffff, 1);
            sun.position.set(50, 100, 50);
            scene.add(sun);
            scene.add(new THREE.AmbientLight(0x404040, 1.2));

            const ground = new THREE.Mesh(
                new THREE.PlaneGeometry(1000, 1000),
                new THREE.MeshPhongMaterial({ color: 0x348C31 })
            );
            ground.rotation.x = -Math.PI / 2;
            scene.add(ground);
        }

        function generateEnvironment() {
            const treeTrunkGeo = new THREE.CylinderGeometry(0.5, 0.7, 4, 8);
            const treeTrunkMat = new THREE.MeshPhongMaterial({ color: 0x4d2926 });
            const treeLeavesGeo = new THREE.ConeGeometry(3, 6, 8);
            const treeLeavesMat = new THREE.MeshPhongMaterial({ color: 0x228B22 });

            for (let i = 0; i < 40; i++) {
                const group = new THREE.Group();
                const x = Math.random() * 200 - 100;
                const z = Math.random() * 200 - 100;
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
            if (!auth.currentUser) return;
            const playersRef = collection(db, 'artifacts', appId, 'public', 'data', 'players');
            const wallsRef = collection(db, 'artifacts', appId, 'public', 'data', 'walls');

            onSnapshot(playersRef, (snapshot) => {
                snapshot.docChanges().forEach(change => {
                    const data = change.doc.data();
                    if (change.doc.id === localPlayer.id) return;
                    if (change.type === "added" || change.type === "modified") {
                        updateRemotePlayer(change.doc.id, data);
                    } else if (change.type === "removed") {
                        removeRemotePlayer(change.doc.id);
                    }
                });
            }, (error) => {});

            onSnapshot(wallsRef, (snapshot) => {
                snapshot.docChanges().forEach(change => {
                    if (change.type === "added") createWallMesh(change.doc.id, change.doc.data());
                });
            }, (error) => {});

            setInterval(async () => {
                if (auth.currentUser && localPlayer.id) {
                    try {
                        await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'players', localPlayer.id), {
                            pos: { x: camera.position.x, y: camera.position.y, z: camera.position.z },
                            rot: { y: camera.rotation.y },
                            hp: localPlayer.hp,
                            lastUpdate: Date.now()
                        }, { merge: true });
                    } catch (e) {}
                }
            }, 300);
        }

        function updateRemotePlayer(id, data) {
            if (!data.pos) return;
            if (!remotePlayers[id]) {
                const geo = new THREE.BoxGeometry(1, 2, 1);
                const mat = new THREE.MeshPhongMaterial({ color: 0xff4444 });
                const mesh = new THREE.Mesh(geo, mat);
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

        function createWallMesh(id, data) {
            if (wallObjects[id]) return;
            const geo = new THREE.BoxGeometry(4, 3, 0.5);
            const mat = new THREE.MeshPhongMaterial({ color: 0x8B4513 });
            const wall = new THREE.Mesh(geo, mat);
            wall.position.set(data.x, data.y, data.z);
            wall.rotation.set(data.rx, data.ry, data.rz);
            scene.add(wall);
            wallObjects[id] = wall;
        }

        function setupMobileControls() {
            const ui = document.getElementById('ui-layer');
            ui.style.display = 'block';

            const container = document.getElementById('joystick-container');
            const knob = document.getElementById('joystick-knob');
            
            const handleJoystick = (e) => {
                e.preventDefault();
                e.stopPropagation();
                
                const touch = e.touches ? e.touches[0] : e;
                const rect = container.getBoundingClientRect();
                const centerX = rect.left + rect.width / 2;
                const centerY = rect.top + rect.height / 2;
                
                let dx = touch.clientX - centerX;
                let dy = touch.clientY - centerY;
                const dist = Math.sqrt(dx*dx + dy*dy);
                const maxDist = 45;

                if (dist > maxDist) {
                    dx *= maxDist / dist;
                    dy *= maxDist / dist;
                }

                knob.style.transform = `translate(${dx}px, ${dy}px)`;
                moveDir.x = dx / maxDist;
                moveDir.z = dy / maxDist;
            };

            const resetJoystick = (e) => {
                e.preventDefault();
                knob.style.transform = `translate(0px, 0px)`;
                moveDir = { x: 0, z: 0 };
            };

            container.addEventListener('touchstart', handleJoystick, { passive: false });
            container.addEventListener('touchmove', handleJoystick, { passive: false });
            container.addEventListener('touchend', resetJoystick, { passive: false });

            // Câmara
            let lastTouch = null;
            document.addEventListener('touchstart', (e) => {
                if (e.target.closest('.ui-active')) return;
                lastTouch = { x: e.touches[0].clientX, y: e.touches[0].clientY };
            }, { passive: false });

            document.addEventListener('touchmove', (e) => {
                if (e.target.closest('.ui-active')) return;
                if (!lastTouch) return;
                const touch = e.touches[0];
                const dx = touch.clientX - lastTouch.x;
                const dy = touch.clientY - lastTouch.y;
                camera.rotation.y -= dx * 0.005;
                camera.rotation.x -= dy * 0.005;
                camera.rotation.x = Math.max(-Math.PI/2.2, Math.min(Math.PI/2.2, camera.rotation.x));
                lastTouch = { x: touch.clientX, y: touch.clientY };
            }, { passive: false });

            document.addEventListener('touchend', () => lastTouch = null);

            // Ações com suporte duplo
            const bindAction = (id, fn) => {
                const el = document.getElementById(id);
                const trigger = (e) => {
                    e.preventDefault();
                    e.stopPropagation();
                    fn();
                };
                el.addEventListener('touchstart', trigger, { passive: false });
                el.addEventListener('mousedown', trigger);
            };

            bindAction('btn-jump', () => { if(localPlayer.isGrounded) localPlayer.velocity = 0.15; });
            bindAction('btn-build', buildWall);
            bindAction('btn-shoot', shootOrChop);
        }

        async function buildWall() {
            if (!auth.currentUser || !localPlayer.id || localPlayer.wood < 10) return;
            localPlayer.wood -= 10;
            document.getElementById('wood-val').innerText = localPlayer.wood;

            const vector = new THREE.Vector3(0, 0, -4);
            vector.applyQuaternion(camera.quaternion);
            const pos = new THREE.Vector3().copy(camera.position).add(vector);

            try {
                const wallsCol = collection(db, 'artifacts', appId, 'public', 'data', 'walls');
                await setDoc(doc(wallsCol), {
                    x: pos.x, y: 1.5, z: pos.z,
                    rx: 0, ry: camera.rotation.y, rz: 0,
                    owner: localPlayer.id,
                    timestamp: Date.now()
                });
            } catch (e) {}
        }

        function shootOrChop() {
            const flash = document.createElement('div');
            flash.style = "position:fixed; top:50%; left:50%; width:10px; height:10px; background:white; border-radius:50%; transform:translate(-50%,-50%); pointer-events:none; z-index:10000;";
            document.body.appendChild(flash);
            setTimeout(() => flash.remove(), 50);

            trees.forEach(tree => {
                const dist = camera.position.distanceTo(tree.position);
                if (dist < 8) {
                    localPlayer.wood += 5;
                    document.getElementById('wood-val').innerText = localPlayer.wood;
                }
            });
        }

        function animate() {
            requestAnimationFrame(animate);
            if (!isStarted) return;
            
            if (moveDir.x !== 0 || moveDir.z !== 0) {
                const forward = new THREE.Vector3(0, 0, -1).applyQuaternion(new THREE.Quaternion().setFromEuler(new THREE.Euler(0, camera.rotation.y, 0)));
                const right = new THREE.Vector3(1, 0, 0).applyQuaternion(new THREE.Quaternion().setFromEuler(new THREE.Euler(0, camera.rotation.y, 0)));
                camera.position.addScaledVector(forward, -moveDir.z * 0.18);
                camera.position.addScaledVector(right, moveDir.x * 0.18);
            }

            localPlayer.velocity -= 0.008;
            camera.position.y += localPlayer.velocity;
            if (camera.position.y <= 1.8) {
                camera.position.y = 1.8;
                localPlayer.velocity = 0;
                localPlayer.isGrounded = true;
            } else {
                localPlayer.isGrounded = false;
            }
            renderer.render(scene, camera);
        }

        window.onload = init;
    </script>
    <style>
        body { margin: 0; overflow: hidden; background: #000; touch-action: none; -webkit-user-select: none; font-family: sans-serif; }
        
        #start-overlay {
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            background: linear-gradient(45deg, #4c1d95, #1e3a8a); color: white; display: flex;
            flex-direction: column; align-items: center; justify-content: center;
            z-index: 20000; cursor: pointer; text-align: center;
        }

        #ui-layer { 
            position: fixed; top: 0; left: 0; width: 100%; height: 100%; 
            pointer-events: none; z-index: 10000; display: none;
        }

        .ui-active { pointer-events: auto !important; }

        #joystick-container { 
            position: absolute; bottom: 60px; left: 60px; width: 130px; height: 130px; 
            background: rgba(255,255,255,0.1); border: 3px solid rgba(255,255,255,0.4);
            border-radius: 50%; touch-action: none;
        }
        
        #joystick-knob {
            position: absolute; top: 40px; left: 40px; width: 50px; height: 50px;
            background: #fff; border-radius: 50%; pointer-events: none;
        }

        #action-buttons {
            position: absolute; bottom: 60px; right: 60px; 
            display: flex; flex-direction: column; gap: 20px;
        }

        #action-buttons button {
            width: 85px; height: 85px; border-radius: 50%; border: 3px solid white; 
            background: rgba(0,0,0,0.8); color: white; font-size: 34px;
            touch-action: manipulation;
        }
        
        #crosshair { 
            position: fixed; top: 50%; left: 50%; width: 20px; height: 20px; 
            border: 2px solid white; border-radius: 50%; 
            transform: translate(-50%, -50%); pointer-events: none; z-index: 5000; 
        }

        #stats { 
            position: fixed; top: 20px; left: 20px; color: white; 
            background: rgba(0,0,0,0.7); padding: 10px 20px; border-radius: 10px; 
            font-weight: bold; z-index: 10000; pointer-events: none;
        }
    </style>
</head>
<body>
    <div id="start-overlay">
        <h1>FORTNITE MOBILE</h1>
        <p>TOCA PARA JOGAR</p>
    </div>
    <div id="crosshair"></div>
    <div id="stats">
        MADEIRA: <span id="wood-val">50</span> | HP: 100
    </div>
    <div id="ui-layer">
        <div id="joystick-container" class="ui-active">
            <div id="joystick-knob"></div>
        </div>
        <div id="action-buttons" class="ui-active">
            <button id="btn-shoot">⛏️</button>
            <button id="btn-build">🧱</button>
            <button id="btn-jump">⬆️</button>
        </div>
    </div>
</body>
</html>
