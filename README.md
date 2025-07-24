import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';
import { OBJLoader } from './library/loaders/OBJLoader.js';
import { MTLLoader } from './library/loaders/MTLLoader.js';

// --- Get UI Elements ---
const loaderContainer = document.getElementById('loader-container');
const loaderTextElement = document.getElementById('loader-text');
const modelTreePanel = document.getElementById('modelTreePanel');
const modelTreeList = document.getElementById('modelTreeList');
const modelTreeSearch = document.getElementById('modelTreeSearch');
const closeModelTreeBtn = document.getElementById('closeModelTreeBtn');
const objectInfoPanel = document.getElementById('objectInfo');
const modelSelector = document.getElementById('model-selector');

// --- Global variables ---
const BOX_ACCESS_TOKEN = "YOUR_DEVELOPER_TOKEN_HERE"; // IMPORTANT: Replace with a real token
const BOX_MAIN_FOLDER_ID = "YOUR_MAIN_FOLDER_ID_HERE"; // IMPORTANT: Replace with the ID of "MainFolder"

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
const highlightColorSingle = new THREE.Color(0xa0c4ff);
const elementIdDataMap = new Map();

// --- Scene Setup, Camera, Renderer, Lighting, Controls ---
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xe8e8e8);
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 20000);
camera.position.copy(initialCameraPosition);
camera.lookAt(initialCameraLookAt);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.getElementById('viewer-container').appendChild(renderer.domElement);
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

// --- Helper Functions ---

async function parseObjHeader(objContent) {
    try {
        const lines = objContent.split(/\r?\n/);
        if (lines.length > 0) {
            const firstLine = lines[0].trim();
            if (firstLine.startsWith("# ")) {
                const content = firstLine.substring(2).trim();
                const pattern1Match = content.match(/^([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})_([a-zA-Z0-9]+)$/);
                if (pattern1Match) {
                    parsedWSCenID = pattern1Match[1];
                    parsedPJNo = pattern1Match[2];
                    return;
                }
                if (content.includes("ワークシェアリングされてない") && (content.includes("_PJNo無し") || content.includes("＿PJNo無し"))) {
                    parsedWSCenID = "";
                    parsedPJNo = "";
                    return;
                }
            }
        }
    } catch (error) {
        console.error("Error parsing OBJ header:", error);
    }
}

async function fetchAllCategoryData(wscenId, allElementIds) {
    if (allElementIds.length === 0) return;
    const batchSize = 900;
    for (let i = 0; i < allElementIds.length; i += batchSize) {
        const batch = allElementIds.slice(i, i + batchSize);
        if (loaderTextElement) loaderTextElement.textContent = `Fetching Categories... (${i + batch.length}/${allElementIds.length})`;
        await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/DLDWH/getDatas",
                data: { _token: CSRF_TOKEN, WSCenID: wscenId, ElementIds: batch },
                success: (data) => {
                    for (const elementId in data) {
                        elementIdDataMap.set(elementId, data[elementId]);
                    }
                    resolve();
                },
                error: (err) => {
                    console.error(`Error fetching batch starting at index ${i}:`, err);
                    reject(err);
                }
            });
        });
    }
}

async function fetchBoxFileContent(fileId) {
    const url = `https://api.box.com/2.0/files/${fileId}/content`;
    const headers = { 'Authorization': `Bearer ${BOX_ACCESS_TOKEN}` };
    const response = await fetch(url, { headers });
    if (!response.ok) throw new Error(`Failed to fetch file ${fileId} from Box: ${response.statusText}`);
    return await response.text();
}

function frameObject(objectToFrame) {
    const box = new THREE.Box3().setFromObject(objectToFrame);
    if (box.isEmpty()) {
        camera.position.set(50, 50, 50);
        camera.lookAt(0, 0, 0);
        controls.target.set(0, 0, 0);
        controls.update();
        return;
    }
    const center = box.getCenter(new THREE.Vector3());
    const sphere = box.getBoundingSphere(new THREE.Sphere());
    const radius = sphere.radius;
    initialCameraLookAt.copy(center);
    controls.target.copy(center);
    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    const distance = radius / Math.sin(fovInRadians / 2);
    const zoomOutFactor = 1.3;
    const finalDistance = distance * zoomOutFactor;
    const cameraDirection = new THREE.Vector3(1, 0.6, 1).normalize();
    initialCameraPosition.copy(center).addScaledVector(cameraDirection, finalDistance);
    camera.position.copy(initialCameraPosition);
    camera.lookAt(initialCameraLookAt);
    controls.update();
}

