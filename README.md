<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CCC 3D Viewer</title>
    <style>
        /* ... (body, app-container, top-nav, main-content styles remain the same) ... */

        /* Model Tree Panel (Left Column) - Color Change for Selection */
        #modelTreePanel {
            /* ... (keep existing panel styles) ... */
        }
        /* ... (keep existing panel-header, container, ul styles) ... */
        #modelTreePanel .tree-item {
            /* ... (keep existing tree-item styles) ... */
        }
        #modelTreePanel .tree-item:hover { background-color: #3f3f3f; }
        
        /* --- NEW SELECTION COLOR --- */
        #modelTreePanel .tree-item.selected { 
            background-color: #a0c4ff; /* Light blue */
            color: #000000; /* Black text */
        }
        
        /* ... (keep toggler, group-name, visibility-toggle styles) ... */

        /* Viewer Container (Right Column) */
        #viewer-container {
            flex-grow: 1;
            position: relative;
            overflow: hidden;
            /* The background color of the canvas is now set in JS */
        }
        #viewer-container canvas { /* ... */ }

        /* Info Panel (Positioned inside viewer-container) */
        #objectInfo {
            /* ... (keep existing styles: position, top, right, etc.) ... */
        }

        /* Loader */
        /* ... (keep existing loader styles) ... */

        /* --- NEW UI TOGGLE BUTTON --- */
        #toggle-ui-button {
            position: absolute;
            right: 20px;
            bottom: 20px;
            width: 48px;
            height: 48px;
            background-color: #2c3e50;
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 24px; /* Adjust icon size */
            z-index: 50;
            box-shadow: 0 2px 8px rgba(0,0,0,0.3);
            transition: background-color 0.2s ease;
        }
        #toggle-ui-button:hover {
            background-color: #34495e;
        }
    </style>
</head>
<body>
    <div id="app-container">
        <!-- ... (your #top-nav and #main-content structure remains the same) ... -->
        <div id="main-content">
            <div id="modelTreePanel">
                <!-- ... -->
            </div>
            <div id="viewer-container">
                <!-- ... -->
                <div id="objectInfo">None</div>
            </div>
        </div>
    </div>

    <!-- Add the new toggle button outside the main layout container -->
    <button id="toggle-ui-button" title="Toggle UI Panels">📊</button>

    <script type="module" src="script.js"></script>
</body>
</html>




// ... (Imports) ...

// --- Get UI Elements ---
const toggleUiButton = document.getElementById('toggle-ui-button'); // NEW
// ... (keep other getElementById calls)

// --- Global variables ---
// ... (keep existing global variables) ...
const highlightColorSingle = new THREE.Color(0xa0c4ff); // NEW: Light Blue (matches CSS)

// --- Scene Setup ---
const scene = new THREE.Scene();
// --- BACKGROUND COLOR CHANGE (Request ①) ---
// Option A: Simple Light Color
scene.background = new THREE.Color(0xe8e8e8); // A very light gray

// Option B: Gradient Background (More advanced)
// To achieve a gradient, we add a background plane with a custom shader.
// This will be done after the scene is created.
const backgroundGradientMaterial = new THREE.ShaderMaterial({
    uniforms: {
        color1: { value: new THREE.Color(0xadd8e6) }, // Light blue at top
        color2: { value: new THREE.Color(0xe8e8e8) }  // Light gray at bottom
    },
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
    `,
    fragmentShader: `
        uniform vec3 color1;
        uniform vec3 color2;
        varying vec2 vUv;
        void main() {
            gl_FragColor = vec4(mix(color2, color1, vUv.y), 1.0);
        }
    `,
    depthWrite: false, // Don't write to depth buffer
    side: THREE.BackSide
});

const backgroundGeometry = new THREE.SphereGeometry(5000, 32, 32); // A large sphere
const backgroundMesh = new THREE.Mesh(backgroundGeometry, backgroundGradientMaterial);
scene.add(backgroundMesh);
// Note: If you use the gradient, you can comment out `scene.background = ...`


// --- Camera, Renderer, Lighting, Controls Setup ---
// ... (This section remains unchanged) ...

// ... (All other functions from the previous full code version remain the same) ...
// parseObjHeader, fetchAllCategoryData, frameObject, buildAndPopulateCategorizedTree,
// createCategoryNode, createObjectNode, applyHighlight, removeAllHighlights,
// zoomToAndIsolate, deIsolateAllObjects, handleSelection, updateInfoPanel, main, animate, etc.

// --- Event Listeners for Panel Controls ---
// ... (keep closeModelTreeBtn, modelTreeSearch listeners) ...

// --- NEW: Event listener for the UI Toggle Button (Request ③) ---
if (toggleUiButton) {
    toggleUiButton.addEventListener('click', () => {
        // Check the current visibility of one of the panels to decide the action
        const isVisible = modelTreePanel.style.display !== 'none';
        
        if (isVisible) {
            // Hide panels
            if (modelTreePanel) modelTreePanel.style.display = 'none';
            if (objectInfoPanel) objectInfoPanel.style.display = 'none';
            toggleUiButton.textContent = '📊'; // Change icon to "show"
            toggleUiButton.title = "Show UI Panels";
        } else {
            // Show panels
            if (modelTreePanel) modelTreePanel.style.display = 'block';
            if (objectInfoPanel) objectInfoPanel.style.display = 'block';
            toggleUiButton.textContent = '❌'; // Change icon to "hide"
            toggleUiButton.title = "Hide UI Panels";
        }
    });
}


// --- Final Startup ---
// ... (The main() and animate() calls remain at the end) ...
main();
animate();
