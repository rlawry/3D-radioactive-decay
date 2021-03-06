<html>

<head>
    <meta charset="utf-8">
    <title>Decaying Simulations</title>
    <style>
        body {
            margin: 0;
        }

        canvas {
            display: block;
            z-index: 0;
        }

        .controls {
            position: absolute;
            background: white;
            top: 50px;
            left: 50px;
            margin: 20px;
            z-index: 2;
            border-radius: 15px;
            box-shadow: 0px 0px 10px black inset, 0 0 10px  black;
            padding: 10px;
        }
        .select-good {
            position: relative;
            font-family: Arial;
        }
        div.elapsed {
            display: block;
            margin-top: 10px;
            top: 10px;
            z-index:2;
        }

    </style>
</head>

<body>
    <script type="module">
        var xAtomTotal = 10
        var yAtomTotal = 10
        var atomTotal = xAtomTotal * yAtomTotal;
        var halves = [atomTotal/2,atomTotal/4,atomTotal/8,atomTotal/16,atomTotal/32,atomTotal/64];
        
        import {
               DoubleSide, FrontSide, BackSide, TextBufferGeometry
        } from "../node_modules/three/build/three.module.js";

        import * as THREE from 'https://cdn.jsdelivr.net/npm/three@0.124/build/three.module.js'; 

        import { OrbitControls } from 'https://cdn.jsdelivr.net/npm/three@0.124/examples/jsm/controls/OrbitControls.js'; 
        import { TTFLoader } from 'https://cdn.jsdelivr.net/npm/three@0.124/examples/jsm/loaders/TTFLoader.js'; 
        
        let atomNum, counter, textMaterial, time0, time1, texture;
        var running = false;

        var params = {
            scale: 1,
            rotation: 0.01,
            color: "rgb(200,200,255)" //starting color
        };

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 2000);
        const renderer = new THREE.WebGLRenderer();

        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);
        const controls = new OrbitControls(camera, renderer.domElement);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFSoftShadowMap; // default THREE.PCFShadowMap

        const loader = new THREE.CubeTextureLoader();

        var radius = 0.6;
        var detail = 2;
        const geometry1 = new THREE.OctahedronBufferGeometry(radius, 2);
        const geometry2 = new THREE.OctahedronBufferGeometry(radius, 0);
        const geometry3 = new THREE.OctahedronBufferGeometry(radius, 1);
        const geometry4 = new THREE.OctahedronBufferGeometry(radius, 3);
        const geometries = [geometry1, geometry2, geometry3, geometry4];
        const materials = [];
        const atom = [];
        const originalColors = [0xFFFF00,0x55AA00,0x5500AA,0xAA0055];
        const decayColors = [0x000011,0xfc6c85,0xFFFFFA,0x002200];
        var textColors = [0xFF0000,0x0000FF,0x33FF33,0x00AAFF,0xFF00FF,0xFF6600];
        
        var atomSpeed = [];
        var randNr;
        var halfLife = [10,15,5,3.5];                    
        var on = true;
        var timeVariable = 0;
        var t0;                            
        var timeDecay = new Array(atomTotal);                        
        var decayed = new Array(atomTotal);
        var elapsedTime = 0;
        var bufferTime = 0;
        var timeBoost = 0;
        
        const height = 0.25,
				size = 1,
				hover = 2,
				curveSegments = 4,
				bevelThickness = 0.05,
                bevelSize = 0.05;

        var simulationNumber = 1;

        var speedCountdown = [];
        var killSpeed = 100;
        var xRotation = [], yRotation = [], zRotation = [];
        //setup the units with a unique material and mesh, and a unique time of decay

        counter = atomTotal;
        textMaterial = new THREE.MeshPhongMaterial();
        
        let font = null;
        let text = "16";
        let textMesh;
        let group;
        const fontLoader = new TTFLoader();

        group = new THREE.Group();
        fontLoader.load('./Fonts/arial-bold.ttf', function (json) {
                font = new THREE.Font(json);
                createText();
        });

        function createText() {
            if(counter == 0){text = '0'}
            else if(counter > 0){text = counter.toString()}
            const textGeo = new THREE.TextBufferGeometry(text, {

                font: font,

                size: size,
                height: height,
                curveSegments: curveSegments,

                bevelThickness: bevelThickness,
                bevelSize: bevelSize,
                bevelEnabled: true

            });
            group.position.y = yAtomTotal/2+1;
            scene.add(group);

            textGeo.computeBoundingBox();
            textGeo.computeVertexNormals();

            const centerOffset = - 0.18 * (textGeo.boundingBox.max.x - textGeo.boundingBox.min.x);

            textMesh = new THREE.Mesh(textGeo, textMaterial);
            if(counter>halves[0]){textMesh.material.color.setHex(textColors[0]);}
            else if(counter>halves[1]){textMesh.material.color.setHex(textColors[1]);}
            else if(counter>halves[2]){textMesh.material.color.setHex(textColors[2]);}
            else if(counter>halves[3]){textMesh.material.color.setHex(textColors[3]);}
            else if(counter>halves[4]){textMesh.material.color.setHex(textColors[4]);}
            else if(counter>halves[5]){textMesh.material.color.setHex(textColors[5]);}
            textMesh.position.x = centerOffset;
            textMesh.position.y = hover;
            textMesh.position.z = 0;

            textMesh.rotation.x = 0;
            textMesh.rotation.y = Math.PI * 2;

            group.add(textMesh);

        }

        function generatePlane(){
            const planeGeometry = new THREE.PlaneBufferGeometry(20, 20, 100, 32);
            const planeMaterial = new THREE.MeshPhongMaterial({ color: 0xccddff, envMap: texture, refractionRatio: 0.9 });
            const plane = new THREE.Mesh(planeGeometry, planeMaterial);
        
            plane.receiveShadow = true;
            
            plane.position.z = -10;
            scene.add(plane);
        }

        function generateLights(){
            const light = new THREE.PointLight(0xffffff, 1.5, 100);
            light.position.set(0, 10, 10);
            const areaLight = new THREE.AmbientLight(0xccff33, 0.2);
            const light2 = new THREE.PointLight(0xffffff, 1.5, 100);
            light2.position.set(0, 10, -10);
            scene.add(light);
            scene.add(light2);
        }

        function createBackground(){
            var url = './images/simulation' + simulationNumber + '/';
            texture = loader.load([
                    url + 'px.png',
                    url + 'nx.png',
                    url + 'py.png',
                    url + 'ny.png',
                    url + 'pz.png',
                    url + 'nz.png',
                ],     
                function ( xhr ) {                                                                                    
                    if ( xhr.lengthComputable ) {                                                                                       
                    console.log( 'percent: ' + (xhr.loaded / xhr.total * 100) );                                                                                   
                    }
                },
                function ( err ) {                                                                                      
                    console.log( 'An error happened' );
                }
            );
            scene.background = texture;
            scene.rotation.y = 45 / Math.PI  ;
        }

        function refreshText() {
            if (counter == 0){text = '0';}
            group.remove(textMesh);
            createText();

        }

        function generateDecayInstance(x){
            do randNr = Math.random();
            while (randNr == 0 || randNr == 1);
            timeDecay[x] = (-1) * halfLife[simulationNumber - 1] / Math.LN2 * Math.log(randNr);
        }

        camera.position.z = -2;
        camera.position.x = 15;
        camera.position.y = 4;


        //draw the atoms in the correct position
        function rotateCubes(){
            for (var i = 0; i < atom.length; i++) {
                if(!decayed[i]){
                    atom[i].rotation.x += atomSpeed[i][0]*speedCountdown[i]; 
                    atom[i].rotation.y += atomSpeed[i][1]*speedCountdown[i];
                    atom[i].rotation.z += atomSpeed[i][2]*speedCountdown[i];
                }
            }
        }

        function generateAtoms(){
            for (var x = 0; x < atomTotal; x++) {
                materials[x] = new THREE.MeshStandardMaterial( {
                    color: originalColors[simulationNumber - 1],
                    metalness: 0.3,
                    roughness: 0,
                    envMap: texture,
                    flatShading: true,
                });
                atom[x] = new THREE.Mesh(geometries[simulationNumber-1], materials[x]);
                
                xRotation[x] = (Math.random()>0.5)? 1 : 0;
                yRotation[x] = (Math.random()>0.5)? 1 : 0;
                zRotation[x] = (Math.random()>0.5)? 1 : 0;
                
                while (xRotation[x] == 0 && yRotation[x] == 0 && zRotation[x] == 0){
                    xRotation[x] = (Math.random()>0.5)? 1 : 0;
                    yRotation[x] = (Math.random()>0.5)? 1 : 0;
                    zRotation[x] = (Math.random()>0.5)? 1 : 0;
                }

                atomSpeed[x] = [Math.random() * 0.02 * xRotation[x], Math.random() * 0.02 * yRotation[x], Math.random() * 0.02 * zRotation[x]];
                generateDecayInstance(x);
                decayed[x] = false;
                speedCountdown[x] = 1;
            }

            for (var k = 0, j = 0; j < yAtomTotal; j++) {
                for (var i = 0; i < xAtomTotal; i++, k++) {
                    var x = -xAtomTotal/2 + (xAtomTotal * 1.3 / xAtomTotal) * i //-3 is the leftest number; 8 is the units of space available (-3 to 5); 4 is the amount of atom in a row
                    var y = -yAtomTotal/2 + (yAtomTotal * 1.3 / yAtomTotal) * j //-2.5 is the bottom-most number; 6 is the units of space available (-2.5 to 3.5); 4 is the amount of atom in a column
                    atom[k].position.set(x, y, 0);
                    atom[k].name = k;
                    scene.add(atom[k]);
                }
            }
        }

        function decayCubes(){
            time1 = new Date();
            elapsedTime = (time1 - time0)/1000 - bufferTime + timeBoost ;
            for (var i = 0; i < atomTotal; i++) {                       
                if (decayed[i]) continue;                           //If the atom is decayed, break the for-loop          
                if (elapsedTime < timeDecay[i]){                    //If the elapsed time is still less than the amount of time required for the decay, break the for-loop
                    continue;
                }              
                if(speedCountdown[i]<killSpeed){
                    speedCountdown[i]+=Math.sqrt(speedCountdown[i]*0.2);
                    continue;
                }
                speedCountdown[i]=0;
                decayed[i] = true;                                  //If all other conditions are not met, decay the atom.       
                atom[i].material.color.setHex( decayColors[ simulationNumber - 1 ] );
                atom[i].scale.x = 0.7;
                atom[i].scale.y = 0.7;
                atom[i].scale.z = 0.7;    
                counter--;
                refreshText();                   
            }
        }

        function init() {
            var selected = document.getElementById("simPick");
            var choice = selected.value;
            createBackground();
            generateAtoms();
            generateLights();
            //generatePlane();
            running = false;
            renderer.render(scene, camera);

            controls.update();

        }

        function resetAtoms(){
            for(var x = 0; x < atomTotal ; x++){
                var selectedObject = scene.getObjectByName(x);
                scene.remove(selectedObject);
            }
        }

        function updateInit(){

            createBackground();
            resetAtoms();
            generateAtoms();
            resetDecay();
       
            renderer.render(scene, camera);

            controls.update();
        }

        document.getElementById("resetAnimation").addEventListener("click", function() {
            document.getElementById("startAnimation").innerHTML = "Start";
            resetDecay();
        });

        function pauseAnimation(){
            time0 = new Date();
            running = false;
        }

        function continueAnimation(){
            time1 = new Date();
            bufferTime += (time1 - time0)/1000;
            timeBoost = bufferTime + elapsedTime;
            time0 = new Date();
            time1 = new Date();
            running = true;
        }

        function resetDecay() {
            for (var i = 0; i < atomTotal; i++) {               //reset each atoms decay state, color, and give each atom a new time of decay.
                atom[i].material.color.setHex(originalColors[simulationNumber - 1]);
                atom[i].scale.x = 1;
                atom[i].scale.y = 1;
                atom[i].scale.z = 1;  
                decayed[i] = false;
                generateDecayInstance(i);
                speedCountdown[i]=1;
            }
            elapsedTime = 0;
            timeBoost = 0;
            bufferTime = 0;
            counter = atomTotal;
            refreshText();
            running = false;
        }

        function begin(){
            running = true;
            time0 = new Date();
            time1 = new Date();
        }

        const startButton = document.querySelector("#startAnimation");

        document.getElementById("startAnimation").addEventListener("click", function() {
            if(this.innerHTML == "Start"){
                begin();
                this.innerHTML = "Pause";
            }
            else if(this.innerHTML == "Pause"){
                pauseAnimation();
                this.innerHTML = "Continue";
            }
            else if(this.innerHTML == "Continue"){
                continueAnimation();
                this.innerHTML = "Pause";
            }
        });

        function animate() {
            requestAnimationFrame(animate);
            if(running){
                rotateCubes();
                decayCubes();
            }
            renderer.render(scene, camera);
            controls.update();
            document.getElementById("elapsed").innerHTML = elapsedTime.toFixed(2) + " seconds.";
        }

        init();
        animate();

        const simulationSelector = document.querySelector('#simPick');

        simulationSelector.addEventListener('change', (event) => {
            simulationNumber = parseInt(simulationSelector.value);
            updateInit();
            startButton.innerHTML = "Start";
        });

        function emitParticle(){

        }

    </script>
    <div class="controls">
        <button id="resetAnimation">RESET</button>&nbsp;&nbsp;&nbsp;&nbsp;<button id="startAnimation">Start</button>
        <div class="select-good" style="width:200px; margin-top: 5px;">
            <select id="simPick">
                <option value="1" selected="selected">Simulation 1</option>
                <option value="2">Simulation 2</option>
                <option value="3">Simulation 3</option>
                <option value="4">Simulation 4</option>
            </select>
        </div>
        <div class="elapsed" id="elapsed">0 seconds</div>
    </div>

        
</html>