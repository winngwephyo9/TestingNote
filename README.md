import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';
import { OBJLoader } from './library/controls/OBJLoader.js';
import { MTLLoader } from './library/controls/MTLLoader.js';

// --- Get UI Elements ---
const loaderContainer = document.getElementById('loader-container');
const loaderTextElement = document.getElementById('loader-text');
const modelTreePanel = document.getElementById('modelTreePanel');
const modelTreeList = document.getElementById('modelTreeList');
const modelTreeSearch = document.getElementById('modelTreeSearch');
const closeModelTreeBtn = document.getElementById('closeModelTreeBtn');
const objectInfoPanel = document.getElementById('objectInfo');
const modelSelector = document.getElementById('model-selector');
const viewerContainer = document.getElementById('viewer-container');
const toggleUiButton = document.getElementById('toggle-ui-button');
const resetViewButton = document.getElementById('resetViewButton');


// --- Global variables ---
let parsedWSCenID = "";
let parsedPJNo = "";
let loadedObjectModelRoot = null;
let initialCameraPosition = new THREE.Vector3(10, 10, 10);
let initialCameraLookAt = new THREE.Vector3(0, 0, 0);
let selectedObjectOrGroup = null;
const originalMeshMaterials = new Map();
const originalObjectPropertiesForIsolate = new Map();
let isIsolateModeActive = false;
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const highlightColorSingle = new THREE.Color(0xa0c4ff); // Light Blue (matches CSS)
const elementIdDataMap = new Map(); // Stores all fetched element data


// --- Model Configuration ---
const models = {
    'GF本社移転': {
        obj: '240324_GF本社移転_2022_20250627.obj',
        mtl: '240324_GF本社移転_2022_20250627.mtl',
    },
    'BIKEN15号棟': {
        obj: '240627_BIKEN15号棟_2022_20250630.obj',
        mtl: '240627_BIKEN15号棟_2022_20250630.mtl',
    },
    'パナソニックエナジー西門真地区': {
        obj: '240628_パナソニックエナジー西門真地区_2022__20250630.obj',
        mtl: '240628_パナソニックエナジー西門真地区_2022__20250630.mtl',
    },
};


// --- Scene Setup, Camera, Renderer, Lighting, Controls ---
const scene = new THREE.Scene();