async function buildAndPopulateCategorizedTree() {
    if (!loadedObjectModelRoot || !modelTreeList) return;
    if (loaderTextElement) loaderTextElement.textContent = "Building model tree...";
    const categorizedObjects = {};
    loadedObjectModelRoot.traverse(child => {
        if (child.name && child.parent === loadedObjectModelRoot) {
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

function applyHighlight(target, color) { /* ... keep this function ... */ }
function removeAllHighlights() { /* ... keep this function ... */ }
function zoomToAndIsolate(targetObject) { /* ... keep this function ... */ }
function deIsolateAllObjects() { /* ... keep this function ... */ }
function handleSelection(target) { /* ... keep this function ... */ }
function updateInfoPanel() { /* ... keep this function ... */ }

// --- Main Application Flow ---
async function loadModel(projectFolderId, projectName) {
    if (loadedObjectModelRoot) scene.remove(loadedObjectModelRoot);
    loadedObjectModelRoot = null;
    selectedObjectOrGroup = null;
    originalMeshMaterials.clear();
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
    elementIdDataMap.clear();
    modelTreeList.innerHTML = '';
    if (modelTreePanel) modelTreePanel.style.display = 'none';
    updateInfoPanel();

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list for ${projectName}...`;
        
        const fileList = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getObjList",
                data: { _token: CSRF_TOKEN, folderId: projectFolderId, accessToken: BOX_ACCESS_TOKEN },
                success: resolve,
                error: reject
            });
        });

        if (!fileList || fileList.length === 0) throw new Error(`No valid OBJ/MTL pairs found in project "${projectName}".`);

        const loadingPromises = fileList.map((filePair, index) => {
            return new Promise(async (resolve, reject) => {
                try {
                    const mtlContent = await fetchBoxFileContent(filePair.mtl.id);
                    const objContent = await fetchBoxFileContent(filePair.obj.id);
                    if (index === 0) await parseObjHeader(objContent); // Parse header from the first file

                    const mtlLoader = new MTLLoader();
                    const materialsCreator = mtlLoader.parse(mtlContent, '');
                    materialsCreator.preload();

                    const objLoader = new OBJLoader();
                    objLoader.setMaterials(materialsCreator);
                    const object = objLoader.parse(objContent);
                    
                    if (loaderTextElement) loaderTextElement.textContent = `Loading Geometry... (${index + 1}/${fileList.length})`;
                    resolve(object);
                } catch (error) { reject(error); }
            });
        });

        const loadedObjects = await Promise.all(loadingPromises);

        const combinedModel = new THREE.Group();
        loadedObjects.forEach(object => {
            while (object.children.length > 0) {
                combinedModel.add(object.children[0]);
            }
        });
        loadedObjectModelRoot = combinedModel;

        const initialBox = new THREE.Box3().setFromObject(loadedObjectModelRoot);
        const initialCenter = initialBox.getCenter(new THREE.Vector3());
        loadedObjectModelRoot.position.sub(initialCenter);
        const scaledBox = new THREE.Box3().setFromObject(loadedObjectModelRoot);
        const maxDim = Math.max(scaledBox.getSize(new THREE.Vector3()).x, scaledBox.getSize(new THREE.Vector3()).y, scaledBox.getSize(new THREE.Vector3()).z);
        const desiredMaxDimension = 150;
        if (maxDim > 0) {
            const scale = desiredMaxDimension / maxDim;
            loadedObjectModelRoot.scale.set(scale, scale, scale);
        }
        loadedObjectModelRoot.rotation.x = -Math.PI / 2;

        const allIds = [];
        loadedObjectModelRoot.traverse(child => {
            if (child.name) {
                const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                if (splitIndex > 0) allIds.push(child.name.substring(splitIndex + 1));
            }
        });
        await fetchAllCategoryData(parsedWSCenID, [...new Set(allIds)]);
        
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        if (loaderContainer) loaderContainer.style.display = 'none';

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model: ${projectName}. Check console.`;
    }
}

async function populateProjectDropdown() {
    try {
        const projects = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getProjectList",
                data: { _token: CSRF_TOKEN, folderId: BOX_MAIN_FOLDER_ID, accessToken: BOX_ACCESS_TOKEN },
                success: resolve,
                error: reject
            });
        });
        modelSelector.innerHTML = '';
        if (projects && projects.length > 0) {
            projects.forEach(project => {
                const option = document.createElement('option');
                option.value = project.id;
                option.textContent = project.name;
                modelSelector.appendChild(option);
            });
            loadModel(projects[0].id, projects[0].name); // Load the first model by default
        } else {
            if (loaderTextElement) loaderTextElement.textContent = "No projects found in the main folder.";
        }
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
        if (loaderTextElement) loaderTextElement.textContent = "Error fetching project list from Box.";
    }
}

// --- Event Listeners and Animation Loop ---
if (modelSelector) {
    modelSelector.addEventListener('change', (event) => {
        const selectedId = event.target.value;
        const selectedName = event.target.options[event.target.selectedIndex].text;
        loadModel(selectedId, selectedName);
    });
}
window.addEventListener('click', (event) => { /* ... */ });
if (closeModelTreeBtn) { /* ... */ }
if (modelTreeSearch) { /* ... */ }
window.addEventListener('resize', () => { /* ... */ });
function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}

// --- Start Application ---
populateProjectDropdown();
animate();
