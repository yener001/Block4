<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>JS-Craft Pro 3D</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            user-select: none;
        }
        #canvas-container {
            width: 100vw;
            height: 100vh;
        }
        /* Crosshair / Hedef Noktası */
        #crosshair {
            position: absolute;
            top: 50%;
            left: 50%;
            width: 10px;
            height: 10px;
            transform: translate(-50%, -50%);
            pointer-events: none;
        }
        #crosshair::before, #crosshair::after {
            content: '';
            position: absolute;
            background: white;
            opacity: 0.8;
        }
        #crosshair::before { top: 4px; left: 0; width: 10px; height: 2px; }
        #crosshair::after { top: 0; left: 4px; width: 2px; height: 10px; }

        /* UI Paneli */
        #ui {
            position: absolute;
            top: 10px;
            left: 10px;
            color: white;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.8);
            font-weight: bold;
            pointer-events: none;
        }
        /* Hotbar (Envanter) */
        #hotbar {
            position: absolute;
            bottom: 20px;
            left: 50%;
            transform: translateX(-50%);
            display: flex;
            background: rgba(0, 0, 0, 0.5);
            padding: 5px;
            border-radius: 5px;
            border: 2px solid #333;
            pointer-events: none;
        }
        .slot {
            width: 50px;
            height: 50px;
            margin: 0 5px;
            border: 3px solid #555;
            display: flex;
            justify-content: center;
            align-items: center;
            color: white;
            font-size: 12px;
            font-weight: bold;
            background-size: cover;
        }
        .slot.active {
            border-color: #fff;
            box-shadow: 0 0 10px #fff;
        }
        #instructions {
            position: absolute;
            width: 100%;
            height: 100%;
            background: rgba(0,0,0,0.7);
            color: white;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            text-align: center;
            cursor: pointer;
            z-index: 10;
        }
    </style>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