const backgroundGeometry = new THREE.PlaneGeometry(2, 2, 1, 1);
const backgroundMaterial = new THREE.ShaderMaterial({
    vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = vec4(position.xy, 1.0, 1.0); }`,
    fragmentShader: `uniform vec3 topColor; uniform vec3 bottomColor; varying vec2 vUv; void main() { gl_FragColor = vec4(mix(bottomColor, topColor, vUv.y), 1.0); }`,
    uniforms: {
        topColor: { value: new THREE.Color(0xd8e3ee) },
        bottomColor: { value: new THREE.Color(0xf0f0f0) }
    },
    depthWrite: false
});
const backgroundMesh = new THREE.Mesh(backgroundGeometry, backgroundMaterial);
backgroundMesh.renderOrder = -1;
scene.add(backgroundMesh);

const camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight, 0.1, 20000);
camera.position.copy(initialCameraPosition);
camera.lookAt(initialCameraLookAt);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
viewerContainer.appendChild(renderer.domElement);

let mouseDownPosition = new THREE.Vector2();

const ambientLight = new THREE.AmbientLight(0x606060, 2);
scene.add(ambientLight);
const directionalLight = new THREE.DirectionalLight(0xFFFFFF, 2.5);
directionalLight.position.set(50, 100, 75);
directionalLight.castShadow = true;
directionalLight.shadow.mapSize.width = 2048;
directionalLight.shadow.mapSize.height = 2048;
scene.add(directionalLight);
const hemiLight = new THREE.HemisphereLight(0xffffff, 0x8d8d8d, 1.5);
hemiLight.position.set(0, 50, 0);
scene.add(hemiLight);

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;

var CSRF_TOKEN = $('meta[name="csrf-token"]').attr('content');

/**
 * @function $(document).ready
 * @description Main entry point of the application. Fires when the DOM is fully loaded.
 */
$(document).ready(function () {
    var login_user_id = $("#hidLoginID").val();
    if (login_user_id) {
        var img_src = "/DL_DWH.png";
        var url = "DLDWH/objviewer";
        var content_name = "OBJビューア";
        recordAccessHistory(login_user_id, img_src, url, content_name);
    }
    onWindowResize();
    loadModel('GF本社移転');
    animate();
    if (resetViewButton) {
        resetViewButton.addEventListener('click', handleResetView);
    }
});


/**
 * @function loadModel
 * @description The core function to load, process, and display a 3D model.
 * @param {string} modelKey - The key from the `models` object corresponding to the model to load.
 */
async function loadModel(modelKey) {
    if (loadedObjectModelRoot) {
        scene.remove(loadedObjectModelRoot);
    }
    loadedObjectModelRoot = null;
    selectedObjectOrGroup = null;
    originalMeshMaterials.clear();
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
    elementIdDataMap.clear();
    modelTreeList.innerHTML = '';
    if (modelTreePanel) modelTreePanel.style.display = 'none';

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';

        const modelFiles = models[modelKey];
        if (!modelFiles) throw new Error(`Model key "${modelKey}" not found.`);

        const assetPath = '/ccc/public/objFiles/';
        const objFileName = modelFiles.obj;
        const mtlFileName = modelFiles.mtl;
        const fullObjPath = assetPath + objFileName;

        await parseObjHeader(fullObjPath);

        if (loaderTextElement) loaderTextElement.textContent = `Loading Materials...`;
        const mtlLoader = new MTLLoader().setPath(assetPath);
        const materialsCreator = await mtlLoader.loadAsync(mtlFileName);
        materialsCreator.preload();

        const objLoader = new OBJLoader().setMaterials(materialsCreator);
        const object = await objLoader.loadAsync(fullObjPath, (xhr) => {
            if (loaderTextElement) {
                const percent = Math.round(xhr.loaded / xhr.total * 100);
                loaderTextElement.textContent = isFinite(percent) && percent < 100 ? `Loading 3D Geometry: ${percent}%` : `Processing Geometry...`;
            }
        });

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
        const maxDim = Math.max(scaledBox.getSize(new THREE.Vector3()).x, scaledBox.getSize(new THREE.Vector3()).y, scaledBox.getSize(new THREE.Vector3()).z);
        const desiredMaxDimension = 150;
        if (maxDim > 0) {
            const scale = desiredMaxDimension / maxDim;
            object.scale.set(scale, scale, scale);
        }
        object.rotation.x = -Math.PI / 2;

        const allIds = [];
        loadedObjectModelRoot.traverse(child => {
            if (child.name && child.parent === loadedObjectModelRoot) {
                const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                if (splitIndex > 0) allIds.push(child.name.substring(splitIndex + 1));
            }
        });
        await fetchAllCategoryData(parsedWSCenID, [...new Set(allIds)]);

        buildAndPopulateCategorizedTree();

        scene.add(object);
        frameObject(object);

    } catch (error) {
        console.error('Failed to initialize the viewer:', error);
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model: ${modelKey}.`;
    } finally {
        if (loaderContainer) loaderContainer.style.display = 'none';
    }
}

/**
 * @function parseObjHeader
 * @description Fetches an OBJ file and reads the first line to parse metadata.
 * @param {string} filePath - The full path to the .obj file.
 */
async function parseObjHeader(filePath) {
    try {
        const response = await fetch(filePath);
        if (!response.ok) return;
        const text = await response.text();
        const firstLine = text.split(/\r?\n/)[0].trim();
        if (firstLine.startsWith("# ")) {
            const content = firstLine.substring(2).trim();
            const pattern1Match = content.match(/^([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})_([a-zA-Z0-9]+)$/);
            if (pattern1Match) {
                parsedWSCenID = pattern1Match[1];
                parsedPJNo = pattern1Match[2];
            }
        }
    } catch (error) {
        console.error("Error fetching or parsing OBJ header:", error);
    }
}

