<style>
    /* ... (your existing styles for body, canvas, objectInfo, loader) ... */

    #modelTreePanel {
        position: absolute;
        top: 10px;
        /* left: 10px; */ /* Let's move it to the right for this example */
        right: 10px;
        width: 300px;
        max-height: calc(100vh - 20px);
        background-color: #2c2c2c;
        color: #e0e0e0;
        border-radius: 8px;
        box-shadow: 0 2px 10px rgba(0,0,0,0.3);
        overflow: auto; /* For scrolling the panel itself */
        padding: 0;
        z-index: 20;
        display: none; /* Hidden by default */
        font-size: 14px;
    }
    #modelTreePanel .panel-header {
        padding: 10px 15px;
        font-weight: bold;
        border-bottom: 1px solid #444;
        display: flex;
        justify-content: space-between;
        align-items: center;
        background-color: #383838;
    }
    #modelTreePanel .panel-header input[type="text"] { /* Search input */
        width: calc(100% - 70px); /* Adjust width as needed */
        padding: 6px 8px;
        background-color: #4f4f4f;
        border: 1px solid #666;
        color: #e0e0e0;
        border-radius: 4px;
        margin-left: 10px;
    }
    #modelTreePanel .close-button {
        cursor: pointer;
        font-size: 20px;
        line-height: 1;
    }
    #modelTreeContainer { /* For scrolling the tree content */
        max-height: calc(100vh - 60px); /* Adjust based on header height */
        overflow-y: auto;
    }
    #modelTreePanel ul {
        list-style-type: none;
        padding: 0;
        margin: 0;
    }
    #modelTreePanel li {
        /* Direct li styling is minimal, structure is more important */
    }
    #modelTreePanel .tree-item {
        padding: 6px 10px 6px 0; /* Add padding-left dynamically for indentation */
        border-bottom: 1px solid #3a3a3a;
        display: flex;
        align-items: center;
        cursor: pointer; /* Make items clickable for selection/zoom */
    }
    #modelTreePanel .tree-item:hover {
        background-color: #3f3f3f;
    }
    #modelTreePanel .tree-item.selected {
        background-color: #007bff;
        color: white;
    }
    #modelTreePanel .toggler {
        display: inline-block;
        width: 20px; /* Space for toggler */
        text-align: center;
        cursor: pointer;
        margin-right: 5px;
        user-select: none; /* Prevent text selection on toggle click */
    }
    #modelTreePanel .toggler.empty-toggler { /* For items with no children */
        color: transparent; /* Make it invisible but take up space */
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
        font-size: 16px; /* Eye icon size */
        margin-left: auto; /* Push to the right */
        padding: 0 5px;
    }
    #modelTreePanel .visibility-toggle.visible-icon::before { content: '👁️'; }
    #modelTreePanel .visibility-toggle.hidden-icon::before { content: '🚫'; }
    #modelTreePanel ul ul { /* Nested lists for hierarchy */
        padding-left: 0; /* Indentation handled by .tree-item padding */
    }
</style>
</head>
<body>
    <!-- ... loader, objectInfo ... -->
    <div id="modelTreePanel">
        <div class="panel-header">
            <span>モデル</span>
            <input type="text" id="modelTreeSearch" placeholder="検索...">
            <span class="close-button" id="closeModelTreeBtn" title="Close">×</span>
        </div>
        <div id="modelTreeContainer">
            <ul id="modelTreeList"></ul>
        </div>
    </div>
    <!-- ... script tag ... -->
</body>
</html>

import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';
import { OBJLoader } from './library/controls/OBJLoader.js';

// --- Get UI Elements ---
const loaderContainer = document.getElementById('loader-container');
const loaderTextElement = document.getElementById('loader-text');
const modelTreePanel = document.getElementById('modelTreePanel');
const modelTreeList = document.getElementById('modelTreeList');
const modelTreeSearch = document.getElementById('modelTreeSearch');
const closeModelTreeBtn = document.getElementById('closeModelTreeBtn');
const objectInfoPanel = document.getElementById('objectInfo');


// --- Global variables ---
let parsedWSCenID = "";
let parsedPJNo = "";
let loadedObjectModelRoot = null;
let initialCameraPosition = new THREE.Vector3(10, 10, 10);
let initialCameraLookAt = new THREE.Vector3(0, 0, 0);

