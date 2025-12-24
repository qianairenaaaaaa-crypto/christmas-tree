# christmas-tree
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>手势控制3D照片圣诞树</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }
        body {
            overflow: hidden;
            background: #0a0a0f; /* 深黑底衬托辉光 */
            color: #fff;
        }
        #canvas-container {
            position: fixed;
            top: 0;
            left: 0;
            width: 100vw;
            height: 100vh;
            z-index: 1;
        }
        /* 照片上传控件 */
        .upload-panel {
            position: fixed;
            bottom: 30px;
            left: 50%;
            transform: translateX(-50%);
            z-index: 10;
            display: flex;
            flex-direction: column;
            align-items: center;
            gap: 10px;
            background: rgba(15, 20, 30, 0.8);
            padding: 15px 25px;
            border-radius: 12px;
            backdrop-filter: blur(8px);
            border: 1px solid rgba(255, 215, 0, 0.3);
        }
        #photo-upload {
            display: none;
        }
        .upload-btn {
            background: linear-gradient(135deg, #d4af37, #ffd700);
            color: #0a0a0f;
            border: none;
            padding: 10px 20px;
            border-radius: 8px;
            font-weight: 600;
            cursor: pointer;
            transition: all 0.3s ease;
            box-shadow: 0 0 15px rgba(255, 215, 0, 0.4);
        }
        .upload-btn:hover {
            transform: scale(1.05);
            box-shadow: 0 0 20px rgba(255, 215, 0, 0.6);
        }
        .upload-tip {
            font-size: 12px;
            color: #e0e0e0;
            opacity: 0.8;
        }
        /* 手势提示 */
        .gesture-tip {
            position: fixed;
            top: 20px;
            left: 50%;
            transform: translateX(-50%);
            z-index: 10;
            background: rgba(15, 20, 30, 0.8);
            padding: 10px 20px;
            border-radius: 8px;
            backdrop-filter: blur(8px);
            border: 1px solid rgba(154, 205, 50, 0.3);
            font-size: 14px;
            color: #f0f0f0;
        }
        /* 加载提示 */
        .loading {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            z-index: 20;
            font-size: 18px;
            color: #ffd700;
            text-shadow: 0 0 10px rgba(255, 215, 0, 0.5);
        }
    </style>
</head>
<body>
    <div class="loading" id="loading">加载中... 请允许摄像头权限</div>
    <div class="gesture-tip">
        手势提示：握拳=合拢 | 五指打开=散开 | 旋转手=旋转画面 | 捏合手指=放大照片
    </div>
    <div class="upload-panel">
        <label for="photo-upload" class="upload-btn">上传圣诞照片</label>
        <input type="file" id="photo-upload" accept="image/*" multiple />
        <p class="upload-tip">支持多张照片，上传后融入圣诞树</p>
    </div>
    <div id="canvas-container"></div>

    <!-- 依赖CDN引入 -->
    <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/js/controls/OrbitControls.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/js/postprocessing/EffectComposer.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/js/postprocessing/RenderPass.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/js/postprocessing/UnrealBloomPass.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/js/shaders/LuminosityShader.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/js/postprocessing/ShaderPass.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands@0.4.1646424915/hands.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils@0.3.1646425220/camera_utils.min.js"></script>

    <script>
        // ========== 全局配置 ==========
        const CONFIG = {
            // 颜色配置
            colors: {
                matteGreen: 0x2e4b28, // 哑光绿
                metalGold: 0xffd700,  // 金属金
                christmasRed: 0xc70039, // 圣诞红
                glow: 0xfff8e1        // 辉光底色
            },
            // 元素数量
            elementCounts: {
                spheres: 80,    // 绿色球
                cubes: 40,      // 金色正方体
                candyCanes: 20, // 圣诞红糖果棍
                photos: 0       // 上传照片数量
            },
            // 状态控制
            states: {
                CLOSED: 'closed',   // 合拢态
                SCATTERED: 'scattered', // 散开态
                PHOTO_ZOOM: 'photo_zoom' // 照片放大态
            },
            currentState: 'closed',
            targetPhoto: null, // 选中的放大照片
            // 手势阈值
            gestureThresholds: {
                fist: 0.1,       // 握拳阈值（手指弯曲度）
                openHand: 0.8,   // 开手阈值
                pinch: 0.05,     // 捏合阈值（拇指食指距离）
                rotateSensitivity: 0.01 // 旋转灵敏度
            },
            // 动画过渡
            transitionSpeed: 0.05,
            // 相机参数
            camera: {
                fov: 75,
                near: 0.1,
                far: 1000,
                position: { x: 0, y: 0, z: 10 },
                target: { x: 0, y: 0, z: 0 }
            }
        };

        // ========== Three.js 核心变量 ==========
        let scene, camera, renderer, controls, composer;
        let christmasTreeGroup, photoGroup;
        let instancedSpheres, instancedCubes, candyCaneGroup;
        let spherePositions = [], cubePositions = [], candyCanePositions = [];
        let scatteredSpherePositions = [], scatteredCubePositions = [], scatteredCandyCanePositions = [];
        let photoTextures = [], photoMeshes = [];

        // ========== MediaPipe 变量 ==========
        let hands, cameraMP, isHandDetected = false;
        let lastHandLandmarks = null;
        let cameraRotation = { x: 0, y: 0 };

        // ========== 初始化 Three.js 场景 ==========
        function initThreeJS() {
            // 1. 创建场景
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x0a0a0f);
            scene.fog = new THREE.Fog(0x0a0a0f, 20, 80); // 电影感雾效

            // 2. 创建相机
            camera = new THREE.PerspectiveCamera(
                CONFIG.camera.fov,
                window.innerWidth / window.innerHeight,
                CONFIG.camera.near,
                CONFIG.camera.far
            );
            camera.position.set(
                CONFIG.camera.position.x,
                CONFIG.camera.position.y,
                CONFIG.camera.position.z
            );
            camera.lookAt(CONFIG.camera.target.x, CONFIG.camera.target.y, CONFIG.camera.target.z);

            // 3. 创建渲染器
            renderer = new THREE.WebGLRenderer({ 
                antialias: true, 
                alpha: true,
                powerPreference: "high-performance"
            });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
            renderer.shadowMap.enabled = true;
            renderer.shadowMap.type = THREE.PCFSoftShadowMap;
            document.getElementById('canvas-container').appendChild(renderer.domElement);

            // 4. 控制器（备用，主要靠手势）
            controls = new THREE.OrbitControls(camera, renderer.domElement);
            controls.enabled = false; // 禁用手动控制，改用手势

            // 5. 光照系统（营造金属质感和辉光）
            initLighting();

            // 6. 创建圣诞树元素
            initChristmasTreeElements();

            // 7. 后期处理（电影感辉光）
            initPostProcessing();

            // 8. 窗口自适应
            window.addEventListener('resize', onWindowResize);

            // 隐藏加载提示
            document.getElementById('loading').style.display = 'none';
        }

        // ========== 初始化光照系统 ==========
        function initLighting() {
            // 环境光（柔和基础光）
            const ambientLight = new THREE.AmbientLight(CONFIG.colors.glow, 0.4);
            scene.add(ambientLight);

            // 主方向光（金属金高光）
            const dirLight1 = new THREE.DirectionalLight(CONFIG.colors.metalGold, 1.2);
            dirLight1.position.set(10, 15, 10);
            dirLight1.castShadow = true;
            dirLight1.shadow.mapSize.width = 2048;
            dirLight1.shadow.mapSize.height = 2048;
            scene.add(dirLight1);

            // 辅助方向光（绿色/红色元素补光）
            const dirLight2 = new THREE.DirectionalLight(CONFIG.colors.matteGreen, 0.8);
            dirLight2.position.set(-10, 10, -10);
            scene.add(dirLight2);

            // 点光（辉光核心）
            const pointLight = new THREE.PointLight(CONFIG.colors.metalGold, 1.5, 50);
            pointLight.position.set(0, 5, 0);
            pointLight.castShadow = true;
            scene.add(pointLight);

            // 红色点光（糖果棍补光）
            const redPointLight = new THREE.PointLight(CONFIG.colors.christmasRed, 0.8, 30);
            redPointLight.position.set(0, -5, 0);
            scene.add(redPointLight);
        }

        // ========== 初始化后期处理（辉光效果） ==========
        function initPostProcessing() {
            // 渲染通道
            const renderPass = new THREE.RenderPass(scene, camera);

            // 辉光通道（UnrealBloomPass 模拟电影级辉光）
            const bloomPass = new THREE.UnrealBloomPass(
                new THREE.Vector2(window.innerWidth, window.innerHeight),
                1.2,    // 强度
                0.8,    // 半径
                0.3     // 阈值
            );
            bloomPass.threshold = 0.05;
            bloomPass.strength = 1.5; // 辉光强度
            bloomPass.radius = 0.8;   // 辉光半径

            // 亮度通道（优化辉光）
            const luminosityShader = THREE.LuminosityShader;
            const luminosityPass = new THREE.ShaderPass(luminosityShader);

            // 合成器
            composer = new THREE.EffectComposer(renderer);
            composer.addPass(renderPass);
            composer.addPass(luminosityPass);
            composer.addPass(bloomPass);
        }

        // ========== 创建圣诞树基础元素 ==========
        function initChristmasTreeElements() {
            christmasTreeGroup = new THREE.Group();
            photoGroup = new THREE.Group();
            scene.add(christmasTreeGroup);
            scene.add(photoGroup);

            // 1. 哑光绿球体（InstancedMesh 优化性能）
            createInstancedSpheres();

            // 2. 金属金正方体（InstancedMesh）
            createInstancedCubes();

            // 3. 圣诞红糖果棍
            createCandyCanes();

            // 初始化位置（合拢态）
            updateElementPositions('closed');
        }

        // ========== 创建哑光绿球体 ==========
        function createInstancedSpheres() {
            const sphereGeometry = new THREE.SphereGeometry(0.3, 16, 16);
            const sphereMaterial = new THREE.MeshStandardMaterial({
                color: CONFIG.colors.matteGreen,
                roughness: 0.2, // 哑光
                metalness: 0.1,
                emissive: CONFIG.colors.matteGreen,
                emissiveIntensity: 0.1
            });

            instancedSpheres = new THREE.InstancedMesh(sphereGeometry, sphereMaterial, CONFIG.elementCounts.spheres);
            instancedSpheres.instanceMatrix.setUsage(THREE.DynamicDrawUsage);
            christmasTreeGroup.add(instancedSpheres);

            // 生成合拢态和散开态位置
            const matrix = new THREE.Matrix4();
            for (let i = 0; i < CONFIG.elementCounts.spheres; i++) {
                // 合拢态：圆锥体分布
                const height = Math.random() * 8 + 2;
                const radius = (10 - height) * 0.3;
                const angle = Math.random() * Math.PI * 2;
                const x = Math.cos(angle) * radius;
                const y = -height;
                const z = Math.sin(angle) * radius;
                spherePositions.push({ x, y, z });

                // 散开态：随机分布
                scatteredSpherePositions.push({
                    x: (Math.random() - 0.5) * 30,
                    y: (Math.random() - 0.5) * 20,
                    z: (Math.random() - 0.5) * 30
                });

                matrix.setPosition(x, y, z);
                instancedSpheres.setMatrixAt(i, matrix);
            }
            instancedSpheres.instanceMatrix.needsUpdate = true;
        }

        // ========== 创建金属金正方体 ==========
        function createInstancedCubes() {
            const cubeGeometry = new THREE.BoxGeometry(0.4, 0.4, 0.4);
            const cubeMaterial = new THREE.MeshStandardMaterial({
                color: CONFIG.colors.metalGold,
                roughness: 0.1, // 高金属感
                metalness: 0.9,
                emissive: CONFIG.colors.metalGold,
                emissiveIntensity: 0.2
            });

            instancedCubes = new THREE.InstancedMesh(cubeGeometry, cubeMaterial, CONFIG.elementCounts.cubes);
            instancedCubes.instanceMatrix.setUsage(THREE.DynamicDrawUsage);
            christmasTreeGroup.add(instancedCubes);

            // 生成合拢态和散开态位置
            const matrix = new THREE.Matrix4();
            for (let i = 0; i < CONFIG.elementCounts.cubes; i++) {
                // 合拢态：圆锥体分布（比球体更内层）
                const height = Math.random() * 7 + 3;
                const radius = (10 - height) * 0.2;
                const angle = Math.random() * Math.PI * 2;
                const x = Math.cos(angle) * radius;
                const y = -height;
                const z = Math.sin(angle) * radius;
                cubePositions.push({ x, y, z });

                // 散开态：随机分布
                scatteredCubePositions.push({
                    x: (Math.random() - 0.5) * 25,
                    y: (Math.random() - 0.5) * 18,
                    z: (Math.random() - 0.5) * 25
                });

                matrix.setPosition(x, y, z);
                instancedCubes.setMatrixAt(i, matrix);
            }
            instancedCubes.instanceMatrix.needsUpdate = true;
        }

        // ========== 创建圣诞红糖果棍 ==========
        function createCandyCanes() {
            candyCaneGroup = new THREE.Group();
            christmasTreeGroup.add(candyCaneGroup);

            for (let i = 0; i < CONFIG.elementCounts.candyCanes; i++) {
                // 糖果棍几何体（圆柱+条纹）
                const caneGeometry = new THREE.CylinderGeometry(0.15, 0.15, 1.2, 12);
                const caneMaterial = new THREE.MeshStandardMaterial({
                    color: CONFIG.colors.christmasRed,
                    roughness: 0.3,
                    metalness: 0.2,
                    emissive: CONFIG.colors.christmasRed,
                    emissiveIntensity: 0.1
                });

                const candyCane = new THREE.Mesh(caneGeometry, caneMaterial);
                candyCane.castShadow = true;

                // 合拢态位置
                const height = Math.random() * 6 + 4;
                const radius = (10 - height) * 0.25;
                const angle = Math.random() * Math.PI * 2;
                const x = Math.cos(angle) * radius;
                const y = -height + 0.6;
                const z = Math.sin(angle) * radius;
                candyCane.position.set(x, y, z);
                candyCane.rotation.z = Math.random() * Math.PI;
                candyCaneGroup.add(candyCane);

                // 保存位置
                candyCanePositions.push({ x, y, z, rotation: candyCane.rotation.z });

                // 散开态位置
                scatteredCandyCanePositions.push({
                    x: (Math.random() - 0.5) * 28,
                    y: (Math.random() - 0.5) * 19,
                    z: (Math.random() - 0.5) * 28,
                    rotation: Math.random() * Math.PI * 2
                });
            }
        }

        // ========== 照片上传处理 ==========
        document.getElementById('photo-upload').addEventListener('change', handlePhotoUpload);
        function handlePhotoUpload(e) {
            const files = e.target.files;
            if (!files.length) return;

            // 清空原有照片
            photoGroup.clear();
            photoMeshes = [];
            photoTextures = [];
            CONFIG.elementCounts.photos = files.length;

            // 处理每张照片
            Array.from(files).forEach((file, index) => {
                const reader = new FileReader();
                reader.onload = (event) => {
                    // 创建纹理
                    const texture = new THREE.TextureLoader().load(event.target.result);
                    texture.needsUpdate = true;
                    photoTextures.push(texture);

                    // 创建照片平面
                    const photoGeometry = new THREE.PlaneGeometry(2, 2);
                    const photoMaterial = new THREE.MeshStandardMaterial({
                        map: texture,
                        transparent: true,
                        opacity: 0.9,
                        side: THREE.DoubleSide,
                        emissive: CONFIG.colors.glow,
                        emissiveIntensity: 0.1
                    });

                    const photoMesh = new THREE.Mesh(photoGeometry, photoMaterial);
                    photoMesh.userData = { 
                        index,
                        closedPosition: { // 合拢态位置（圣诞树间隙）
                            x: (Math.random() - 0.5) * 4,
                            y: -Math.random() * 8 - 2,
                            z: (Math.random() - 0.5) * 4
                        },
                        scatteredPosition: { // 散开态位置
                            x: (Math.random() - 0.5) * 35,
                            y: (Math.random() - 0.5) * 22,
                            z: (Math.random() - 0.5) * 35
                        },
                        zoomPosition: { // 放大态位置（屏幕中央）
                            x: 0, y: 0, z: -2
                        },
                        scale: 1,
                        targetScale: 1
                    };

                    // 初始位置（合拢态）
                    photoMesh.position.copy(photoMesh.userData.closedPosition);
                    photoGroup.add(photoMesh);
                    photoMeshes.push(photoMesh);

                    // 更新位置
                    updateElementPositions(CONFIG.currentState);
                };
                reader.readAsDataURL(file);
            });
        }

        // ========== 更新元素位置（状态切换） ==========
        function updateElementPositions(targetState) {
            const matrix = new THREE.Matrix4();

            // 1. 更新球体位置
            for (let i = 0; i < CONFIG.elementCounts.spheres; i++) {
                let targetPos;
                if (targetState === CONFIG.states.CLOSED) {
                    targetPos = spherePositions[i];
                } else {
                    targetPos = scatteredSpherePositions[i];
                }

                // 平滑过渡
                const currentPos = new THREE.Vector3();
                instancedSpheres.getMatrixAt(i, matrix);
                currentPos.setFromMatrixPosition(matrix);

                const newPos = currentPos.lerp(new THREE.Vector3(targetPos.x, targetPos.y, targetPos.z), CONFIG.transitionSpeed);
                matrix.setPosition(newPos);
                instancedSpheres.setMatrixAt(i, matrix);
            }
            instancedSpheres.instanceMatrix.needsUpdate = true;

            // 2. 更新正方体位置
            for (let i = 0; i < CONFIG.elementCounts.cubes; i++) {
                let targetPos;
                if (targetState === CONFIG.states.CLOSED) {
                    targetPos = cubePositions[i];
                } else {
                    targetPos = scatteredCubePositions[i];
                }

                const currentPos = new THREE.Vector3();
                instancedCubes.getMatrixAt(i, matrix);
                currentPos.setFromMatrixPosition(matrix);

                const newPos = currentPos.lerp(new THREE.Vector3(targetPos.x, targetPos.y, targetPos.z), CONFIG.transitionSpeed);
                matrix.setPosition(newPos);
                instancedCubes.setMatrixAt(i, matrix);
            }
            instancedCubes.instanceMatrix.needsUpdate = true;

            // 3. 更新糖果棍位置
            for (let i = 0; i < CONFIG.elementCounts.candyCanes; i++) {
                const candyCane = candyCaneGroup.children[i];
                let targetPos, targetRotation;

                if (targetState === CONFIG.states.CLOSED) {
                    targetPos = candyCanePositions[i];
                    targetRotation = candyCanePositions[i].rotation;
                } else {
                    targetPos = scatteredCandyCanePositions[i];
                    targetRotation = scatteredCandyCanePositions[i].rotation;
                }

                // 位置过渡
                candyCane.position.lerp(new THREE.Vector3(targetPos.x, targetPos.y, targetPos.z), CONFIG.transitionSpeed);
                // 旋转过渡
                candyCane.rotation.z = THREE.MathUtils.lerp(candyCane.rotation.z, targetRotation, CONFIG.transitionSpeed);
            }

            // 4. 更新照片位置
            if (photoMeshes.length) {
                photoMeshes.forEach(photo => {
                    let targetPos, targetScale;

                    if (CONFIG.currentState === CONFIG.states.PHOTO_ZOOM && photo === CONFIG.targetPhoto) {
                        // 选中照片放大
                        targetPos = photo.userData.zoomPosition;
                        targetScale = 4;
                        photo.material.opacity = THREE.MathUtils.lerp(photo.material.opacity, 1, CONFIG.transitionSpeed);
                    } else if (targetState === CONFIG.states.CLOSED) {
                        // 合拢态
                        targetPos = photo.userData.closedPosition;
                        targetScale = 1;
                        photo.material.opacity = THREE.MathUtils.lerp(photo.material.opacity, 0.9, CONFIG.transitionSpeed);
                    } else {
                        // 散开态（非选中照片透明度降低）
                        targetPos = photo.userData.scatteredPosition;
                        targetScale = 1;
                        const opacity = CONFIG.currentState === CONFIG.states.PHOTO_ZOOM ? 0.3 : 0.9;
                        photo.material.opacity = THREE.MathUtils.lerp(photo.material.opacity, opacity, CONFIG.transitionSpeed);
                    }

                    // 位置和缩放过渡
                    photo.position.lerp(new THREE.Vector3(targetPos.x, targetPos.y, targetPos.z), CONFIG.transitionSpeed);
                    photo.scale.setScalar(THREE.MathUtils.lerp(photo.scale.x, targetScale, CONFIG.transitionSpeed));
                });
            }
        }

        // ========== 初始化 MediaPipe Hands ==========
        function initMediaPipeHands() {
            // 配置Hands
            hands = new window.mediapipe.Hands({
                locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands@0.4.1646424915/${file}`
            });

            hands.setOptions({
                maxNumHands: 1,
                modelComplexity: 1,
                minDetectionConfidence: 0.7,
                minTrackingConfidence: 0.7
            });

            // 手部检测回调
            hands.onResults(handleHandResults);

            // 初始化摄像头
            cameraMP = new window.mediapipe.CameraUtils(renderer.domElement, {
                onFrame: async () => {
                    await hands.send({ image: renderer.domElement });
                },
                width: 1280,
                height: 720
            });

            // 启动摄像头
            cameraMP.start().catch(err => {
                console.error('摄像头启动失败:', err);
                document.getElementById('loading').textContent = '请允许摄像头权限以使用手势控制';
            });
        }

        // ========== 处理手部识别结果 ==========
        function handleHandResults(results) {
            isHandDetected = results.multiHandLandmarks && results.multiHandLandmarks.length > 0;
            if (!isHandDetected) {
                lastHandLandmarks = null;
                return;
            }

            const landmarks = results.multiHandLandmarks[0];
            lastHandLandmarks = landmarks;

            // 1. 识别手势类型
            const gestureType = detectGesture(landmarks);

            // 2. 处理不同手势
            switch (gestureType) {
                case 'fist': // 握拳 - 合拢态
                    if (CONFIG.currentState !== CONFIG.states.CLOSED) {
                        CONFIG.currentState = CONFIG.states.CLOSED;
                        CONFIG.targetPhoto = null;
                    }
                    break;
                case 'openHand': // 开手 - 散开态
                    if (CONFIG.currentState !== CONFIG.states.SCATTERED && CONFIG.currentState !== CONFIG.states.PHOTO_ZOOM) {
                        CONFIG.currentState = CONFIG.states.SCATTERED;
                    }
                    break;
                case 'pinch': // 捏合 - 放大照片
                    if (CONFIG.currentState === CONFIG.states.SCATTERED && photoMeshes.length) {
                        CONFIG.currentState = CONFIG.states.PHOTO_ZOOM;
                        // 选中最近的照片
                        CONFIG.targetPhoto = getNearestPhoto();
                    }
                    break;
                case 'rotate': // 旋转 - 调整相机
                    if (CONFIG.currentState === CONFIG.states.SCATTERED || CONFIG.currentState === CONFIG.states.PHOTO_ZOOM) {
                        updateCameraRotation(landmarks);
                    }
                    break;
            }
        }

        // ========== 手势识别逻辑 ==========
        function detectGesture(landmarks) {
            // 计算手指弯曲度（0=完全弯曲，1=完全伸直）
            const fingerFlex = calculateFingerFlexibility(landmarks);
            const avgFlex = fingerFlex.reduce((a, b) => a + b, 0) / fingerFlex.length;

            // 1. 检测握拳（平均弯曲度 < 阈值）
            if (avgFlex < CONFIG.gestureThresholds.fist) {
                return 'fist';
            }

            // 2. 检测开手（平均弯曲度 > 阈值）
            if (avgFlex > CONFIG.gestureThresholds.openHand) {
                // 检测是否旋转
                if (lastHandLandmarks) {
                    const wristNow = landmarks[0];
                    const wristLast = lastHandLandmarks[0];
                    const moveX = Math.abs(wristNow.x - wristLast.x);
                    const moveY = Math.abs(wristNow.y - wristLast.y);
                    if (moveX > CONFIG.gestureThresholds.rotateSensitivity || moveY > CONFIG.gestureThresholds.rotateSensitivity) {
                        return 'rotate';
                    }
                }
                return 'openHand';
            }

            // 3. 检测捏合（拇指和食指距离）
            const thumbTip = landmarks[4];
            const indexTip = landmarks[8];
            const pinchDistance = Math.hypot(
                thumbTip.x - indexTip.x,
                thumbTip.y - indexTip.y,
                thumbTip.z - indexTip.z
            );
            if (pinchDistance < CONFIG.gestureThresholds.pinch) {
                return 'pinch';
            }

            return 'unknown';
        }

        // ========== 计算手指弯曲度 ==========
        function calculateFingerFlexibility(landmarks) {
            // 手指关键点索引：拇指(4), 食指(8), 中指(12), 无名指(16), 小指(20)
            const fingerTips = [4, 8, 12, 16, 20];
            const fingerMCPs = [2, 5, 9, 13, 17]; // 掌指关节

            const flexibilities = [];
            for (let i = 0; i < fingerTips.length; i++) {
                const tip = landmarks[fingerTips[i]];
                const mcp = landmarks[fingerMCPs[i]];
                const wrist = landmarks[0];

                // 计算手指伸直度（距离比例）
                const tipToWrist = Math.hypot(tip.x - wrist.x, tip.y - wrist.y, tip.z - wrist.z);
                const mcpToWrist = Math.hypot(mcp.x - wrist.x, mcp.y - wrist.y, mcp.z - wrist.z);
                const flex = Math.min(1, (tipToWrist - mcpToWrist) / 0.1); // 归一化到0-1
                flexibilities.push(flex);
            }
            return flexibilities;
        }

        // ========== 获取最近的照片 ==========
        function getNearestPhoto() {
            if (!photoMeshes.length) return null;

            let nearestPhoto = photoMeshes[0];
            let minDistance = Infinity;

            photoMeshes.forEach(photo => {
                const distance = camera.position.distanceTo(photo.position);
                if (distance < minDistance) {
                    minDistance = distance;
                    nearestPhoto = photo;
                }
            });

            return nearestPhoto;
        }

        // ========== 更新相机旋转（手势控制） ==========
        function updateCameraRotation(landmarks) {
            if (!lastHandLandmarks) return;

            // 获取手腕位置变化
            const wristDeltaX = landmarks[0].x - lastHandLandmarks[0].x;
            const wristDeltaY = landmarks[0].y - lastHandLandmarks[0].y;

            // 调整相机角度
            cameraRotation.y += wristDeltaX * 2;
            cameraRotation.x += wristDeltaY * 2;

            // 限制旋转角度
            cameraRotation.x = THREE.MathUtils.clamp(cameraRotation.x, -Math.PI / 4, Math.PI / 4);

            // 应用旋转
            camera.position.x = Math.sin(cameraRotation.y) * 10;
            camera.position.z = Math.cos(cameraRotation.y) * 10;
            camera.position.y = cameraRotation.x * 5;
            camera.lookAt(0, 0, 0);
        }

        // ========== 窗口自适应 ==========
        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
            composer.setSize(window.innerWidth, window.innerHeight);
        }

        // ========== 动画循环 ==========
        function animate() {
            requestAnimationFrame(animate);

            // 更新元素位置（状态过渡）
            updateElementPositions(CONFIG.currentState);

            // 旋转元素增加动态感
            if (CONFIG.currentState === CONFIG.states.SCATTERED || CONFIG.currentState === CONFIG.states.PHOTO_ZOOM) {
                // 球体旋转
                const matrix = new THREE.Matrix4();
                for (let i = 0; i < CONFIG.elementCounts.spheres; i++) {
                    instancedSpheres.getMatrixAt(i, matrix);
                    matrix.multiply(new THREE.Matrix4().makeRotationY(0.005));
                    instancedSpheres.setMatrixAt(i, matrix);
                }
                instancedSpheres.instanceMatrix.needsUpdate = true;

                // 正方体旋转
                for (let i = 0; i < CONFIG.elementCounts.cubes; i++) {
                    instancedCubes.getMatrixAt(i, matrix);
                    matrix.multiply(new THREE.Matrix4().makeRotationX(0.003).makeRotationZ(0.004));
                    instancedCubes.setMatrixAt(i, matrix);
                }
                instancedCubes.instanceMatrix.needsUpdate = true;

                // 糖果棍旋转
                candyCaneGroup.children.forEach(cane => {
                    cane.rotation.x += 0.002;
                    cane.rotation.y += 0.003;
                });

                // 照片旋转
                photoMeshes.forEach(photo => {
                    if (photo !== CONFIG.targetPhoto) {
                        photo.rotation.y += 0.002;
                    }
                });
            }

            // 渲染场景（后期处理）
            composer.render();
        }

        // ========== 初始化应用 ==========
        function initApp() {
            initThreeJS();
            initMediaPipeHands();
            animate();
        }

        // 启动应用
        window.addEventListener('load', initApp);
    </script>
</body>
</html>