/**
 * @function fetchAllCategoryData
 * @description Fetches category and family data from the server for a given set of element IDs.
 * @param {string} wscenId - The Worksharing Central ID to use in the API call.
 * @param {string[]} allElementIds - An array of all element IDs to fetch data for.
 */
async function fetchAllCategoryData(wscenId, allElementIds) {
    if (allElementIds.length === 0) return Promise.resolve();
    const batchSize = 900;
    for (let i = 0; i < allElementIds.length; i += batchSize) {
        const batch = allElementIds.slice(i, i + batchSize);
        if (loaderTextElement) loaderTextElement.textContent = `Fetching Categories... (${i + batch.length}/${allElementIds.length})`;
        await $.ajax({
            type: "post",
            url: url_prefix + "/DLDWH/getDatas",
            data: { _token: CSRF_TOKEN, WSCenID: wscenId, ElementIds: batch },
            success: function (data) {
                for (const elementId in data) {
                    elementIdDataMap.set(elementId, data[elementId]);
                }
            },
            error: function (err) {
                console.error(`Error fetching batch starting at index ${i}:`, err);
            }
        });
    }
}

/**
 * @function buildAndPopulateCategorizedTree
 * @description Organizes the loaded 3D objects by category and dynamically builds the tree view UI.
 */
function buildAndPopulateCategorizedTree() {
    if (!loadedObjectModelRoot || !modelTreeList) return;
    if (loaderTextElement) loaderTextElement.textContent = "Building model tree...";
    const categorizedObjects = {};
    loadedObjectModelRoot.traverse(child => {
        if ((child.isGroup || child.isMesh) && child.name && child.parent === loadedObjectModelRoot) {
            let rawName = child.name;
            let displayId = null;
            const splitIndex = Math.max(rawName.lastIndexOf('_'), rawName.lastIndexOf('＿'));
            if (splitIndex > 0) displayId = rawName.substring(splitIndex + 1);
            let category = "カテゴリー無し";
            if (displayId && elementIdDataMap.has(displayId)) {
                category = elementIdDataMap.get(displayId)['カテゴリー名'] || "カテゴリー無し";
            } else if (!displayId) {
                category = "名称未分類";
            }
            if (!categorizedObjects[category]) categorizedObjects[category] = [];
            categorizedObjects[category].push(child);
        }
    });
    modelTreeList.innerHTML = '';
    Object.keys(categorizedObjects).sort().forEach(categoryName => {
        createCategoryNode(categoryName, categorizedObjects[categoryName]);
    });
    if (modelTreePanel) modelTreePanel.style.display = 'block';
}

/**
 * @function frameObject
 * @description Calculates the appropriate camera position to fit a given 3D object in the view.
 * @param {THREE.Object3D} objectToFrame - The object to be framed.
 */
function frameObject(objectToFrame) {
    const box = new THREE.Box3().setFromObject(objectToFrame);
    if (box.isEmpty()) {
        camera.position.set(50, 50, 50);
        controls.target.set(0, 0, 0);
        controls.update();
        return;
    }
    const center = box.getCenter(new THREE.Vector3());
    const sphere = box.getBoundingSphere(new THREE.Sphere());
    const radius = sphere.radius;
    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    const distance = (radius / Math.sin(fovInRadians / 2)) * 1.3;
    const cameraDirection = new THREE.Vector3(1, 0.6, 1).normalize();
    initialCameraPosition.copy(center).addScaledVector(cameraDirection, distance);
    camera.position.copy(initialCameraPosition);
    controls.target.copy(center);
    controls.update();
}