// Selection & Highlighting & Isolate
let selectedObjectOrGroup = null; // Stores the currently selected Mesh or Group
const originalMeshMaterials = new Map(); // Key: mesh.uuid, Value: original material(s) used for HIGHLIGHTING
const originalObjectPropertiesForIsolate = new Map(); // Key: object.uuid, Value: { material, visible, opacity, transparent } for ISOLATE mode
let isIsolateModeActive = false;

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const highlightColorSingle = new THREE.Color(0xffff00); // Yellow

// --- Scene Setup ---
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xcccccc);

// --- Camera Setup ---
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 20000);
camera.position.copy(initialCameraPosition);
camera.lookAt(initialCameraLookAt);

// --- Renderer Setup ---
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

// --- Lighting Setup ---
const ambientLight = new THREE.AmbientLight(0x606060, 2);
scene.add(ambientLight);
const directionalLight = new THREE.DirectionalLight(0xFFFFFF, 2.5);
directionalLight.position.set(50, 100, 75);
directionalLight.castShadow = true;
directionalLight.shadow.mapSize.width = 2048;
directionalLight.shadow.mapSize.height = 2048;
directionalLight.shadow.camera.near = 0.5;
directionalLight.shadow.camera.far = 500;
directionalLight.shadow.camera.left = -100;
directionalLight.shadow.camera.right = 100;
directionalLight.shadow.camera.top = 100;
directionalLight.shadow.camera.bottom = -100;
scene.add(directionalLight);
const hemiLight = new THREE.HemisphereLight(0xffffff, 0x8d8d8d, 1.5);
hemiLight.position.set(0, 50, 0);
scene.add(hemiLight);

// --- Controls Setup ---
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;
controls.screenSpacePanning = false;
controls.minDistance = 1;
controls.maxDistance = 10000;

// --- Function to parse the first line of OBJ ---
async function parseObjHeader(filePath) {
    try {
        const response = await fetch(filePath);
        if (!response.ok) {
            console.error(`Failed to fetch OBJ for header parsing: ${response.statusText}`);
            return;
        }
        const text = await response.text();
        const lines = text.split(/\r?\n/);
        if (lines.length > 0) {
            const firstLine = lines[0].trim();
            console.log("First line of OBJ:", firstLine);
            if (firstLine.startsWith("# ")) {
                const content = firstLine.substring(2).trim();
                const pattern1Match = content.match(/^([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})_([a-zA-Z0-9]+)$/);
                if (pattern1Match) {
                    parsedWSCenID = pattern1Match[1];
                    parsedPJNo = pattern1Match[2];
                    console.log(`Pattern 1 Matched: WSCenID=${parsedWSCenID}, PJNo=${parsedPJNo}`);
                    return;
                }
                if (content.includes("ワークシェアリングされてない") && (content.includes("_PJNo無し") || content.includes("＿PJNo無し"))) {
                    parsedWSCenID = "";
                    parsedPJNo = "";
                    console.log(`Pattern 2 Matched: WSCenID=${parsedWSCenID}, PJNo=${parsedPJNo}`);
                    return;
                }
            }
        }
        console.log("First line did not match expected patterns for WSCenID/PJNo.");
    } catch (error) {
        console.error("Error fetching or parsing OBJ header:", error);
    }
}

