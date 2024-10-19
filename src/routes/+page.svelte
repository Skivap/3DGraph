<script lang="ts">
    import { browser } from '$app/environment';
    import { onMount } from 'svelte';
    import * as THREE from 'three';
    import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
    import { ComputeEngine } from "@cortex-js/compute-engine";

    onMount(() => {
        const ce = new ComputeEngine();
        const container = document.getElementById('container');
        const renderer = new THREE.WebGLRenderer();
        renderer.setSize(window.innerWidth, window.innerHeight);

        if (container) {
            container.appendChild(renderer.domElement);
        }

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.z = 5;

        const controls = new OrbitControls(camera, renderer.domElement);

        // Add axes helper
        const axesHelper = new THREE.AxesHelper(5);
        scene.add(axesHelper);

        // Setup lighting
        const ambientLight = new THREE.AmbientLight(0x404040);
        scene.add(ambientLight);
        const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
        directionalLight.position.set(1, 1, 1);
        scene.add(directionalLight);

        // Function to create graph
        function createGraph() {
            const geometry = new THREE.PlaneGeometry(2 * Math.PI, 2 * Math.PI, 64, 64);
            const material = new THREE.MeshBasicMaterial({ color: 0xff0000, wireframe: true });
            const mesh = new THREE.Mesh(geometry, material);
            scene.add(mesh);

            // Modify vertex positions
            const positions = geometry.attributes.position;
            const vertexCount = positions.count;
            for (let i = 0; i < vertexCount; i++) {
                const x = positions.getX(i);
                const y = positions.getY(i);
                ce.assign('x', x );
                ce.assign('y', y );
                const z = Number(ce.parse("x^2 + y^2").N().value);
                positions.setZ(i, z);
            }
            positions.needsUpdate = true;
        }

        createGraph();

        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            renderer.render(scene, camera);
        }

        animate();
    });
</script>

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