/**
 * @function createCategoryNode
 * @description Creates a single top-level list item (a category) in the model tree UI.
 * @param {string} categoryName - The name of the category.
 * @param {THREE.Object3D[]} objectsInCategory - An array of objects in this category.
 */
function createCategoryNode(categoryName, objectsInCategory) {
    const categoryLi = document.createElement('li');
    const itemContent = document.createElement('div');
    itemContent.className = 'tree-item';
    itemContent.style.fontWeight = 'bold';
    const toggler = document.createElement('span');
    toggler.className = 'toggler';
    toggler.textContent = '▼';
    itemContent.appendChild(toggler);
    const nameSpan = document.createElement('span');
    nameSpan.className = 'group-name';
    nameSpan.textContent = `${categoryName} (${objectsInCategory.length})`;
    itemContent.appendChild(nameSpan);
    const subList = document.createElement('ul');
    toggler.addEventListener('click', (e) => {
        e.stopPropagation();
        const isCollapsed = subList.style.display === 'none';
        subList.style.display = isCollapsed ? 'block' : 'none';
        toggler.textContent = isCollapsed ? '▼' : '▶';
    });
    categoryLi.appendChild(itemContent);
    categoryLi.appendChild(subList);
    modelTreeList.appendChild(categoryLi);
    objectsInCategory.forEach(object => createObjectNode(object, subList, 1));
}

/**
 * @function createObjectNode
 * @description Creates a single child list item (an object) within a category in the model tree.
 * @param {THREE.Object3D} object - The 3D object this tree node represents.
 * @param {HTMLElement} parentULElement - The `<ul>` element of the parent category.
 * @param {number} depth - The nesting level of the item.
 */
function createObjectNode(object, parentULElement, depth) {
    const listItem = document.createElement('li');
    listItem.dataset.uuid = object.uuid;
    const itemContent = document.createElement('div');
    itemContent.className = 'tree-item';
    itemContent.style.paddingLeft = `${depth * 15 + 10}px`;
    const toggler = document.createElement('span');
    toggler.className = 'toggler empty-toggler';
    toggler.innerHTML = ' ';
    itemContent.appendChild(toggler);
    const nameSpan = document.createElement('span');
    nameSpan.className = 'group-name';
    nameSpan.textContent = object.name;
    nameSpan.title = object.name;
    itemContent.appendChild(nameSpan);
    const visibilityToggle = document.createElement('span');
    visibilityToggle.className = 'visibility-toggle visible-icon';
    visibilityToggle.title = 'Hide';
    itemContent.appendChild(visibilityToggle);
    listItem.appendChild(itemContent);
    parentULElement.appendChild(listItem);
    itemContent.addEventListener('click', () => handleSelection(object));
    visibilityToggle.addEventListener('click', (e) => {
        e.stopPropagation();
        object.visible = !object.visible;
        visibilityToggle.classList.toggle('visible-icon', object.visible);
        visibilityToggle.classList.toggle('hidden-icon', !object.visible);
        visibilityToggle.title = object.visible ? 'Hide' : 'Show';
        if (!object.visible && selectedObjectOrGroup && selectedObjectOrGroup.uuid === object.uuid) {
            handleSelection(null);
        }
    });
}


/**
 * @function handleSelection
 * @description Manages the entire selection process.
 * @param {THREE.Object3D | null} target - The object to be selected, or null to deselect all.
 */
function handleSelection(target) {
    removeAllHighlights();
    deIsolateAllObjects();
    let newSelection = null;
    if (target && (!selectedObjectOrGroup || selectedObjectOrGroup.uuid !== target.uuid)) {
        newSelection = target;
    }
    selectedObjectOrGroup = newSelection;
    if (selectedObjectOrGroup) {
        applyHighlight(selectedObjectOrGroup, highlightColorSingle);
        zoomToAndIsolate(selectedObjectOrGroup);
    }
    document.querySelectorAll('#modelTreePanel .tree-item.selected').forEach(el => el.classList.remove('selected'));
    if (selectedObjectOrGroup) {
        const treeItemDiv = document.querySelector(`#modelTreePanel li[data-uuid="${selectedObjectOrGroup.uuid}"] .tree-item`);
        if (treeItemDiv) treeItemDiv.classList.add('selected');
    }
    updateInfoPanel();
}

