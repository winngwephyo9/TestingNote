![image](https://github.com/user-attachments/assets/30131c4b-a61a-4be2-b735-b5ffb5d0687e)

            
            const zoomOutFactor = 1.1;
                        const desiredMaxDimension = 100;

Scaled Object Size (finalWorldSize): 91.35109548282391 10.543176900189362 100
objViewerStandard.js:253 Calculated Camera Distance (before zoomOutFactor): 41.791103090298606
objViewerStandard.js:254 Camera Distance (after zoomOutFactor & Math.max): 41.791103090298606


const finalWorldBox = new THREE.Box3().setFromObject(object);
const finalWorldCenter = finalWorldBox.getCenter(new THREE.Vector3());
const finalWorldSize = finalWorldBox.getSize(new THREE.Vector3());
console.log("Scaled Object Size (finalWorldSize):", finalWorldSize.x, finalWorldSize.y, finalWorldSize.z);

// ... after cameraDistance calculation ...
console.log("Calculated Camera Distance (before zoomOutFactor):", effectiveSizeDimension / (2 * Math.tan(fovInRadians / 2)));
console.log("Camera Distance (after zoomOutFactor & Math.max):", cameraDistance);




// --- Scene Setup ---
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xcccccc);

// --- Camera Setup ---
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 20000);
let initialCameraPosition = new THREE.Vector3(10, 10, 10); // Store for reset
let initialCameraLookAt = new THREE.Vector3(0, 0, 0);   // Store for reset
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
// ... (shadow camera properties)
scene.add(directionalLight);
const hemiLight = new THREE.HemisphereLight( 0xffffff, 0x8d8d8d, 1.5 );
hemiLight.position.set( 0, 50, 0 );
scene.add( hemiLight );

// --- Controls Setup ---
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;
controls.screenSpacePanning = false;
controls.minDistance = 1;
controls.maxDistance = 10000;



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
let selectedObjectOrGroup = null;
const originalMeshMaterials = new Map();
const originalObjectPropertiesForIsolate = new Map();
let isIsolateModeActive = false;

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const highlightColorSingle = new THREE.Color(0xffff00);

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
            // console.log("First line of OBJ:", firstLine);
            if (firstLine.startsWith("# ")) {
                const content = firstLine.substring(2).trim();
                const pattern1Match = content.match(/^([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})_([a-zA-Z0-9]+)$/);
                if (pattern1Match) {
                    parsedWSCenID = pattern1Match[1];
                    parsedPJNo = pattern1Match[2];
                    // console.log(`Pattern 1 Matched: WSCenID=${parsedWSCenID}, PJNo=${parsedPJNo}`);
                    return;
                }
                if (content.includes("ワークシェアリングされてない") && (content.includes("_PJNo無し") || content.includes("＿PJNo無し"))) {
                    parsedWSCenID = "";
                    parsedPJNo = "";
                    // console.log(`Pattern 2 Matched: WSCenID=${parsedWSCenID}, PJNo=${parsedPJNo}`);
                    return;
                }
            }
        }
        // console.log("First line did not match expected patterns for WSCenID/PJNo.");
    } catch (error) {
        console.error("Error fetching or parsing OBJ header:", error);
    }
}

// --- Function to Populate Model Tree ---
function buildTreeRecursive(object, parentULElement, depth = 0) {
    if (!object.name && object.isMesh && object.children.length === 0 && object !== loadedObjectModelRoot) return;
    if (!object.name && object.isGroup && object.children.length === 0) return;

    const listItem = document.createElement('li');
    listItem.dataset.uuid = object.uuid;
    const itemContent = document.createElement('div');
    itemContent.className = 'tree-item';
    itemContent.style.paddingLeft = `${depth * 15 + 10}px`;

    const hasVisibleChildrenForToggler = object.children.some(child => (child.isGroup && child.name) || (child.isMesh && child.name) || (child.isGroup && !child.name && child.children.length > 0));

    const toggler = document.createElement('span');
    toggler.className = 'toggler';
    if (object.isGroup && hasVisibleChildrenForToggler) {
        toggler.textContent = '▶';
    } else {
        toggler.classList.add('empty-toggler');
        toggler.innerHTML = ' ';
    }
    itemContent.appendChild(toggler);

    const nameSpan = document.createElement('span');
    nameSpan.className = 'group-name';
    nameSpan.textContent = object.name || (object.isMesh ? "Unnamed Mesh" : "Unnamed Group");
    nameSpan.title = object.name || (object.isMesh ? "Unnamed Mesh" : "Unnamed Group");
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
        removeAllHighlights();
        deIsolateAllObjects();
        if (selectedObjectOrGroup === object) {
            selectedObjectOrGroup = null;
        } else {
            selectedObjectOrGroup = object;
            applyHighlight(selectedObjectOrGroup, highlightColorSingle);
            zoomToAndIsolate(selectedObjectOrGroup);
        }
        document.querySelectorAll('#modelTreePanel .tree-item.selected').forEach(el => el.classList.remove('selected'));
        if (selectedObjectOrGroup) itemContent.classList.add('selected');
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
        object.children.forEach(child => buildTreeRecursive(child, subList, depth + 1));
    }
}

