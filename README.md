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

// NEW: A map to store all fetched element data for fast lookups
const elementIdDataMap = new Map();


// --- Scene Setup, Camera, Renderer, Lighting, Controls ---
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
scene.add(directionalLight);
const hemiLight = new THREE.HemisphereLight(0xffffff, 0x8d8d8d, 1.5);
hemiLight.position.set(0, 50, 0);
scene.add(hemiLight);
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;

// --- Helper Functions ---

async function parseObjHeader(filePath) { /* ... keep this function as is ... */ }

// --- NEW Data Fetching Function ---
async function fetchAllCategoryData(wscenId, allElementIds) {
    if (allElementIds.length === 0) {
        return; // Nothing to fetch
    }
    return new Promise((resolve) => {
        $.ajax({
            type: "post",
            url: url_prefix + "/DLDWH/getDatas", // The NEW route
            data: { 
                _token: CSRF_TOKEN, 
                WSCenID: wscenId, 
                ElementIds: allElementIds // Send array of IDs
            },
            success: function (data) {
                console.log("Batch data received:", data);
                // The backend returns an object keyed by ElementId, so we can create a Map from it
                for (const elementId in data) {
                    elementIdDataMap.set(elementId, data[elementId]);
                }
                resolve(); // Resolve when done
            },
            error: function (err) {
                console.error(`Error fetching batch data:`, err);
                alert("モデル情報の取得に失敗しました。");
                resolve(); // Resolve anyway so the app doesn't hang
            }
        });
    });
}

// --- NEW Model Tree Logic ---
function buildAndPopulateCategorizedTree() {
    if (!loadedObjectModelRoot || !modelTreeList) return;

    // This function now assumes elementIdDataMap is already populated
    const categorizedObjects = {};

    loadedObjectModelRoot.traverse(child => {
        if ((child.isGroup || child.isMesh) && child.name && child.parent === loadedObjectModelRoot) {
            let rawName = child.name;
            let displayId = null;
            const splitIndex = Math.max(rawName.lastIndexOf('_'), rawName.lastIndexOf('＿'));
            if (splitIndex > 0) displayId = rawName.substring(splitIndex + 1);

            let category = "カテゴリー無し"; // Default
            if (displayId && elementIdDataMap.has(displayId)) {
                // IMPORTANT: Adjust 'カテゴリー名' to match the exact key from your API response
                category = elementIdDataMap.get(displayId)['カテゴリー名'] || "カテゴリー無し";
            } else if (!displayId) {
                category = "名称未分類";
            }
            
            if (!categorizedObjects[category]) categorizedObjects[category] = [];
            categorizedObjects[category].push(child);
        }
    });

    console.log("Data grouped by category:", categorizedObjects);

    modelTreeList.innerHTML = '';
    Object.keys(categorizedObjects).sort().forEach(categoryName => {
        createCategoryNode(categoryName, categorizedObjects[categoryName]);
    });
    if (modelTreePanel) modelTreePanel.style.display = 'block';
}

function createCategoryNode(categoryName, objectsInCategory) {
    // ... This function remains the same as before ...
}

function createObjectNode(object, parentULElement, depth) {
    // ... This function remains the same as before ...
}