/**
 * @function applyHighlight
 * @description Traverses a 3D object and applies a highlight color to all its child meshes.
 * @param {THREE.Object3D} target - The object to highlight.
 * @param {THREE.Color} color - The color to use for the highlight.
 */
const applyHighlight = (target, color) => {
    if (!target) return;
    target.traverse(child => {
        if (child.isMesh) {
            if (!originalMeshMaterials.has(child.uuid)) {
                originalMeshMaterials.set(child.uuid, child.material);
            }
            if (Array.isArray(child.material)) {
                child.material = child.material.map(originalMat => {
                    const highlightMat = originalMat.clone();
                    if (highlightMat.color) highlightMat.color.set(color);
                    return highlightMat;
                });
            } else {
                const highlightMat = child.material.clone();
                if (highlightMat.color) highlightMat.color.set(color);
                child.material = highlightMat;
            }
        }
    });
};

/**
 * @function removeAllHighlights
 * @description Restores the original materials to all meshes that were previously highlighted.
 */
const removeAllHighlights = () => {
    originalMeshMaterials.forEach((originalMaterialOrArray, meshUuid) => {
        const mesh = scene.getObjectByProperty('uuid', meshUuid);
        if (mesh && mesh.isMesh) mesh.material = originalMaterialOrArray;
    });
    originalMeshMaterials.clear();
};

/**
 * @function zoomToAndIsolate
 * @description Zooms the camera to a selected object and makes all other objects semi-transparent.
 * @param {THREE.Object3D} targetObject - The object to isolate and zoom to.
 */
function zoomToAndIsolate(targetObject) {
    if (!targetObject) return;
    deIsolateAllObjects();
    isIsolateModeActive = true;
    const box = new THREE.Box3().setFromObject(targetObject);
    if (box.isEmpty()) { isIsolateModeActive = false; return; }
    const center = box.getCenter(new THREE.Vector3());
    const sphere = box.getBoundingSphere(new THREE.Sphere());
    const radius = sphere.radius;
    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    let distance = (radius / Math.sin(fovInRadians / 2)) * 1.5;
    const offsetDirection = camera.position.clone().sub(controls.target).normalize();
    if (offsetDirection.lengthSq() === 0) offsetDirection.set(0.5, 0.5, 1).normalize();
    camera.position.copy(center).addScaledVector(offsetDirection, distance);
    controls.target.copy(center);
    controls.update();

    loadedObjectModelRoot.traverse((object) => {
        if (object.isMesh) {
            let isPartOfSelectedTarget = false;
            let temp = object;
            while (temp) {
                if (temp === targetObject) { isPartOfSelectedTarget = true; break; }
                temp = temp.parent;
            }
            if (!isPartOfSelectedTarget) {
                if (!originalObjectPropertiesForIsolate.has(object.uuid)) {
                    originalObjectPropertiesForIsolate.set(object.uuid, { material: object.material, visible: object.visible });
                }
                if (object.visible) {
                    const makeTransparent = mat => {
                        const newMat = mat.clone();
                        newMat.transparent = true;
                        newMat.opacity = 0.1;
                        return newMat;
                    };
                    object.material = Array.isArray(object.material) ? object.material.map(makeTransparent) : makeTransparent(object.material);
                }
            }
        }
    });
}


/**
 * @function deIsolateAllObjects
 * @description Restores all objects to their original materials and visibility.
 */
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
}

/**
 * @function updateInfoPanel
 * @description Updates the content of the top-left information panel to display
 * details and calculated volume about the currently selected object.
 */
