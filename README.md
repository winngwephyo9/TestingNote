表示されているオブジェクトの一覧を表示することは可能ですか？
![image](https://github.com/user-attachments/assets/0f1fab81-9034-4638-8c39-03f6686f255d)

選択したら、そのオブジェクトに近寄って、他のオブジェクトを半透明とかって可能ですか？
![image](https://github.com/user-attachments/assets/21b4ebcd-c84a-44f7-98d7-7e6c5d34e7ee)


<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Three.js OBJ Viewer</title>
    <style>
        body { margin: 0; overflow: hidden; font-family: sans-serif; }
        canvas { display: block; }
        #objectInfo { /* Your existing styles for selection info */
            position: absolute;
            top: 10px;
            left: 10px;
            background-color: rgba(0,0,0,0.7);
            color: white;
            padding: 10px;
            border-radius: 5px;
            font-family: monospace;
            white-space: pre-wrap;
            max-height: 20vh; /* Adjusted max-height */
            overflow-y: auto;
            z-index: 10;
        }
        #loader-container { /* Container for loader elements */
            position: absolute;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            background-color: rgba(200, 200, 200, 0.5); /* Slight overlay */
            z-index: 1000; /* High z-index */
        }
        #loader { /* Spinner */
            border: 12px solid #f3f3f3;
            border-radius: 50%;
            border-top: 12px solid #3498db;
            width: 80px;
            height: 80px;
            animation: spin 2s linear infinite;
        }
        #loader-text { /* Text below spinner */
            margin-top: 20px;
            color: #333;
            font-size: 16px;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }

        /* --- Model Tree Panel Styles --- */
        #modelTreePanel {
            position: absolute;
            top: 10px;
            right: 10px; /* Positioned to the right */
            width: 300px; /* Adjust as needed */
            max-height: calc(100vh - 20px); /* Max height with some padding */
            background-color: #2c2c2c; /* Dark background */
            color: #e0e0e0; /* Light text */
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.3);
            overflow-y: auto;
            padding: 0;
            z-index: 20;
            display: none; /* Hidden by default, shown after OBJ loads */
        }
        #modelTreePanel .panel-header {
            padding: 12px 15px;
            font-weight: bold;
            border-bottom: 1px solid #444;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        #modelTreePanel .panel-header input[type="text"] {
            width: calc(100% - 30px);
            padding: 6px;
            background-color: #3a3a3a;
            border: 1px solid #555;
            color: #e0e0e0;
            border-radius: 4px;
            margin-top: 5px;
        }
        #modelTreePanel ul {
            list-style-type: none;
            padding: 0;
            margin: 0;
        }
        #modelTreePanel li {
            padding: 8px 15px;
            border-bottom: 1px solid #3a3a3a;
            display: flex;
            justify-content: space-between;
            align-items: center;
            cursor: default; /* No hand cursor for the li itself */
        }
        #modelTreePanel li:last-child {
            border-bottom: none;
        }
        #modelTreePanel .group-name {
            flex-grow: 1;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
            margin-right: 10px;
        }
        #modelTreePanel .visibility-toggle {
            cursor: pointer;
            font-size: 18px; /* Eye icon size */
        }
        #modelTreePanel .visibility-toggle.visible-icon::before {
            content: '👁️'; /* Simple eye emoji */
        }
        #modelTreePanel .visibility-toggle.hidden-icon::before {
            content: '🚫'; /* Simple crossed-out eye emoji or similar */
        }
    </style>
</head>
<body>
    <div id="loader-container"> <!-- Encapsulate loader elements -->
        <div id="loader"></div>
        <div id="loader-text">Loading 3D Model...</div>
    </div>
    <div id="objectInfo">None</div>
    <div id="modelTreePanel">
        <div class="panel-header">
            <span>モデル</span>
            <!-- <input type="text" id="modelTreeSearch" placeholder="検索..."> -->
        </div>
        <ul id="modelTreeList">
            <!-- Tree items will be populated here by JavaScript -->
        </ul>
    </div>

    <script type="module" src="script.js"></script>
</body>
</html>


import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';
import { OBJLoader } from './library/controls/OBJLoader.js';

// --- Get UI Elements ---
const loaderContainer = document.getElementById('loader-container'); // Updated
const loaderTextElement = document.getElementById('loader-text');
const modelTreePanel = document.getElementById('modelTreePanel');
const modelTreeList = document.getElementById('modelTreeList');
// const modelTreeSearch = document.getElementById('modelTreeSearch'); // For future search functionality

// --- Global variables for parsed OBJ header info ---
// ... (keep parsedWSCenID, parsedPJNo)