// --- Function to Populate Model Tree ---
function buildTreeRecursive(object, parentULElement, depth = 0) {
    // Skip if it's an unnamed mesh with no children (unless it's the root of selection)
    if (!object.name && object.isMesh && object.children.length === 0 && object !== loadedObjectModelRoot) return;
    // Skip unnamed empty groups
    if (!object.name && object.isGroup && object.children.length === 0) return;


    const listItem = document.createElement('li');
    listItem.dataset.uuid = object.uuid; // Store uuid for easier selection later
    const itemContent = document.createElement('div');
    itemContent.className = 'tree-item';
    itemContent.style.paddingLeft = `${depth * 15 + 10}px`; // Indentation

    const hasVisibleChildrenForToggler = object.children.some(child => (child.isGroup && child.name) || (child.isMesh && child.name) || (child.isGroup && !child.name && child.children.length > 0) );

    const toggler = document.createElement('span');
    toggler.className = 'toggler';
    if (object.isGroup && hasVisibleChildrenForToggler) {
        toggler.textContent = '▶';
    } else {
        toggler.classList.add('empty-toggler');
        toggler.innerHTML = ' '; // Non-breaking space for alignment
    }
    itemContent.appendChild(toggler);

    const nameSpan = document.createElement('span');
    nameSpan.className = 'group-name';
    nameSpan.textContent = object.name || (object.isMesh ? "Unnamed Mesh" : "Unnamed Group (contains children)");
    nameSpan.title = object.name || (object.isMesh ? "Unnamed Mesh" : "Unnamed Group (contains children)");
    itemContent.appendChild(nameSpan);

    const visibilityToggle = document.createElement('span');
    visibilityToggle.className = 'visibility-toggle';
    visibilityToggle.classList.add(object.visible ? 'visible-icon' : 'hidden-icon');
    visibilityToggle.title = object.visible ? 'Hide' : 'Show';
    itemContent.appendChild(visibilityToggle);

    listItem.appendChild(itemContent);
    parentULElement.appendChild(listItem);

    let subList = null;
    if (object.isGroup && hasVisibleChildrenForToggler) {
        subList = document.createElement('ul');
        subList.style.display = 'none';
        listItem.appendChild(subList);

        toggler.addEventListener('click', (e) => {
            e.stopPropagation();
            const isCollapsed = subList.style.display === 'none';
            subList.style.display = isCollapsed ? 'block' : 'none';
            toggler.textContent = isCollapsed ? '▼' : '▶';
        });
    }

    itemContent.addEventListener('click', () => {
        console.log("Tree item clicked:", object.name || object.uuid);
        removeAllHighlights(); // Always remove previous highlights
        deIsolateAllObjects();   // Always remove previous isolation

        if (selectedObjectOrGroup === object) { // Clicked same object
            selectedObjectOrGroup = null; // Deselect
             // Tree item selection class removed below
        } else {
            selectedObjectOrGroup = object;
            applyHighlight(selectedObjectOrGroup, highlightColorSingle);
            zoomToAndIsolate(selectedObjectOrGroup); // Zoom and isolate this one
        }
        
        document.querySelectorAll('#modelTreePanel .tree-item.selected').forEach(el => el.classList.remove('selected'));
        if(selectedObjectOrGroup) { // Only add class if an object is actually selected
            itemContent.classList.add('selected');
        }
        updateInfoPanel();
    });

    visibilityToggle.addEventListener('click', (e) => {
        e.stopPropagation();
        object.visible = !object.visible;
        visibilityToggle.classList.toggle('visible-icon', object.visible);
        visibilityToggle.classList.toggle('hidden-icon', !object.visible);
        visibilityToggle.title = object.visible ? 'Hide' : 'Show';
        if (!object.visible && selectedObjectOrGroup && selectedObjectOrGroup.uuid === object.uuid) {
            removeAllHighlights();
            deIsolateAllObjects();
            selectedObjectOrGroup = null;
            updateInfoPanel();
            itemContent.classList.remove('selected');
        }
    });

    if (object.isGroup && hasVisibleChildrenForToggler && subList) {
        object.children.forEach(child => {
            buildTreeRecursive(child, subList, depth + 1);
        });
    }
}

function populateModelTree() {
    if (!loadedObjectModelRoot || !modelTreeList) return;
    modelTreeList.innerHTML = '';
    // Build tree starting from the children of the loaded root, or the root itself if named
    if (loadedObjectModelRoot.name) { // If the root itself is a named group
         buildTreeRecursive(loadedObjectModelRoot, modelTreeList, 0);
    } else { // If root is unnamed, iterate its children
        loadedObjectModelRoot.children.forEach(child => {
            buildTreeRecursive(child, modelTreeList, 0);
        });
    }
    if (modelTreePanel) modelTreePanel.style.display = 'block';
}

