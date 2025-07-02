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
let selectedObjectOrGroup = null;
const originalMeshMaterials = new Map();
const originalObjectPropertiesForIsolate = new Map();
let isIsolateModeActive = false;
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const highlightColorSingle = new THREE.Color(0xffff00);

// --- Scene, Camera, Renderer, Lighting, Controls Setup ---
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xcccccc);
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 20000);
camera.position.copy(initialCameraPosition);
camera.lookAt(initialCameraLookAt);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);
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

// --- Helper Functions (Header Parsing, Highlighting, Isolation) ---
async function parseObjHeader(filePath) { /* ... keep this function from previous version ... */ }
const applyHighlight = (target, color) => { /* ... keep this function ... */ };
const removeAllHighlights = () => { /* ... keep this function ... */ };
function zoomToAndIsolate(targetObject) { /* ... keep this function ... */ }
function deIsolateAllObjects() { /* ... keep this function ... */ }

// --- AJAX Function Adapted to Return a Promise ---
function getCategoryInfo(wscenId, elementId) {
    return new Promise((resolve, reject) => {
        // --- YOUR ACTUAL AJAX CALL ---
        // Make sure url_prefix and CSRF_TOKEN are defined globally or passed in.
        $.ajax({
            type: "post",
            url: url_prefix + "/DLDWH/getData",
            data: { _token: CSRF_TOKEN, WSCenID: wscenId, ElementId: elementId },
            success: function (data) {
                const result = (data && data.length > 0) ? data[0] : null;
                resolve(result); // Resolve promise with the result
            },
            error: function (err) {
                console.error(`Error fetching data for ElementId ${elementId}:`, err);
                // Resolve with null instead of rejecting, so Promise.all doesn't fail completely
                resolve(null); 
            }
        });
    });
}

// --- Function to build the categorized tree ---
async function buildAndPopulateCategorizedTree() {
    if (!loadedObjectModelRoot || !modelTreeList) return;

    if (loaderTextElement) loaderTextElement.textContent = "Fetching object categories...";
    const promises = [];
    const objectsAndIds = [];

    // 1. Collect all named objects and create promises for fetching their category
    loadedObjectModelRoot.traverse(child => {
        if ((child.isGroup || child.isMesh) && child.name && child.parent === loadedObjectModelRoot) {
            let rawName = child.name;
            let displayId = null;
            const lastUnderscoreHalf = rawName.lastIndexOf('_');
            const lastUnderscoreFull = rawName.lastIndexOf('＿');
            let splitIndex = Math.max(lastUnderscoreHalf, lastUnderscoreFull);

            if (splitIndex > 0) displayId = rawName.substring(splitIndex + 1);

            if (displayId) {
                objectsAndIds.push({ object: child, id: displayId });
                promises.push(getCategoryInfo(parsedWSCenID, displayId));
            } else {
                objectsAndIds.push({ object: child, id: null, category: "名称未分類" });
            }
        }
    });

    try {
        // 2. Wait for all database lookups to complete
        const results = await Promise.all(promises);
        
        // 3. Group the objects by the fetched category name
        const categorizedObjects = {};
        let objectIndex = 0;
        for (let i = 0; i < objectsAndIds.length; i++) {
            const item = objectsAndIds[i];
            let category = item.category; // Use default "名称未分類" if no ID
            
            if (item.id) {
                const data = results[objectIndex];
                // IMPORTANT: Adjust 'カテゴリー名' to match the exact key from your API response
                category = data ? (data['カテゴリー名'] || "カテゴリー無し") : "カテゴリー無し";
                objectIndex++;
            }
            
            if (!categorizedObjects[category]) categorizedObjects[category] = [];
            categorizedObjects[category].push(item.object);
        }

        console.log("Data grouped by category:", categorizedObjects);

        // 4. Populate the HTML tree
        modelTreeList.innerHTML = '';
        for (const categoryName in categorizedObjects) {
            createCategoryNode(categoryName, categorizedObjects[categoryName]);
        }
        if (modelTreePanel) modelTreePanel.style.display = 'block';

    } catch (error) {
        console.error("Failed to build model tree:", error);
        alert("モデル情報の取得に失敗しました。");
    }
}

