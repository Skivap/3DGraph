    <script lang="ts">
        import { browser } from '$app/environment';
        import { onMount } from 'svelte';
        import * as THREE from 'three';
        import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
        import { ComputeEngine } from "@cortex-js/compute-engine";

        const vertexShader = `
            varying float vX;  
            varying float vY;  
            varying float vZ;  

            void main() {
                vec4 modelPosition = vec4(position, 1.0);  // Use model local coordinates
                vZ = modelPosition.z;
                vX = modelPosition.x;
                vY = modelPosition.y;
                gl_Position = projectionMatrix * modelViewMatrix * modelPosition;
            }
            `;

            const fragmentShader = `
            varying float vX;  
            varying float vY;  
            varying float vZ;  
            uniform bool useRed;
            uniform bool useGreen;
            uniform bool useBlue;
            uniform float initRed;
            uniform float initGreen;
            uniform float initBlue;

            uniform float maxZ;
            uniform float minZ;

            void main() {
                float red = initRed;
                float green = initGreen;
                float blue = initBlue;

                if(useRed){
                    red += (1.0 - initRed) * (maxZ - vZ) / (maxZ - minZ);
                }
                if(useGreen){
                    red += (1.0 - initGreen) * (maxZ - vZ) / (maxZ - minZ);
                }
                if(useBlue){
                    red += (1.0 - initBlue) * (maxZ - vZ) / (maxZ - minZ);
                }

                gl_FragColor = vec4(red, green, blue, 1.0);

                // if(vZ > 10.0 || vZ < -10.0){
                //     gl_FragColor = vec4(1.0, 1.0, 1.0, 0.0);
                // }
            }
            `;

        let all_formula: string[] = [
            "\\sin(x) * \\cos(y)",
        ];
        let formula = "";
        const size = 20;
        const splitter = 64;

        const scene = new THREE.Scene();
        const ce = new ComputeEngine();
        
        var docNotes = null
        let inputFormula = "";

        function addDiv(str:string){
            let parentDiv = document.getElementById("notes");
            const newDiv = document.createElement('div');
            newDiv.className = 'child-div';  // Add class name
            newDiv.textContent = str;  // Add text content
            parentDiv?.appendChild(newDiv);  // Append the child div to the parent div
        }

        function onClickButton(){
            if (inputFormula.trim() !== "") {
                all_formula.push(inputFormula);
                formula = inputFormula;
                addDiv(formula);
                createGraph();
                inputFormula = "";  // Clear the input after adding
            }
            all_formula.push(inputFormula);
        }

        function createGraph() {

            console.log("Creating Mesh ...");

            const geometry = new THREE.PlaneGeometry(size, size, splitter, splitter);
            const materialGrid = new THREE.MeshBasicMaterial({
                color: 0x000000, // Hex code for black
                wireframe: true,
                opacity: 0.05,
                transparent: true 
            });                

            var useRed:boolean = Math.random() < 0.5;
            var useGreen:boolean = Math.random() < 0.5;
            var useBlue:boolean = (useRed || useGreen) ? (Math.random() < 0.5) : true;

            var maxZ = -Infinity;
            var minZ = Infinity;

            const material = new THREE.ShaderMaterial({
                vertexShader,
                fragmentShader,
                side: THREE.DoubleSide,
                transparent: true,
                uniforms: {
                    maxZ: { value: 0 },
                    minZ: { value: 0 },
                    useRed: { value: useRed},
                    initRed: { value: useRed ? 0.2 : 0.8 },
                    useGreen: { value: useGreen },
                    initGreen: { value: useRed ? 0.2 : 0.8 },
                    useBlue: { value: useBlue },
                    initBlue: { value: useRed ? 0.2 : 0.8 },
                },
            });
            const mesh = new THREE.Mesh(geometry, material);
            const mesh2 = new THREE.Mesh(geometry, materialGrid);
            mesh.castShadow = true;  // This mesh casts shadows
            mesh.receiveShadow = true; 

            scene.add(mesh);
            // scene.add(mesh2);

            // Modify vertex positions
            const positions = geometry.attributes.position;
            const vertexCount = positions.count;
            for (let i = 0; i < vertexCount; i++) {
                const x = positions.getX(i);
                const y = positions.getY(i);
                ce.assign('x', x );
                ce.assign('y', y );
                const z = Number(ce.parse(formula).N().value);
                positions.setZ(i, z);
                if (z > maxZ) {
                    maxZ = z;
                }
                if(z < minZ){
                    minZ = z;
                }
            }
            // geometry.scale(1.0, 1.0, size/(maxZ * 2))

            material.uniforms.maxZ.value =  maxZ;
            material.uniforms.minZ.value =  minZ;
            positions.needsUpdate = true;

            console.log("Finish Mesh ...");
        }

        onMount(() => {

            docNotes = document.getElementById("notes");

            const container = document.getElementById('container');
            const renderer = new THREE.WebGLRenderer();
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setClearColor(0xffffff);
            renderer.shadowMap.enabled = true;  // Enable shadow mapping
            renderer.shadowMap.type = THREE.PCFSoftShadowMap;

            if (container) {
                container.appendChild(renderer.domElement);
            }

            const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(15.0, 2.0, 5.0);
            camera.up.set(0, 0, 1);
            camera.lookAt(0, 0, 0);


            const controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true; 
            renderer.sortObjects = true;

            // Add axes helper
            const axesHelper = new THREE.AxesHelper(5);
            scene.add(axesHelper);

            // Function to create graph
            

            function addGridHelper() {
                const divisions = size;  // One line per unit
                const gridHelper = new THREE.GridHelper(size, divisions, 0x888888, 0x888888);
                gridHelper.position.y = 0;  // Align it with the base of your mesh if needed
                gridHelper.rotateX(Math.PI / 2);
                scene.add(gridHelper);
            }

            addGridHelper();

            for (let str of all_formula) {
                formula = str;
                console.log(formula);
                addDiv(formula);
                createGraph();
            }
            

            // const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
            // directionalLight.position.set(0, 10, 0);
            // directionalLight.lookAt(0, 0, 0);
            // directionalLight.castShadow = true;  // Enable shadows for this light

            // // Configure shadow settings
            // directionalLight.shadow.mapSize.width = size;  
            // directionalLight.shadow.mapSize.height = size; 
            // directionalLight.shadow.camera.near = 0.5;
            // directionalLight.shadow.camera.far = size*2;
            // directionalLight.intensity = 0.5; 

            // scene.add(directionalLight);

            function animate() {
                requestAnimationFrame(animate);
                controls.update();
                renderer.render(scene, camera);
            }

            animate();
        });
    </script>

    <style>
        :global(html, body) {
            margin: 0;
            height: 100%;
        }

        .notes{
            top:0;
            left: 0;
            width: 100px;
            height: 100vh;
            position: absolute;
            z-index: 1;
            background-color: lightgray;
        }
        .card{
            background-color: black;
            color: white;
            margin:10px;
            width: 100px;
            height: 50px;
        }
    </style>

    <svelte:head>
        {#if browser}
            <script
                type="text/javascript"
                src="https://cdn.jsdelivr.net/npm/mathbox@latest/build/bundle/mathbox.js"
            ></script>
            
            <!-- Include the MathBox CSS: -->
            <link
                rel="stylesheet"
                href="https://cdn.jsdelivr.net/npm/mathbox@latest/build/mathbox.css"
            />
        {/if}
    </svelte:head>

    <div id="container" style="width: 100%; height: 100vh;"></div>
    <div id="notes" class="notes">
        <div>
            <input type="text" id="fname" name="fname" bind:value={inputFormula} ><br><br>
            <button on:click={() => {onClickButton();}}>Submit</button>
        </div>
        
    </div>