// --- OBJ Loading ---
if (loaderContainer) loaderContainer.style.display = 'flex';
parseObjHeader(objPath).then(() => {
    objLoader.load(
        objPath,
        (object) => {
            loadedObjectModelRoot = object;
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
            const scaledSize = scaledBox.getSize(new THREE.Vector3());
            const maxDim = Math.max(scaledSize.x, scaledSize.y, scaledSize.z);
            const desiredMaxDimension = 50;
            if (maxDim > 0) {
                const scale = desiredMaxDimension / maxDim;
                object.scale.set(scale, scale, scale);
            }
            object.rotation.x = -Math.PI / 2;
            object.rotation.y = -Math.PI / 2;
            scene.add(object);
            console.log("OBJ loaded by OBJLoader:", loadedObjectModelRoot);

            populateModelTree();

            const finalWorldBox = new THREE.Box3().setFromObject(object);
            const finalWorldCenter = finalWorldBox.getCenter(new THREE.Vector3());
            initialCameraLookAt.copy(finalWorldCenter);
            controls.target.copy(finalWorldCenter);
            const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
            const aspect = camera.aspect;
            const effectiveSizeDimension = Math.max(finalWorldBox.getSize(new THREE.Vector3()).y, finalWorldBox.getSize(new THREE.Vector3()).x / aspect);
            let cameraDistance = effectiveSizeDimension / (2 * Math.tan(fovInRadians / 2));
            const zoomOutFactor = 1.5;
            cameraDistance *= zoomOutFactor;
            cameraDistance = Math.max(cameraDistance, maxDim * 0.75);
            const cameraDirection = new THREE.Vector3(1, 0.6, 1).normalize();
            initialCameraPosition.copy(finalWorldCenter).addScaledVector(cameraDirection, cameraDistance);
            camera.position.copy(initialCameraPosition);
            camera.lookAt(initialCameraLookAt);
            controls.update();
            updateInfoPanel();
            if (loaderContainer) loaderContainer.style.display = 'none';
        },
        (xhr) => {
            if (loaderTextElement) {
                const percentLoaded = Math.round(xhr.loaded / xhr.total * 100);
                loaderTextElement.textContent = isFinite(percentLoaded) && percentLoaded > 0 ?
                    `Loading 3D Model: ${percentLoaded}%` : `Processing Model...`;
            }
        },
        (error) => {
            console.error('An error happened:', error);
            if (loaderContainer) loaderContainer.style.display = 'flex';
            if (loaderTextElement) loaderTextElement.textContent = 'Error loading model.';
        }
    );
});

// --- Selection, Highlighting, Isolation Logic ---
const applyHighlight = (target, color) => {
    if (!target) return;
    const meshesToHighlight = [];
    if (target.isMesh) meshesToHighlight.push(target);
    else if (target.isGroup) target.traverse(child => { if (child.isMesh) meshesToHighlight.push(child); });

    meshesToHighlight.forEach(mesh => {
        if (mesh.material) {
            if (!originalMeshMaterials.has(mesh.uuid)) {
                originalMeshMaterials.set(mesh.uuid, mesh.material);
            }
            if (Array.isArray(mesh.material)) {
                mesh.material = mesh.material.map(originalMat => {
                    const highlightMat = originalMat.clone();
                    if (highlightMat.color) highlightMat.color.set(color);
                    else if (highlightMat.emissive) highlightMat.emissive.set(color);
                    return highlightMat;
                });
            } else {
                const highlightMat = mesh.material.clone();
                if (highlightMat.color) highlightMat.color.set(color);
                else if (highlightMat.emissive) highlightMat.emissive.set(color);
                mesh.material = highlightMat;
            }
        }
    });
};

const removeAllHighlights = () => {
    originalMeshMaterials.forEach((originalMaterialOrArray, meshUuid) => {
        const mesh = scene.getObjectByProperty('uuid', meshUuid);
        if (mesh && mesh.isMesh) mesh.material = originalMaterialOrArray;
    });
    originalMeshMaterials.clear();
};

function zoomToAndIsolate(targetObject) {
    if (!targetObject) return;
    deIsolateAllObjects(); // Ensure previous isolation is cleared
    isIsolateModeActive = true;

    const box = new THREE.Box3().setFromObject(targetObject);
    if (box.isEmpty()) { // Handle cases where bounding box might be empty (e.g. empty group)
        console.warn("Cannot zoom to object with empty bounding box:", targetObject.name);
        isIsolateModeActive = false; // Don't proceed with isolation if no bounds
        return;
    }
    const center = box.getCenter(new THREE.Vector3());
    const size = box.getSize(new THREE.Vector3());
    const maxDim = Math.max(size.x, size.y, size.z);

    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    const aspect = camera.aspect;
    const effectiveSize = Math.max(size.y, size.x / aspect);
    let distance = effectiveSize / (2 * Math.tan(fovInRadians / 2));
    distance = Math.max(distance, maxDim * 1.2); // Ensure some padding
    distance *= 1.5; // Zoom out factor

    const offsetDirection = camera.position.clone().sub(controls.target).normalize();
    if (offsetDirection.lengthSq() === 0) offsetDirection.set(0.5, 0.5, 1).normalize(); // Default if looking straight at target

    camera.position.copy(center).addScaledVector(offsetDirection, distance);
    controls.target.copy(center);
    controls.update();

    loadedObjectModelRoot.traverse((object) => { // Traverse the whole loaded model
        if (object.isMesh) {
            let isPartOfSelectedTarget = false;
            let temp = object;
            while(temp && temp !== loadedObjectModelRoot.parent /* scene */) {
                if (temp === targetObject) {
                    isPartOfSelectedTarget = true;
                    break;
                }
                temp = temp.parent;
            }
             // If targetObject is a mesh itself
            if (object === targetObject) isPartOfSelectedTarget = true;


            if (!isPartOfSelectedTarget) {
                if (!originalObjectPropertiesForIsolate.has(object.uuid)) {
                    originalObjectPropertiesForIsolate.set(object.uuid, {
                        material: object.material,
                        visible: object.visible
                    });
                }
                if (object.visible) {
                    if (Array.isArray(object.material)) {
                        object.material = object.material.map(mat => {
                            const newMat = mat.clone();
                            newMat.transparent = true;
                            newMat.opacity = 0.1;
                            return newMat;
                        });
                    } else {
                        const newMat = object.material.clone();
                        newMat.transparent = true;
                        newMat.opacity = 0.1;
                        object.material = newMat;
                    }
                }
            } else { // Part of selected target
                if (originalObjectPropertiesForIsolate.has(object.uuid)) {
                    // If it was previously made translucent, restore it
                    const props = originalObjectPropertiesForIsolate.get(object.uuid);
                    object.material = props.material;
                    object.visible = props.visible; // Restore original visibility
                    originalObjectPropertiesForIsolate.delete(object.uuid); // Clean up
                } else { // Ensure it's fully opaque and visible
                     if (Array.isArray(object.material)) {
                        object.material.forEach(mat => { mat.transparent = false; mat.opacity = 1.0; });
                    } else {
                        object.material.transparent = false; object.material.opacity = 1.0;
                    }
                    object.visible = true;
                }
            }
        }
    });
}

