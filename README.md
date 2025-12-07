<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gesture Particles</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #050505; font-family: sans-serif; }
        
        /* Loading Screen */
        #loading {
            position: absolute;
            top: 50%; left: 50%; transform: translate(-50%, -50%);
            color: white; font-size: 20px; text-align: center;
            z-index: 200; pointer-events: none;
        }

        /* Video Preview (Small & Hidden mostly) */
        #video-container {
            position: absolute; bottom: 20px; right: 20px;
            width: 120px; height: 90px;
            border-radius: 8px; overflow: hidden;
            border: 2px solid rgba(255,255,255,0.2);
            z-index: 10; transform: scaleX(-1);
        }
        /* CRITICAL for iPhone: playsinline */
        video { width: 100%; height: 100%; object-fit: cover; }

        /* UI */
        #ui {
            position: absolute; top: 20px; left: 20px;
            background: rgba(0,0,0,0.7); color: white;
            padding: 15px; border-radius: 12px;
            z-index: 100; max-width: 200px;
            border: 1px solid rgba(255,255,255,0.2);
        }
        button {
            background: #333; color: white; border: 1px solid #555;
            padding: 8px; width: 100%; margin-bottom: 5px;
            cursor: pointer; border-radius: 4px;
        }
        button:hover { background: #555; }
        input[type="color"] { width: 100%; height: 30px; border: none; }
    </style>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
</head>
<body>

    <div id="loading">Initialize Camera...<br><span style="font-size:12px; opacity:0.6">Please Allow Access</span></div>

    <div id="ui">
        <h3 style="margin:0 0 10px 0;">Controls</h3>
        <button onclick="changeShape('heart')">Heart</button>
        <button onclick="changeShape('saturn')">Saturn</button>
        <button onclick="changeShape('flower')">Flower</button>
        <input type="color" id="colPick" value="#ff0066">
        <div style="font-size:11px; margin-top:10px;">Hand Tension: <span id="tension">0%</span></div>
    </div>

    <div id="video-container">
        <video id="input_video" playsinline autoplay muted></video>
    </div>

    <script>
        // --- 1. Three.js Setup ---
        const scene = new THREE.Scene();
        scene.fog = new THREE.FogExp2(0x050505, 0.02);
        
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
        camera.position.z = 30;

        const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        // --- Particles ---
        const count = 12000;
        const geom = new THREE.BufferGeometry();
        const pos = new Float32Array(count * 3);
        // Start random
        for(let i=0; i<count*3; i++) pos[i] = (Math.random()-0.5)*50;
        geom.setAttribute('position', new THREE.BufferAttribute(pos, 3));

        // Texture
        const canvas = document.createElement('canvas');
        canvas.width = 32; canvas.height = 32;
        const ctx = canvas.getContext('2d');
        const grad = ctx.createRadialGradient(16,16,0,16,16,16);
        grad.addColorStop(0, 'white');
        grad.addColorStop(1, 'transparent');
        ctx.fillStyle = grad;
        ctx.fillRect(0,0,32,32);
        const tex = new THREE.Texture(canvas);
        tex.needsUpdate = true;

        const mat = new THREE.PointsMaterial({
            size: 0.2, map: tex, transparent: true, 
            opacity: 0.8, color: 0xff0066, blending: THREE.AdditiveBlending, depthWrite: false
        });
        const particles = new THREE.Points(geom, mat);
        scene.add(particles);

        // --- Shape Logic ---
        let targetPos = new Float32Array(count * 3);
        let handFactor = 0.5; // 0=Closed, 1=Open
        let smoothFactor = 0.5;

        function updateTarget(shape) {
            for(let i=0; i<count; i++) {
                let x,y,z; 
                const idx = i*3;
                if(shape === 'heart') {
                    const t = Math.random()*Math.PI*2;
                    const p = Math.random()*Math.PI*2;
                    x = 16 * Math.pow(Math.sin(t), 3);
                    y = 13 * Math.cos(t) - 5*Math.cos(2*t) - 2*Math.cos(3*t) - Math.cos(4*t);
                    z = 4 * Math.cos(p) * Math.sin(t);
                    x*=0.8; y*=0.8; z*=0.8;
                } else if(shape === 'saturn') {
                    if(i < count*0.3) {
                         // Planet
                        const r = 6;
                        const theta = Math.random()*Math.PI*2;
                        const phi = Math.acos(2*Math.random()-1);
                        x=r*Math.sin(phi)*Math.cos(theta);
                        y=r*Math.sin(phi)*Math.sin(theta);
                        z=r*Math.cos(phi);
                    } else {
                        // Ring
                        const ang = Math.random()*Math.PI*2;
                        const rad = 10 + Math.random()*8;
                        x=Math.cos(ang)*rad;
                        z=Math.sin(ang)*rad;
                        y=(Math.random()-0.5);
                    }
                } else { // Flower
                    const theta = Math.random()*Math.PI*2;
                    const phi = Math.random()*Math.PI;
                    const r = 10 * Math.cos(4*theta);
                    x = r * Math.sin(phi) * Math.cos(theta);
                    y = r * Math.sin(phi) * Math.sin(theta);
                    z = (Math.random()-0.5)*5;
                }
                targetPos[idx] = x;
                targetPos[idx+1] = y;
                targetPos[idx+2] = z;
            }
        }
        updateTarget('heart'); // Init

        // UI Functions
        window.changeShape = (s) => updateTarget(s);
        document.getElementById('colPick').addEventListener('input', e => mat.color.set(e.target.value));

        // --- MediaPipe Logic ---
        const videoElement = document.getElementById('input_video');
        
        function onResults(results) {
            document.getElementById('loading').style.display = 'none';
            let val = 0.5;
            if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                const lm = results.multiHandLandmarks[0];
                // Dist between Wrist(0) and Middle Tip(12)
                const d = Math.sqrt(Math.pow(lm[12].x - lm[0].x, 2) + Math.pow(lm[12].y - lm[0].y, 2));
                val = (d - 0.15) * 3.5; // Normalize
                val = Math.max(0, Math.min(1, val));
            } else {
                val = 0.5 + Math.sin(Date.now()*0.002)*0.2; // Idle animation
            }
            handFactor = val;
            document.getElementById('tension').innerText = Math.round(handFactor*100)+'%';
        }

        const hands = new Hands({locateFile: (file) => {
            return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
        }});
        hands.setOptions({ maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.5, minTrackingConfidence: 0.5 });
        hands.onResults(onResults);

        const cam = new Camera(videoElement, {
            onFrame: async () => { await hands.send({image: videoElement}); },
            width: 640, height: 480
        });
        cam.start();

        // --- Animate ---
        function animate() {
            requestAnimationFrame(animate);
            smoothFactor += (handFactor - smoothFactor) * 0.1;
            const scale = 0.2 + (smoothFactor * 2.0);

            const p = geom.attributes.position.array;
            for(let i=0; i<count; i++) {
                const k = i*3;
                p[k] += (targetPos[k]*scale - p[k])*0.05;
                p[k+1] += (targetPos[k+1]*scale - p[k+1])*0.05;
                p[k+2] += (targetPos[k+2]*scale - p[k+2])*0.05;
            }
            geom.attributes.position.needsUpdate = true;
            particles.rotation.y += 0.002;
            
            renderer.render(scene, camera);
        }
        animate();

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth/window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