function updateInfoPanel() {
    let headerInfo = `WSCenID: ${parsedWSCenID || "N/A"}\nPJNo: ${parsedPJNo || "N/A"}\n----\n`;
    if (!parsedWSCenID && !parsedPJNo) headerInfo = `WSCenID: \nPJNo: \n----\n`;

    if (selectedObjectOrGroup) {
        let rawName = selectedObjectOrGroup.name || "Unnamed";
        let displayName, displayId = "N/A";
        const splitIndex = Math.max(rawName.lastIndexOf('_'), rawName.lastIndexOf('＿'));
        if (splitIndex > 0) {
            displayName = rawName.substring(0, splitIndex);
            displayId = rawName.substring(splitIndex + 1);
        } else {
            displayName = rawName;
        }
        let selectionInfo = `名前: ${displayName}\nID: ${displayId}\n`;
        if (elementIdDataMap.has(displayId)) {
            const data = elementIdDataMap.get(displayId);
            selectionInfo += `\nカテゴリー名: ${data['カテゴリー名'] || "N/A"}\nファミリ名: ${data['ファミリ名'] || "N/A"}\nタイプ_ID: ${data['タイプ_ID'] || "N/A"}`;
        } else {
            selectionInfo += '\nカテゴリー名: "" \nファミリ名: ""';
        }

        // --- NEW: Calculate and display volume ---
        const volume = getVolumeOfSelectedObject();
        // Only display volume if it's a meaningful (positive) number.
        // This avoids showing "Volume: 0.0000" for flat objects.
        if (volume > 0.0001) { 
            // The units (e.g., mm³, m³) depend on the source CAD model's units.
            // We display a generic "units³" to reflect this.
            selectionInfo += `\n\n体積 (Volume): ${volume.toFixed(4)} units³`;
        }

        objectInfoPanel.textContent = headerInfo + selectionInfo;
    } else {
        objectInfoPanel.textContent = headerInfo + 'None Selected';
    }
}

/**
 * @event mousedown
 * @description Listens for the mouse button being pressed down on the viewer container.
 */
viewerContainer.addEventListener('mousedown', (event) => {
    mouseDownPosition.set(event.clientX, event.clientY);
}, false);

/**
 * @event mouseup
 * @description Listens for the mouse button being released and triggers a selection if it was a click.
 */
viewerContainer.addEventListener('mouseup', (event) => {
    const mouseUpPosition = new THREE.Vector2(event.clientX, event.clientY);
    const DRAG_THRESHOLD = 5;
    if (mouseDownPosition.distanceTo(mouseUpPosition) > DRAG_THRESHOLD) return;

    if (!loadedObjectModelRoot) return;
    const rect = viewerContainer.getBoundingClientRect();
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);
    let newlyClickedTarget = null;
    if (intersects.length > 0) {
        let current = intersects[0].object;
        while (current && current.parent !== loadedObjectModelRoot && current.parent !== scene) {
            current = current.parent;
        }
        if (current) newlyClickedTarget = current;
    }
    handleSelection(newlyClickedTarget);
}, false);


if (closeModelTreeBtn) {
    closeModelTreeBtn.addEventListener('click', () => { if (modelTreePanel) modelTreePanel.style.display = 'none'; });
}

if (modelTreeSearch) {
    modelTreeSearch.addEventListener('input', (e) => {
        const searchTerm = e.target.value.toLowerCase().trim();
    });
}

if (toggleUiButton) {
    toggleUiButton.addEventListener('click', () => {
        const isVisible = modelTreePanel.style.display !== 'none';
        if (modelTreePanel) modelTreePanel.style.display = isVisible ? 'none' : 'block';
        toggleUiButton.textContent = isVisible ? '📊' : '❌';
        toggleUiButton.title = isVisible ? "Show UI Panels" : "Hide UI Panels";
    });
}

/**
 * @function onWindowResize
 * @description Handles the browser window being resized. Fixes blurriness on high-DPI screens.
 */
