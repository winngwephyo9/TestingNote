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
        console.error("Error fetching or parsing OBJ header:", error);
    }
}

function getCategoryInfo(wscenId, elementId) {
    return new Promise((resolve, reject) => {
        // --- THIS IS A MOCKUP for testing. Replace with your actual AJAX call. ---
        const mockDatabase = {
            "5328476": { "カテゴリー名": "スペース", "ファミリ名": "スペースファミリー" },
            "5328477": { "カテゴリー名": "スペース", "ファミリ名": "スペースファミリー" },
            "9387209": { "カテゴリー名": "構造基礎", "ファミリ名": "S_F4" },
            "9394301": { "カテゴリー名": "壁", "ファミリ名": "標準壁 150" },
            "default": { "カテゴリー名": "一般モデル", "ファミリ名": "不明なファミリー" }
        };
        setTimeout(() => {
            const data = mockDatabase[elementId] ? [mockDatabase[elementId]] : [mockDatabase["default"]];
            const result = (data && data.length > 0) ? data[0] : null;
            resolve(result);
        }, 30); // Simulate network delay
        // --- MOCKUP END ---

        /*
        // --- YOUR ACTUAL AJAX CALL ---
        $.ajax({
            type: "post",
            url: url_prefix + "/DLDWH/getData",
            data: { _token: CSRF_TOKEN, WSCenID: wscenId, ElementId: elementId },
            success: function (data) {
                const result = (data && data.length > 0) ? data[0] : null;
                resolve(result);
            },
            error: function (err) {
                console.error(`Error fetching data for ElementId ${elementId}:`, err);
                resolve(null); // Resolve null on error to not break Promise.all
            }
        });
        */
    });
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
    if (loaderTextElement) loaderTextElement.textContent = "Fetching object categories...";

    const promises = [];
    const objectsAndIds = [];

    loadedObjectModelRoot.traverse(child => {
        if ((child.isGroup || child.isMesh) && child.name && child.parent === loadedObjectModelRoot) {
            let rawName = child.name;
            let displayId = null;
            const splitIndex = Math.max(rawName.lastIndexOf('_'), rawName.lastIndexOf('＿'));
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
        const results = await Promise.all(promises);
        const categorizedObjects = {};
        let objectIndexWithPromise = 0;
        for (const item of objectsAndIds) {
            let category = item.category;
            if (item.id) {
                const data = results[objectIndexWithPromise];
                category = data ? (data['カテゴリー名'] || "カテゴリー無し") : "カテゴリー無し";
                objectIndexWithPromise++;
            }
            if (!categorizedObjects[category]) categorizedObjects[category] = [];
            categorizedObjects[category].push(item.object);
        }

        modelTreeList.innerHTML = '';
        Object.keys(categorizedObjects).sort().forEach(categoryName => {
            createCategoryNode(categoryName, categorizedObjects[categoryName]);
        });

        if (modelTreePanel) modelTreePanel.style.display = 'block';
    } catch (error) {
        console.error("Failed to build model tree:", error);
        alert("モデル情報の取得に失敗しました。");
    }
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

// --- Main Application Flow ---
async function main() {
    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        await parseObjHeader(objPath);
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
        const desiredMaxDimension = 150; // Adjusted for better initial size
        if (maxDim > 0) {
            const scale = desiredMaxDimension / maxDim;
            object.scale.set(scale, scale, scale);
        }
        object.rotation.x = -Math.PI / 2;
        object.rotation.y = -Math.PI / 2;

        await buildAndPopulateCategorizedTree();
        
        scene.add(object);
        frameObject(object);
        
        if (loaderContainer) loaderContainer.style.display = 'none';
    } catch (error) {
        console.error('Failed to initialize the viewer:', error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = 'Error during initialization.';
    }
}

// --- Event Listeners and Animation Loop ---
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
window.addEventListener('click', (event) => {
    if (!loadedObjectModelRoot || event.target.closest('#modelTreePanel')) return;
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);
    let newlyClickedTarget = null;
    if (intersects.length > 0) {
        let current = intersects[0].object;
        while (current && current.parent !== loadedObjectModelRoot && current.parent !== scene) {
            current = current.parent;
        }
        if (current && current.parent === loadedObjectModelRoot) newlyClickedTarget = current;
    }
    handleSelection(newlyClickedTarget);
});
if (closeModelTreeBtn) closeModelTreeBtn.addEventListener('click', () => { if (modelTreePanel) modelTreePanel.style.display = 'none'; });
if (modelTreeSearch) { /* keep search logic */ }
window.addEventListener('resize', () => { /* ... */ });
function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}
// Start Application
main();
animate();