function populateModelTree() {
    if (!loadedObjectModelRoot || !modelTreeList) return;
    modelTreeList.innerHTML = '';
    if (loadedObjectModelRoot.name) {
        buildTreeRecursive(loadedObjectModelRoot, modelTreeList, 0);
    } else {
        loadedObjectModelRoot.children.forEach(child => buildTreeRecursive(child, modelTreeList, 0));
    }
    if (modelTreePanel) modelTreePanel.style.display = 'block';
}

// --- OBJ Loading ---
const objLoader = new OBJLoader();
const objPath = './objFiles/standard_testing.obj'; // YOUR OBJ FILE PATH

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
            // console.log("OBJ loaded by OBJLoader:", loadedObjectModelRoot);

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
    deIsolateAllObjects();
    isIsolateModeActive = true;

    const box = new THREE.Box3().setFromObject(targetObject);
    if (box.isEmpty()) {
        console.warn("Cannot zoom to object with empty bounding box:", targetObject.name);
        isIsolateModeActive = false;
        return;
    }
    const center = box.getCenter(new THREE.Vector3());
    const size = box.getSize(new THREE.Vector3());
    const maxDim = Math.max(size.x, size.y, size.z);

    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    const aspect = camera.aspect;
    const effectiveSize = Math.max(size.y, size.x / aspect);
    let distance = effectiveSize / (2 * Math.tan(fovInRadians / 2));
    distance = Math.max(distance, maxDim * 1.2);
    distance *= 1.5;

    const offsetDirection = camera.position.clone().sub(controls.target).normalize();
    if (offsetDirection.lengthSq() === 0) offsetDirection.set(0.5, 0.5, 1).normalize();

    camera.position.copy(center).addScaledVector(offsetDirection, distance);
    controls.target.copy(center);
    controls.update();

    loadedObjectModelRoot.traverse((object) => {
        if (object.isMesh) {
            let isPartOfSelectedTarget = false;
            let temp = object;
            while (temp && temp !== loadedObjectModelRoot.parent) {
                if (temp === targetObject) {
                    isPartOfSelectedTarget = true;
                    break;
                }
                temp = temp.parent;
            }
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
            } else {
                if (originalObjectPropertiesForIsolate.has(object.uuid)) {
                    const props = originalObjectPropertiesForIsolate.get(object.uuid);
                    object.material = props.material;
                    object.visible = props.visible;
                    originalObjectPropertiesForIsolate.delete(object.uuid);
                } else {
                    if (Array.isArray(object.material)) {
                        object.material.forEach(mat => { mat.transparent = false; mat.opacity = 1.0; });
                    } else if (object.material) { // Check if material exists
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
    // console.log("De-isolated objects");
}

window.addEventListener('click', (event) => {
    if (!loadedObjectModelRoot) return;
    if (event.target.closest('#modelTreePanel')) return;

    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);

    let newlyClickedTarget = null;
    if (intersects.length > 0) {
        let intersectedObject = intersects[0].object;
        let current = intersectedObject;
        while (current && current !== scene) {
            if (current.parent === loadedObjectModelRoot && (current.name || current.isMesh)) {
                newlyClickedTarget = current;
                break;
            }
            if (current === loadedObjectModelRoot && current.name) {
                newlyClickedTarget = current;
                break;
            }
            current = current.parent;
        }
         if (!newlyClickedTarget && intersectedObject.name && intersectedObject.parent === loadedObjectModelRoot) {
            newlyClickedTarget = intersectedObject;
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
            selectedObjectOrGroup = null;
        }
    } else {
        selectedObjectOrGroup = null;
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
    modelTreeSearch.addEventListener('input', (e) => {
        const searchTerm = e.target.value.toLowerCase().trim();
        const allListItems = modelTreeList.querySelectorAll('li'); // Get all <li> elements

        allListItems.forEach(li => {
            const itemContentDiv = li.querySelector('.tree-item');
            if (!itemContentDiv) return;

            const nameSpan = itemContentDiv.querySelector('.group-name');
            if (!nameSpan) return;

            const itemName = nameSpan.textContent.toLowerCase();
            const isMatch = itemName.includes(searchTerm);
            
            if (isMatch) {
                li.style.display = ''; // Show the matching item's li
                // Make its parents visible and expanded
                let parentLi = li.parentElement.closest('li');
                while (parentLi) {
                    parentLi.style.display = '';
                    const parentSubList = parentLi.querySelector('ul');
                    if (parentSubList) parentSubList.style.display = 'block';
                    const parentToggler = parentLi.querySelector('.toggler:not(.empty-toggler)');
                    if (parentToggler) parentToggler.textContent = '▼';
                    parentLi = parentLi.parentElement.closest('li');
                }
            } else {
                li.style.display = 'none'; // Hide non-matching item's li
            }
        });

        // If search is empty, reset all to potentially visible (respecting their own UL's display state)
        if (searchTerm === "") {
            allListItems.forEach(li => {
                // If a UL inside this LI was set to 'none' by a toggler, keep it that way.
                // Otherwise, make the LI visible.
                const subUl = li.querySelector('ul');
                if (!subUl || subUl.style.display !== 'none') {
                    li.style.display = '';
                } else {
                     li.style.display = ''; // Still show the LI, but its children UL remains hidden
                }
            });
        }
    });
}


// --- Info Panel Update ---
function updateInfoPanel() {
    if (!objectInfoPanel) return;
    let headerInfo = "";
    if (parsedWSCenID || parsedPJNo) {
        headerInfo = `WSCenID: ${parsedWSCenID || "N/A"}\nPJNo: ${parsedPJNo || "N/A"}\n----\n`;
    } else if (parsedWSCenID === "" && parsedPJNo === "") {
        headerInfo = `WSCenID: \nPJNo: \n----\n`;
    }

    if (selectedObjectOrGroup) {
        const pos = selectedObjectOrGroup.getWorldPosition(new THREE.Vector3());
        let rawName = selectedObjectOrGroup.name || "Unnamed";
        let displayName = rawName;
        let displayId = "N/A";
        const lastUnderscoreHalf = rawName.lastIndexOf('_');
        const lastUnderscoreFull = rawName.lastIndexOf('＿');
        let splitIndex = -1;
        if (lastUnderscoreHalf !== -1 && lastUnderscoreFull !== -1) splitIndex = Math.max(lastUnderscoreHalf, lastUnderscoreFull);
        else if (lastUnderscoreHalf !== -1) splitIndex = lastUnderscoreHalf;
        else if (lastUnderscoreFull !== -1) splitIndex = lastUnderscoreFull;

        if (splitIndex > 0 && splitIndex < rawName.length - 1) {
            displayName = rawName.substring(0, splitIndex);
            displayId = rawName.substring(splitIndex + 1);
        } else displayName = rawName;
        
        const selectionInfo = `名前: ${displayName}\n  ID: ${displayId}\n  x: ${pos.x.toFixed(2)}, y: ${pos.y.toFixed(2)}, z: ${pos.z.toFixed(2)}`;
        objectInfoPanel.textContent = headerInfo + selectionInfo;
    } else {
        objectInfoPanel.textContent = headerInfo + 'None Selected';
    }
}

// --- Window Resize ---
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// --- Animation Loop ---
function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}
animate();
Use code with caution.
JavaScript
Key improvements in this version's search logic:
allListItems = modelTreeList.querySelectorAll('li');: Gets all <li> elements at once.
itemContentDiv = li.querySelector('.tree-item');: We operate on the inner div for name checking but hide/show the parent <li>.
Showing Parents: When a match is found (isMatch), it iterates upwards (let parentLi = li.parentElement.closest('li');) to ensure all parent <li> elements are displayed and their <ul> children are expanded (by setting display = 'block' and changing the toggler icon).
Hiding Non-Matches: li.style.display = 'none'; for non-matching items.
Clearing Search: If searchTerm === "", it attempts to make all <li> items visible again.
Caveat: Perfectly restoring the exact collapsed/expanded state of every node before the search started is complex with this simple DOM manipulation. A more robust tree would use a data structure to manage node states and re-render from that data. For now, clearing the search makes all list items visible, and if a sub-list was explicitly collapsed by the user clicking its toggler, it should remain collapsed due to its ul.style.display still being 'none'.
This version provides a more usable search functionality for the model tree.