// --- Scene Setup, Camera, Renderer, Lighting, Controls ---
// ... (Keep all this setup as it was in the previous "all code" response) ...
const scene = new THREE.Scene(); // etc.

// --- Global variable to store the loaded OBJ object and its named groups ---
let loadedObjectModelRoot = null;
let namedObjGroups = []; // To store references to named groups from OBJ

// --- Function to parse the first line of OBJ ---
// ... (Keep your existing parseObjHeader function) ...
async function parseObjHeader(filePath) { /* ... */ }


// --- Function to Populate Model Tree ---
function populateModelTree() {
    if (!loadedObjectModelRoot || !modelTreeList) return;
    modelTreeList.innerHTML = ''; // Clear existing items
    namedObjGroups = []; // Reset

    // Iterate through direct children of the loaded OBJ root
    // These are assumed to be the groups from 'g' tags or named meshes
    loadedObjectModelRoot.children.forEach(child => {
        if ((child.isGroup || child.isMesh) && child.name) { // Must have a name to be listed
            namedObjGroups.push(child); // Store reference

            const listItem = document.createElement('li');
            
            const nameSpan = document.createElement('span');
            nameSpan.className = 'group-name';
            nameSpan.textContent = child.name; // Use the 'g' name or mesh name
            nameSpan.title = child.name; // Tooltip for long names

            const toggleButton = document.createElement('span');
            toggleButton.className = 'visibility-toggle';
            toggleButton.classList.add(child.visible ? 'visible-icon' : 'hidden-icon');
            toggleButton.title = child.visible ? 'Hide' : 'Show';
            
            toggleButton.addEventListener('click', (event) => {
                event.stopPropagation(); // Prevent li click if any
                child.visible = !child.visible;
                toggleButton.classList.toggle('visible-icon', child.visible);
                toggleButton.classList.toggle('hidden-icon', !child.visible);
                toggleButton.title = child.visible ? 'Hide' : 'Show';
            });

            listItem.appendChild(nameSpan);
            listItem.appendChild(toggleButton);
            modelTreeList.appendChild(listItem);
        }
    });

    if (modelTreePanel) modelTreePanel.style.display = 'block'; // Show panel
}


// --- OBJ Loading ---
const objLoader = new OBJLoader();
const objPath = './objFiles/standard_testing.obj'; // Your OBJ file path

if (loaderContainer) loaderContainer.style.display = 'flex'; // Show loader container

parseObjHeader(objPath).then(() => {
    objLoader.load(
        objPath,
        (object) => { // Success callback
            loadedObjectModelRoot = object;

            // ... (your existing OBJ processing: centering, scaling, rotation) ...
            // Ensure this processing is still in place
            const initialBox = new THREE.Box3().setFromObject(object); // ... and so on

            scene.add(object);
            console.log("OBJ loaded and processed by OBJLoader:", object);

            populateModelTree(); // Populate the tree after model is loaded and processed

            // ... (your existing camera adjustment logic) ...
            controls.update();
            updateInfoPanel(); // Update selection info

            if (loaderContainer) loaderContainer.style.display = 'none'; // Hide loader
        },
        (xhr) => { // Progress callback
            // console.log((xhr.loaded / xhr.total * 100) + '% loaded');
            if (loaderTextElement) {
                const percentLoaded = Math.round(xhr.loaded / xhr.total * 100);
                loaderTextElement.textContent = isFinite(percentLoaded) && percentLoaded > 0 ?
                    `Loading 3D Model: ${percentLoaded}%` : `Processing Model...`;
            }
        },
        (error) => { // Error callback
            console.error('An error happened while loading the OBJ file:', error);
            // ... (error message display) ...
            if (loaderContainer) loaderContainer.style.display = 'flex'; // Keep loader visible or change text
            if (loaderTextElement) loaderTextElement.textContent = 'Error loading model.';
        }
    );
});


// --- Object Selection Logic ---
// ... (Keep your latest working selection logic: selectedObjGroup, originalMeshMaterials, highlightedMeshesUuids,
//      raycaster, mouse, highlightColorSingle, applyHighlight, removeAllHighlights, click listener) ...
// Make sure this uses the refined click listener that correctly identifies selectedObjGroup
let selectedObjGroup = null;
// ... etc.

// --- Info Panel Update ---
// ... (Keep your existing updateInfoPanel that uses selectedObjGroup) ...

// --- Window Resize ---
// ... (Keep your existing resize listener) ...

// --- Animation Loop ---
function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}

// --- Start ---
animate();


