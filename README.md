// ... (Imports and UI Element selection) ...

// --- Scene Setup ---
const scene = new THREE.Scene();
// We will NOT set scene.background directly. The background will be a custom mesh.

// --- Autodesk-Style Gradient Background ---
// This creates a plane that always fills the camera's view.
const backgroundGeometry = new THREE.PlaneGeometry(2, 2, 1, 1);
const backgroundMaterial = new THREE.ShaderMaterial({
    // This GLSL code runs on the GPU
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = vec4(position.xy, 1.0, 1.0);
        }
    `,
    fragmentShader: `
        uniform vec3 topColor;
        uniform vec3 bottomColor;
        varying vec2 vUv;
        void main() {
            gl_FragColor = vec4(mix(bottomColor, topColor, vUv.y), 1.0);
        }
    `,
    uniforms: {
        // Define the colors for the gradient
        topColor: { value: new THREE.Color(0xd8e3ee) }, // Lighter sky blue
        bottomColor: { value: new THREE.Color(0xf0f0f0) } // Light gray ground
    },
    depthWrite: false // Don't interfere with the 3D objects
});

const backgroundMesh = new THREE.Mesh(backgroundGeometry, backgroundMaterial);
// Tell Three.js not to treat this mesh like a regular 3D object
backgroundMesh.renderOrder = -1; // Render it first (in the background)
scene.add(backgroundMesh);
// --- END Background Setup ---


// --- Camera Setup ---
// ... (rest of the file is unchanged) ...





const scene = new THREE.Scene();

// --- NEW BACKGROUND AND GROUND ---
// Set a light, slightly blue sky color for the background
scene.background = new THREE.Color(0xd8e3ee); // A soft, light blue-gray

// Add a GridHelper to act as the ground plane
const grid_size = 500; // The overall size of the grid
const grid_divisions = 50; // The number of divisions
const grid_color_center = 0x888888; // Color of the center lines (X and Z axes)
const grid_color_grid = 0xcccccc; // Color of the other grid lines

const gridHelper = new THREE.GridHelper(grid_size, grid_divisions, grid_color_center, grid_color_grid);
scene.add(gridHelper);



<img width="989" height="707" alt="image" src="https://github.com/user-attachments/assets/c887bd8b-e3e0-4592-a942-8b59b931ce00" />