// Helper to create a category node in the tree
function createCategoryNode(categoryName, objectsInCategory) {
    const categoryLi = document.createElement('li');
    const categoryItemContent = document.createElement('div');
    categoryItemContent.className = 'tree-item';
    categoryItemContent.style.fontWeight = 'bold';

    const toggler = document.createElement('span');
    toggler.className = 'toggler';
    toggler.textContent = '▼';
    categoryItemContent.appendChild(toggler);

    const nameSpan = document.createElement('span');
    nameSpan.className = 'group-name';
    nameSpan.textContent = `${categoryName} (${objectsInCategory.length})`;
    categoryItemContent.appendChild(nameSpan);

    const subList = document.createElement('ul');

    toggler.addEventListener('click', (e) => {
        e.stopPropagation();
        const isCollapsed = subList.style.display === 'none';
        subList.style.display = isCollapsed ? 'block' : 'none';
        toggler.textContent = isCollapsed ? '▼' : '▶';
    });

    categoryLi.appendChild(categoryItemContent);
    categoryLi.appendChild(subList);
    modelTreeList.appendChild(categoryLi);

    objectsInCategory.forEach(object => {
        createObjectNode(object, subList, 1);
    });
}

// Helper to create a final object/leaf node in the tree
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

    itemContent.addEventListener('click', () => {
        handleSelection(object);
    });

    visibilityToggle.addEventListener('click', (e) => {
        e.stopPropagation();
        object.visible = !object.visible;
        visibilityToggle.classList.toggle('visible-icon', object.visible);
        visibilityToggle.classList.toggle('hidden-icon', !object.visible);
        visibilityToggle.title = object.visible ? 'Hide' : 'Show';
        if (!object.visible && selectedObjectOrGroup && selectedObjectOrGroup.uuid === object.uuid) {
            handleSelection(null); // Deselect if hidden
        }
    });
}

// Encapsulates the logic for handling a selection
function handleSelection(target) {
    removeAllHighlights();
    deIsolateAllObjects();

    let newSelection = null;
    if (target) {
        if (!selectedObjectOrGroup || selectedObjectOrGroup.uuid !== target.uuid) {
            newSelection = target;
        }
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


// --- OBJ Loading ---
if (loaderContainer) loaderContainer.style.display = 'flex';
parseObjHeader(objPath).then(() => {
    objLoader.load(objPath, (object) => {
            loadedObjectModelRoot = object;
            // ... (OBJ processing, centering, scaling, rotation) ...
            scene.add(object);
            buildAndPopulateCategorizedTree();
            // ... (camera adjustment logic) ...
            if (loaderContainer) loaderContainer.style.display = 'none';
        }, (xhr) => { /* progress */ }, (error) => { /* error */ }
    );
});


// --- Click Listener for 3D View ---
window.addEventListener('click', (event) => {
    if (!loadedObjectModelRoot || event.target.closest('#modelTreePanel')) return;

    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);

    let newlyClickedTarget = null;
    if (intersects.length > 0) {
        let intersectedObject = intersects[0].object;
        let current = intersectedObject;
        while (current && current !== scene) {
            if (current.parent === loadedObjectModelRoot && current.name) {
                newlyClickedTarget = current;
                break;
            }
            current = current.parent;
        }
    }
    handleSelection(newlyClickedTarget);
});

// --- Info Panel and other event listeners/functions ---
function updateInfoPanel() { /* ... keep this function, it uses selectedObjectOrGroup ... */ }
function generatebukkenInfoByCenId(...) { /* ... keep this function ... */ }
// ... (close button, search, resize, animate) ...