// --- Main Application Flow ---
async function main() {
    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        const objPath = '/ccc/public/objFiles/240324_GF本社移転_2022_20250618.obj';
        
        // Step 1: Parse header info
        await parseObjHeader(objPath);
        
        // Step 2: Load the OBJ model geometry
        const object = await new Promise((resolve, reject) => {
            objLoader.load(objPath, resolve, (xhr) => {
                if (loaderTextElement) {
                    const percent = Math.round(xhr.loaded / xhr.total * 100);
                    loaderTextElement.textContent = isFinite(percent) && percent < 100 ?
                        `Loading 3D Geometry: ${percent}%` : `Processing Geometry...`;
                }
            }, reject);
        });

        loadedObjectModelRoot = object;

        // Step 3: Process the loaded model (center, scale, rotate)
        // ... (Keep the centering and scaling logic as is) ...
        const desiredMaxDimension = 150; 
        
        // Step 4: Collect all IDs and fetch all category data in one batch
        const allIds = [];
        loadedObjectModelRoot.traverse(child => {
            if (child.name && child.parent === loadedObjectModelRoot) {
                const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                if (splitIndex > 0) allIds.push(child.name.substring(splitIndex + 1));
            }
        });
        await fetchAllCategoryData(parsedWSCenID, [...new Set(allIds)]); // Use Set to get unique IDs

        // Step 5: Build the tree with the now-populated data map
        buildAndPopulateCategorizedTree();
        
        // Step 6: NOW add the model to the scene and frame it
        scene.add(object);
        frameObject(object);
        
        // Step 7: Hide the loader
        if (loaderContainer) loaderContainer.style.display = 'none';

    } catch (error) {
        console.error('Failed to initialize the viewer:', error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = 'Error during initialization.';
    }
}


// --- All other functions (selection, highlighting, isolation, info panel, events) ---
// IMPORTANT: The updateInfoPanel and generatebukkenInfoByCenId are now REPLACED
// by a single, synchronous updateInfoPanel function.

function handleSelection(target) { /* ... Keep this function as is ... */ }
function removeAllHighlights() { /* ... Keep this function as is ... */ }
function applyHighlight(target, color) { /* ... Keep this function as is ... */ }
function deIsolateAllObjects() { /* ... Keep this function as is ... */ }
function zoomToAndIsolate(targetObject) { /* ... Keep this function as is ... */ }

// --- REVISED Info Panel Update ---
function updateInfoPanel() {
    if (!objectInfoPanel) return;
    let headerInfo = "";
    if (parsedWSCenID || parsedPJNo) {
        headerInfo = `WSCenID: ${parsedWSCenID || "N/A"}\nPJNo: ${parsedPJNo || "N/A"}\n----\n`;
    } else if (parsedWSCenID === "" && parsedPJNo === "") {
        headerInfo = `WSCenID: \nPJNo: \n----\n`;
    }

    if (selectedObjectOrGroup) {
        let rawName = selectedObjectOrGroup.name || "Unnamed";
        let displayName = rawName;
        let displayId = "N/A";
        const splitIndex = Math.max(rawName.lastIndexOf('_'), rawName.lastIndexOf('＿'));
        if (splitIndex > 0) {
            displayName = rawName.substring(0, splitIndex);
            displayId = rawName.substring(splitIndex + 1);
        } else displayName = rawName;
        
        let selectionInfo = `名前: ${displayName}\nID: ${displayId}\n`;

        // Get extra info from the pre-fetched map
        if (displayId && elementIdDataMap.has(displayId)) {
            const data = elementIdDataMap.get(displayId);
            const categoryName = data['カテゴリー名'] || "N/A";
            const familyName = data['ファミリ名'] || "N/A";
            const typeId = data['タイプ_ID'] || "N/A";
            selectionInfo += `\nカテゴリー名: ${categoryName}\nファミリ名: ${familyName}\nタイプ_ID: ${typeId}`;
        } else {
             selectionInfo += '\nカテゴリー名: "" \nファミリ名: ""';
        }
        
        objectInfoPanel.textContent = headerInfo + selectionInfo;
    } else {
        objectInfoPanel.textContent = headerInfo + 'None Selected';
    }
}

// The old generatebukkenInfoByCenId is NO LONGER NEEDED as data is pre-fetched.

// --- All Event Listeners and Animation Loop ---
window.addEventListener('click', (event) => { /* ... Keep this function ... */ });
if (closeModelTreeBtn) { /* ... Keep this function ... */ }
if (modelTreeSearch) { /* ... Keep this function ... */ }
window.addEventListener('resize', () => { /* ... */ });
function animate() { /* ... */ }

// --- Start Application ---
main();
animate();