function deIsolateAllObjects() {
    if (!isIsolateModeActive) return;
    originalObjectPropertiesForIsolate.forEach((props, uuid) => {
        const object = scene.getObjectByProperty('uuid', uuid);
        if (object && object.isMesh) {
            object.material = props.material;
            object.visible = props.visible;
        }
    });
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
    console.log("De-isolated objects");
}


window.addEventListener('click', (event) => {
    if (!loadedObjectModelRoot) return;
    if (event.target.closest('#modelTreePanel')) return; // Ignore clicks inside model tree panel

    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);

    let newlyClickedTarget = null;
    if (intersects.length > 0) {
        let intersectedObject = intersects[0].object;
        // Find the named group/mesh that is a child of loadedObjectModelRoot
        let current = intersectedObject;
        while (current && current !== scene) {
            if (current.parent === loadedObjectModelRoot && (current.name || current.isMesh)) { // Prioritize direct children with names
                newlyClickedTarget = current;
                break;
            }
            if (current === loadedObjectModelRoot && current.name) { // The root itself is named
                newlyClickedTarget = current;
                break;
            }
            current = current.parent;
        }
         if (!newlyClickedTarget && intersectedObject.name && intersectedObject.parent === loadedObjectModelRoot) {
            newlyClickedTarget = intersectedObject; // If the mesh itself is named and direct child
        }
    }

    removeAllHighlights();
    deIsolateAllObjects();

    if (newlyClickedTarget) {
        if (!selectedObjectOrGroup || selectedObjectOrGroup.uuid !== newlyClickedTarget.uuid) {
            selectedObjectOrGroup = newlyClickedTarget;
            applyHighlight(selectedObjectOrGroup, highlightColorSingle);
            zoomToAndIsolate(selectedObjectOrGroup);
        } else {
            selectedObjectOrGroup = null; // Deselect by clicking same
        }
    } else {
        selectedObjectOrGroup = null; // Clicked empty space
    }
    
    document.querySelectorAll('#modelTreePanel .tree-item.selected').forEach(el => el.classList.remove('selected'));
    if(selectedObjectOrGroup) {
        const treeItemDiv = document.querySelector(`#modelTreePanel li[data-uuid="${selectedObjectOrGroup.uuid}"] .tree-item`);
        if(treeItemDiv) treeItemDiv.classList.add('selected');
    }
    updateInfoPanel();
});

// --- Event Listeners for Panel Controls ---
if (closeModelTreeBtn) {
    closeModelTreeBtn.addEventListener('click', () => {
        if (modelTreePanel) modelTreePanel.style.display = 'none';
    });
}
if (modelTreeSearch) {
    modelTreeSearch.addEventListener('input', (e) => { /* ... (keep basic search logic) ... */ });
}

// --- Info Panel Update ---
function updateInfoPanel() { /* ... (keep your existing updateInfoPanel, ensure it uses selectedObjectOrGroup) ... */ }

// --- Window Resize ---
window.addEventListener('resize', () => { /* ... */ });

// --- Animation Loop ---
function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}
animate();





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