function onWindowResize() {
    const { clientWidth, clientHeight } = viewerContainer;
    camera.aspect = clientWidth / clientHeight;
    camera.updateProjectionMatrix();
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setSize(clientWidth, clientHeight);
}
window.addEventListener('resize', onWindowResize);


if (modelSelector) {
    modelSelector.addEventListener('change', (event) => {
        loadModel(event.target.value);
    });
}

/**
 * @function animate
 * @description The main rendering loop.
 */
function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.autoClearColor = false;
    renderer.render(scene, camera);
}

/**
 * @function handleResetView
 * @description Resets the viewer to its initial state on a single click.
 */
function handleResetView() {
    deIsolateAllObjects();
    removeAllHighlights();
    selectedObjectOrGroup = null;
    document.querySelectorAll('#modelTreePanel .tree-item.selected').forEach(el => el.classList.remove('selected'));
    updateInfoPanel();
    if (loadedObjectModelRoot) {
        frameObject(loadedObjectModelRoot);
    }
}

// --- NEW HELPER FUNCTIONS FOR VOLUME CALCULATION ---

/**
 * @function getVolumeOfSelectedObject
 * @description Calculates the total volume of the currently selected group or object.
 * It traverses the selection and sums the volumes of all child meshes.
 * @returns {number} The total calculated volume. Returns 0 if no object is selected.
 */
function getVolumeOfSelectedObject() {
    if (!selectedObjectOrGroup) {
        return 0;
    }
    let totalVolume = 0;
    // Traverse all children of the selected object (which could be a group)
    selectedObjectOrGroup.traverse(child => {
        // We can only calculate volume for a mesh with geometry
        if (child.isMesh && child.geometry) {
            totalVolume += calculateMeshVolume(child);
        }
    });
    return totalVolume;
}

/**
 * @function calculateMeshVolume
 * @description Calculates the volume of a single mesh.
 * IMPORTANT: This assumes the mesh is a closed, watertight manifold.
 * The result for open meshes (like a plane) is not meaningful.
 * @param {THREE.Mesh} mesh - The mesh to calculate the volume of.
 * @returns {number} The calculated volume of the mesh in world units.
 */
function calculateMeshVolume(mesh) {
    const geometry = mesh.geometry;
    if (!geometry.isBufferGeometry) {
        console.warn('Volume calculation only supports BufferGeometry.');
        return 0;
    }

    let totalVolume = 0;
    const position = geometry.attributes.position;
    const a = new THREE.Vector3();
    const b = new THREE.Vector3();
    const c = new THREE.Vector3();

    // This function processes one triangle
    const processTriangle = (vA_idx, vB_idx, vC_idx) => {
        // Get vertices from the geometry's position attribute
        a.fromBufferAttribute(position, vA_idx);
        b.fromBufferAttribute(position, vB_idx);
        c.fromBufferAttribute(position, vC_idx);

        // Apply the mesh's world matrix to get the true position of vertices in the scene
        a.applyMatrix4(mesh.matrixWorld);
        b.applyMatrix4(mesh.matrixWorld);
        c.applyMatrix4(mesh.matrixWorld);

        // Calculate the signed volume of the tetrahedron formed by the triangle and the origin
        // The formula is (1/6) * dot(a, cross(b, c))
        totalVolume += a.dot(b.clone().cross(c));
    };

    if (geometry.index) {
        // For indexed geometries
        const index = geometry.index;
        for (let i = 0; i < index.count; i += 3) {
            processTriangle(index.getX(i), index.getX(i + 1), index.getX(i + 2));
        }
    } else {
        // For non-indexed geometries
        for (let i = 0; i < position.count; i += 3) {
            processTriangle(i, i + 1, i + 2);
        }
    }

    // The formula calculates 6x the volume, so we divide by 6.
    // We use Math.abs because volume cannot be negative; the sign depends on triangle winding order.
    return Math.abs(totalVolume / 6.0);
}
