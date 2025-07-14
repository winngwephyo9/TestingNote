<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Three.js OBJ Viewer</title>
    <style>
        /* ... (your existing body, canvas, objectInfo, loader, modelTreePanel styles) ... */

        /* --- Header / Navigation Bar Styles --- */
        #top-nav {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 50px; /* Adjust height as needed */
            background-color: #2c3e50; /* A dark blue-gray */
            color: white;
            display: flex;
            align-items: center;
            padding: 0 20px;
            box-sizing: border-box;
            z-index: 50; /* Above other UI but below popups */
        }
        #top-nav .nav-title {
            font-size: 1.2em;
            font-weight: bold;
            margin-right: 30px;
        }
        #top-nav label {
            margin-right: 10px;
        }
        #top-nav select {
            padding: 5px;
            border-radius: 4px;
            border: 1px solid #7f8c8d;
            background-color: #34495e;
            color: white;
            font-size: 1em;
        }
        /* Adjust main content to not be covered by nav bar */
        #objectInfo { top: 60px; }
        #modelTreePanel { top: 60px; max-height: calc(100vh - 70px); }

    </style>
</head>
<body>
    <!-- New Navigation Bar -->
    <div id="top-nav">
        <div class="nav-title">CCC 3D Viewer</div>
        <label for="model-selector">Select Model:</label>
        <select id="model-selector">
            <option value="GF">GF本社移転</option>
            <option value="BIKEN">BIKEN (Placeholder)</option>
            <!-- Add more models here -->
        </select>
    </div>

    <div id="loader-container">
        <div id="loader"></div>
        <div id="loader-text">Loading 3D Model...</div>
    </div>
    <div id="objectInfo">None</div>
    <div id="modelTreePanel">
        <!-- ... your existing panel content ... -->
    </div>

    <script type="module" src="script.js"></script>
</body>
</html>



// --- CORRECTED IMPORTS ---
import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';
import { OBJLoader } from './library/loaders/OBJLoader.js';
import { MTLLoader } from './library/loaders/MTLLoader.js';

// --- Get UI Elements ---
const modelSelector = document.getElementById('model-selector');
// ... (keep all other getElementById calls)

// --- Global variables ---
// ... (keep all global variables)

// --- Model Configuration ---
// Create an object to store the file paths for each model
const models = {
    'GF': {
        obj: '240324_GF本社移転_2022_20250618.obj',
        mtl: '240324_GF本社移転_2022_20250627.mtl',
        // You can add other model-specific settings here if needed
    },
    'BIKEN': {
        obj: 'biken_model.obj', // Placeholder filename
        mtl: 'biken_model.mtl', // Placeholder filename
    },
    // Add other models here
};

// --- Scene, Camera, Renderer, etc. Setup ---
// ... (Keep all of this setup as is) ...

// --- Helper Functions ---
// ... (Keep all helper functions: parseObjHeader, fetchAllCategoryData, frameObject, buildAndPopulateCategorizedTree, etc.) ...

// --- REFACTORED Main Application Flow into a Reusable Function ---
async function loadModel(modelKey) {
    // 0. Reset the scene and state
    if (loadedObjectModelRoot) {
        scene.remove(loadedObjectModelRoot);
    }
    // Clear all previous data
    loadedObjectModelRoot = null;
    selectedObjectOrGroup = null;
    originalMeshMaterials.clear();
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
    elementIdDataMap.clear();
    modelTreeList.innerHTML = '';
    if (modelTreePanel) modelTreePanel.style.display = 'none';
    updateInfoPanel(); // Clear the info panel

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        
        const modelFiles = models[modelKey];
        if (!modelFiles) {
            throw new Error(`Model key "${modelKey}" not found in configuration.`);
        }

        const assetPath = '/ccc/public/objFiles/';
        const objFileName = modelFiles.obj;
        const mtlFileName = modelFiles.mtl;
        const fullObjPath = assetPath + objFileName;
        
        // Step 1: Parse header info
        await parseObjHeader(fullObjPath);
        
        // Step 2: Load Materials
        if (loaderTextElement) loaderTextElement.textContent = `Loading Materials...`;
        const mtlLoader = new MTLLoader();
        mtlLoader.setPath(assetPath);
        const materialsCreator = await new Promise((resolve, reject) => {
            mtlLoader.load(mtlFileName, resolve, undefined, reject);
        });
        materialsCreator.preload();

        // Step 3: Load Geometry
        const objLoader = new OBJLoader();
        objLoader.setMaterials(materialsCreator);
        const object = await new Promise((resolve, reject) => {
            objLoader.load(fullObjPath, resolve, (xhr) => {
                if (loaderTextElement) {
                    const percent = Math.round(xhr.loaded / xhr.total * 100);
                    loaderTextElement.textContent = isFinite(percent) && percent < 100 ?
                        `Loading 3D Geometry: ${percent}%` : `Processing Geometry...`;
                }
            }, reject);
        });

        loadedObjectModelRoot = object;

        // Step 4: Process Model (center, scale, rotate)
        const initialBox = new THREE.Box3().setFromObject(object);
        const initialCenter = initialBox.getCenter(new THREE.Vector3());
        object.traverse((child) => {
            if (child.isMesh) {
                child.geometry.translate(-initialCenter.x, -initialCenter.y, -initialCenter.z);
                child.castShadow = true;
                child.receiveShadow = true;
            }
        });
        object.position.set(0, 0, 0);
        const scaledBox = new THREE.Box3().setFromObject(object);
        const maxDim = Math.max(scaledBox.getSize(new THREE.Vector3()).x, scaledBox.getSize(new THREE.Vector3()).y, scaledBox.getSize(new THREE.Vector3()).z);
        const desiredMaxDimension = 150;
        if (maxDim > 0) {
            const scale = desiredMaxDimension / maxDim;
            object.scale.set(scale, scale, scale);
        }
        object.rotation.x = -Math.PI / 2;
        object.rotation.y = -Math.PI / 2;

        // Step 5: Fetch category data
        const allIds = [];
        loadedObjectModelRoot.traverse(child => {
            if (child.name && child.parent === loadedObjectModelRoot) {
                const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                if (splitIndex > 0) allIds.push(child.name.substring(splitIndex + 1));
            }
        });
        await fetchAllCategoryData(parsedWSCenID, [...new Set(allIds)]);
        
        // Step 6: Build the tree UI
        buildAndPopulateCategorizedTree();
        
        // Step 7: Add model to scene, frame it, and hide loader
        scene.add(object);
        frameObject(object);
        
        if (loaderContainer) loaderContainer.style.display = 'none';

    } catch (error) {
        console.error('Failed to initialize the viewer:', error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model: ${modelKey}.`;
    }
}


// --- Event Listeners and Animation Loop ---
// ... (Keep your existing click, close, search, and resize listeners) ...

// --- NEW: Event listener for the model selector dropdown ---
if (modelSelector) {
    modelSelector.addEventListener('change', (event) => {
        const selectedModelKey = event.target.value;
        loadModel(selectedModelKey);
    });
}


function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}

// --- Start Application ---
// Load the default model when the page first loads
loadModel('GF'); // 'GF' is the default model
animate();