</head>
<body>

    <div id="instructions">
        <h1 style="font-size: 3rem; margin-bottom: 10px;">JS-CRAFT PRO</h1>
        <p>Oynamak için ekrana tıkla</p>
        <p style="color: #aaa; margin-top: 20px;">
            WASD / Yön Tuşları: Hareket<br>
            SPACE: Zıplama<br>
            SOL TIK: Blok Kır | SAĞ TIK: Blok Koy<br>
            1, 2, 3, 4: Blok Seçimi
        </p>
    </div>

    <div id="crosshair"></div>
    <div id="ui">FPS: <span id="fps">0</span> | Blok: <span id="block-type">Çimen</span></div>
    
    <div id="hotbar">
        <div class="slot active" id="slot-1" style="background-color: #557a2b;">1</div>
        <div class="slot" id="slot-2" style="background-color: #808080;">2</div>
        <div class="slot" id="slot-3" style="background-color: #a07040;">3</div>
        <div class="slot" id="slot-4" style="background-color: #b04040;">4</div>
    </div>

    <div id="canvas-container"></div>

    <script>
        // --- OYUN AYARLARI VE DEĞİŞKENLER ---
        const BLOCK_SIZE = 1;
        const WORLD_SIZE = 16; // 16x16 chunk boyutu
        let scene, camera, renderer, clock;
        let moveForward = false, moveBackward = false, moveLeft = false, moveRight = false, canJump = false;
        
        const velocity = new THREE.Vector3();
        const direction = new THREE.Vector3();
        const objects = []; // Tıklanabilir (kırılabilir) bloklar listesi
        
        // Blok Tipleri Veri Modeli
        const blockTypes = {
            1: { name: 'Çimen', color: 0x557a2b },
            2: { name: 'Taş', color: 0x808080 },
            3: { name: 'Tahta', color: 0xa07040 },
            4: { name: 'Tuğla', color: 0xb04040 }
        };
        let activeBlockType = 1;

        // Pointer Lock (Fare Kilitleme) Kontrolü
        const instructions = document.getElementById('instructions');
        let controls = { isLocked: false };

        // --- BAŞLANGIÇ ---
        init();
        animate();

        function init() {
            // Scene & Camera
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x7ec0ee); // Gökyüzü rengi
            scene.fog = new THREE.FogExp2(0x7ec0ee, 0.05);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(8, 3, 8); // Dünyanın ortasında başlat

            clock = new THREE.Clock();

            // Işıklandırma (Gölge ve Derinlik hissi için şart)
            const ambientLight = new THREE.AmbientLight(0xcccccc, 0.6);
            scene.add(ambientLight);

            const directionalLight = new THREE.DirectionalLight(0xffffff, 0.6);
            directionalLight.position.set(20, 40, 20);
            scene.add(directionalLight);

            // Renderer
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setPixelRatio(window.devicePixelRatio);
            renderer.setSize(window.innerWidth, window.innerHeight);
            document.getElementById('canvas-container').appendChild(renderer.domElement);

            // Evren Oluşturma (Rastgele Zemin)
            generateWorld();

            // Giriş Kontrolleri Dinleyicileri
            setupControls();
            
            // Pencere Boyutu Değişimi
            window.addEventListener('resize', onWindowResize);
        }

        // --- DÜNYA GENERATÖRÜ (Chunk Üretimi) ---
        function generateWorld() {
            const geometry = new THREE.BoxGeometry(BLOCK_SIZE, BLOCK_SIZE, BLOCK_SIZE);
            
            for (let x = 0; x < WORLD_SIZE; x++) {
                for (let z = 0; z < WORLD_SIZE; z++) {
                    // Basit bir sinüs dalgası ile engebeli arazi simülasyonu
                    const height = Math.floor((Math.sin(x * 0.3) + Math.cos(z * 0.3)) * 1) + 1;
                    
                    for (let y = 0; y <= height; y++) {
                        let type = 2; // Varsayılan: Taş
                        if (y === height) type = 1; // En üst katman: Çimen

                        createBlock(x, y, z, type, geometry);
                    }
                }
            }
        }

        function createBlock(x, y, z, type, sharedGeometry) {
            const mat = new THREE.MeshLambertMaterial({ color: blockTypes[type].color });
            const mesh = new THREE.Mesh(sharedGeometry || new THREE.BoxGeometry(BLOCK_SIZE, BLOCK_SIZE, BLOCK_SIZE), mat);
            mesh.position.set(x, y, z);
            scene.add(mesh);
            objects.push(mesh);
            
            // Kenar çizgileri (Wireframe efektli profesyonel görünüm)
            const edges = new THREE.EdgesGeometry(mesh.geometry);
            const line = new THREE.LineSegments(edges, new THREE.LineBasicMaterial({ color: 0x000000, linewidth: 1, opacity: 0.15, transparent: true }));
            mesh.add(line);
        }

        // --- FARE VE KLAVYE KONTROLLERİ ---
        function setupControls() {
            // Tarayıcı içi Pointer Lock API Manuel Entegrasyonu (Modüler yapı için)
            instructions.addEventListener('click', () => {
                document.body.requestPointerLock();
            });

            document.addEventListener('pointerlockchange', () => {
                if (document.pointerLockElement === document.body) {
                    controls.isLocked = true;
                    instructions.style.display = 'none';
                } else {
                    controls.isLocked = false;
                    instructions.style.display = 'flex';
                }
            });

            // Fare Hareketiyle Kamera Döndürme
            document.addEventListener('mousemove', (e) => {
                if (!controls.isLocked) return;
                const sensitivity = 0.002;
                camera.rotation.y -= e.movementX * sensitivity;
                camera.rotation.x -= e.movementY * sensitivity;
                // Kameranın takla atmasını engellemek için sınırla
                camera.rotation.x = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, camera.rotation.x));
            });

            // Kamera rotasyon modunu ayarla
            camera.rotation.order = "YXZ";

            // Klavye Tuş Basışları
            const onKeyDown = (e) => {
                switch (e.code) {
                    case 'KeyW': case 'ArrowUp': moveForward = true; break;
                    case 'KeyS': case 'ArrowDown': moveBackward = true; break;
                    case 'KeyA': case 'ArrowLeft': moveLeft = true; break;
                    case 'KeyD': case 'ArrowRight': moveRight = true; break;
                    case 'Space': if (canJump === true) velocity.y += 6; canJump = false; break;
                    case 'Digit1': selectSlot(1); break;
                    case 'Digit2': selectSlot(2); break;
                    case 'Digit3': selectSlot(3); break;
                    case 'Digit4': selectSlot(4); break;
                }
            };

            const onKeyUp = (e) => {
                switch (e.code) {
                    case 'KeyW': case 'ArrowUp': moveForward = false; break;
                    case 'KeyS': case 'ArrowDown': moveBackward = false; break;
                    case 'KeyA': case 'ArrowLeft': moveLeft = false; break;
                    case 'KeyD': case 'ArrowRight': moveRight = false; break;
                }
            };

            document.addEventListener('keydown', onKeyDown);
            document.addEventListener('keyup', onKeyUp);

            // Blok Kırma ve Koyma (Raycasting Teknolojisi)
            document.addEventListener('mousedown', (e) => {
                if (!controls.isLocked) return;

                // Ekranın tam ortasından ışın fırlat (Crosshair hizası)
                const raycaster = new THREE.Raycaster();
                raycaster.setFromCamera(new THREE.Vector2(0, 0), camera);
                const intersects = raycaster.intersectObjects(objects);

                if (intersects.length > 0 && intersects[0].distance < 6) { // Maksimum erişim mesafesi: 6 blok
                    const intersect = intersects[0];

                    if (e.button === 0) { 
                        // SOL TIK: BLOĞU SİL
                        scene.remove(intersect.object);
                        const index = objects.indexOf(intersect.object);
                        if (index > -1) objects.splice(index, 1);
                    } 
                    else if (e.button === 2) {
                        // SAĞ TIK: BLOK YERLEŞTİR
                        const normal = intersect.face.normal;
                        const position = new THREE.Vector3()
                            .copy(intersect.object.position)
                            .add(normal); // Tıklanan yüzeyin normal vektörüne göre yeni koordinat hesapla
                        
                        createBlock(position.x, position.y, position.z, activeBlockType);
                    }
                }
            });

            // Sağ tık menüsünü engelle (Oyunda sağ tık rahatça kullanılsın diye)
            document.addEventListener('contextmenu', e => e.preventDefault());
        }

        function selectSlot(slotNumber) {
            activeBlockType = slotNumber;
            document.querySelectorAll('.slot').forEach(s => s.classList.remove('active'));
            document.getElementById(`slot-${slotNumber}`).classList.add('active');
            document.getElementById('block-type').innerText = blockTypes[slotNumber].name;
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        // --- ANA OYUN DÖNGÜSÜ (Fizik & Render) ---
        let lastTime = performance.now();
        let frames = 0;

        function animate() {
            requestAnimationFrame(animate);

            if (controls.isLocked) {
                const delta = clock.getDelta();
                
                // FPS Sayacı (Profesyonel optimizasyon takibi için)
                frames++;
                const time = performance.now();
                if (time >= lastTime + 1000) {
                    document.getElementById('fps').innerText = Math.round((frames * 1000) / (time - lastTime));
                    frames = 0;
                    lastTime = time;
                }

                // Sürtünme ve Yavaşlama Katsayıları (Damping)
                velocity.x -= velocity.x * 10.0 * delta;
                velocity.z -= velocity.z * 10.0 * delta;
                velocity.y -= 9.8 * 2.0 * delta; // Yerçekimi kuvveti

                direction.z = Number(moveForward) - Number(moveBackward);
                direction.x = Number(moveRight) - Number(moveLeft);
                direction.normalize();

                // Kameranın baktığı yöne göre hareket etme matematiği
                const camDirection = new THREE.Vector3();
                camera.getWorldDirection(camDirection);
                
                // Sadece XZ düzleminde yürü (Kafayı yukarı kaldırınca uçmayı engeller)
                const forward = new THREE.Vector3(camDirection.x, 0, camDirection.z).normalize();
                const right = new THREE.Vector3().crossVectors(forward, camera.up).negate().normalize();

                if (moveForward || moveBackward) velocity.addScaledVector(forward, direction.z * 40.0 * delta);
                if (moveLeft || moveRight) velocity.addScaledVector(right, direction.x * 40.0 * delta);

                // Pozisyon Güncelleme
                camera.position.addScaledVector(velocity, delta);

                // İlkel Taban Çarpışma Kontrolü (Yere çakılma)
                if (camera.position.y < 3) {
                    velocity.y = 0;
                    camera.position.y = 3;
                    canJump = true;
                }
            }

            renderer.render(scene, camera);
        }
    </script>
</body>
</html>
