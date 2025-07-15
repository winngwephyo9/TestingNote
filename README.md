<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Three.js OBJ Viewer</title>
    <style>
        body { 
            margin: 0; 
            overflow: hidden; 
            font-family: sans-serif; 
            background-color: #f0f0f0; /* Overall page background */
        }

        /* Main container for the entire application layout */
        #app-container {
            display: flex;
            flex-direction: column;
            height: 100vh;
        }

        /* Header / Navigation Bar */
        #top-nav {
            flex-shrink: 0; /* Prevent shrinking */
            height: 50px;
            background-color: #2c3e50;
            color: white;
            display: flex;
            align-items: center;
            padding: 0 20px;
            box-sizing: border-box;
            z-index: 50;
        }
        #top-nav .nav-title { font-size: 1.2em; font-weight: bold; margin-right: 30px; }
        #top-nav label { margin-right: 10px; }
        #top-nav select {
            padding: 5px;
            border-radius: 4px;
            border: 1px solid #7f8c8d;
            background-color: #34495e;
            color: white;
            font-size: 1em;
        }

        /* Container for the two main columns */
        #main-content {
            display: flex;
            flex-grow: 1; /* Take up remaining vertical space */
            overflow: hidden; /* Prevent internal scrolling */
        }
        
        /* Model Tree Panel (Left Column) */
        #modelTreePanel {
            flex-basis: 320px; /* Set a base width */
            flex-shrink: 0; /* Don't shrink */
            display: flex; /* Use flexbox for its own layout */
            flex-direction: column;
            background-color: #2c2c2c;
            color: #e0e0e0;
            overflow: hidden; /* Hide overflow */
        }
        #modelTreePanel .panel-header {
            padding: 10px 15px;
            font-weight: bold;
            border-bottom: 1px solid #444;
            background-color: #383838;
            flex-shrink: 0; /* Header should not shrink */
        }
        #modelTreePanel .panel-header input[type="text"] {
            width: 100%;
            box-sizing: border-box;
            padding: 6px 8px;
            background-color: #4f4f4f;
            border: 1px solid #666;
            color: #e0e0e0;
            border-radius: 4px;
            margin-top: 8px;
        }
        #modelTreeContainer {
            flex-grow: 1; /* Tree list takes remaining space */
            overflow-y: auto; /* Allow the list to scroll */
        }
        #modelTreePanel ul { list-style-type: none; padding: 0; margin: 0; }
        #modelTreePanel .tree-item {
            padding: 6px 10px 6px 0;
            border-bottom: 1px solid #3a3a3a;
            display: flex;
            align-items: center;
            cursor: pointer;
        }
        /* ... (Keep other tree-item, toggler, etc. styles) ... */

        /* Viewer Container (Right Column) */
        #viewer-container {
            flex-grow: 1; /* Take up all remaining horizontal space */
            position: relative; /* Crucial for positioning child elements like the info panel */
            overflow: hidden;
        }
        #viewer-container canvas {
            display: block;
            width: 100%;
            height: 100%;
        }

        /* Info Panel (Positioned inside viewer-container) */
        #objectInfo {
            position: absolute;
            top: 10px;
            left: 10px;
            background-color: rgba(0,0,0,0.7);
            color: white;
            padding: 10px;
            border-radius: 5px;
            font-family: monospace;
            white-space: pre-wrap;
            z-index: 10;
        }

        /* Loader (Centered over the viewer-container) */
        #loader-container {
            position: absolute;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            background-color: rgba(204, 204, 204, 0.7); /* Light overlay */
            z-index: 1000;
        }
        #loader { /* Spinner */
            /* ... (keep loader styles) ... */
        }
        #loader-text { /* ... (keep loader-text styles) ... */ }

    </style>
</head>
<body>
    <div id="app-container">
        <div id="top-nav">
            <div class="nav-title">CCC 3D Viewer</div>
            <label for="model-selector">Select Model:</label>
            <select id="model-selector">
                <option value="GF">GF本社移転</option>
                <option value="BIKEN">BIKEN (Placeholder)</option>
            </select>
        </div>
        <div id="main-content">
            <div id="modelTreePanel">
                <div class="panel-header">
                    <span>モデル</span>
                    <input type="text" id="modelTreeSearch" placeholder="検索...">
                </div>
                <div id="modelTreeContainer">
                    <ul id="modelTreeList"></ul>
                </div>
            </div>
            <div id="viewer-container">
                <!-- Three.js canvas will be appended here by the script -->
                <div id="loader-container">
                    <div id="loader"></div>
                    <div id="loader-text">Loading 3D Model...</div>
                </div>
                <div id="objectInfo">None</div>
            </div>
        </div>
    </div>

    <script type="module" src="script.js"></script>
</body>
</html>




// --- Get UI Elements ---
const viewerContainer = document.getElementById('viewer-container'); // NEW: Get the right-side container
// ... (keep other getElementById calls)

// ... (Global variables, Scene, Camera setup remain the same) ...

// --- Renderer Setup ---
const renderer = new THREE.WebGLRenderer({ antialias: true });
// renderer.setSize(window.innerWidth, window.innerHeight); // <-- We will now set size based on the container
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
viewerContainer.appendChild(renderer.domElement); // <-- Append canvas to the new container

// --- Controls Setup ---
const controls = new OrbitControls(camera, renderer.domElement); // Controls listen on the canvas
// ... (keep controls properties)

// ... (All other functions like parseObjHeader, fetchAllCategoryData, buildAndPopulateCategorizedTree,
//      handleSelection, zoomToAndIsolate, etc., DO NOT NEED to be changed.) ...

// --- Window Resize (MODIFIED) ---
// This function now correctly resizes the renderer based on its container's dimensions
function onWindowResize() {
    const { clientWidth, clientHeight } = viewerContainer;

    camera.aspect = clientWidth / clientHeight;
    camera.updateProjectionMatrix();

    renderer.setSize(clientWidth, clientHeight);
}
window.addEventListener('resize', onWindowResize);

// --- Main Application Flow ---
async function main() {
    // ... (This function remains the same as the previous version) ...
}

// --- Animation Loop ---
function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}

// --- Start Application ---
// Initial resize to set the correct size on page load
onWindowResize(); 
main();
animate();
