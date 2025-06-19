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
        console.log("OBJ loaded and processed:", object);

        // ... (your existing camera adjustment logic) ...
        const finalWorldBox = new THREE.Box3().setFromObject(object);
        const finalWorldCenter = finalWorldBox.getCenter(new THREE.Vector3());
        controls.target.copy(finalWorldCenter);
        // ... (camera distance calculation) ...
        const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
        const aspect = camera.aspect;
        const effectiveSizeDimension = Math.max(finalWorldBox.getSize(new THREE.Vector3()).y, finalWorldBox.getSize(new THREE.Vector3()).x / aspect);
        let cameraDistance = effectiveSizeDimension / (2 * Math.tan(fovInRadians / 2));
        const zoomOutFactor = 1.5;
        cameraDistance *= zoomOutFactor;
        cameraDistance = Math.max(cameraDistance, Math.max(finalWorldBox.getSize(new THREE.Vector3()).x, finalWorldBox.getSize(new THREE.Vector3()).y, finalWorldBox.getSize(new THREE.Vector3()).z) * 0.75);
        const cameraDirection = new THREE.Vector3(1, 0.6, 1).normalize();
        camera.position.copy(finalWorldCenter).addScaledVector(cameraDirection, cameraDistance);
        camera.lookAt(finalWorldCenter);

        controls.update();
        updateInfoPanel();

        // Hide loader now that everything is ready
        if (loaderElement) loaderElement.style.display = 'none';
        if (loaderTextElement) loaderTextElement.style.display = 'none';
    },
    (xhr) => { // Progress callback
        console.log((xhr.loaded / xhr.total * 100) + '% loaded');
        if (loaderTextElement) {
            const percentLoaded = Math.round(xhr.loaded / xhr.total * 100);
            if (isFinite(percentLoaded)) {
                 loaderTextElement.textContent = `Loading 3D Model: ${percentLoaded}%`;
            } else {
                 loaderTextElement.textContent = `Loading 3D Model...`; // Fallback if total is 0
            }
        }
    },
    (error) => { // Error callback
        console.error('An error happened while loading the OBJ file:', error);
        const errorDiv = document.createElement('div');
        // ... (your error message setup) ...
        document.body.appendChild(errorDiv);

        // Hide loader on error too, or display an error message via the loader
        if (loaderElement) loaderElement.style.display = 'none';
        if (loaderTextElement) loaderTextElement.textContent = 'Error loading model.';
    }
);




let selectedObjGroup = null;
const originalMeshMaterials = new Map();
const highlightedMeshesUuids = new Set();
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const highlightColorSingle = new THREE.Color(0xffff00);

const applyHighlight = (target, color) => { // Renamed from applyHighlightToMeshesInGroup
    if (!target) return;
    // console.log("Applying highlight to:", target.name || target.uuid, target.type);

    const meshesToHighlight = [];
    if (target.isMesh) {
        meshesToHighlight.push(target);
    } else if (target.isGroup) {
        target.traverse((child) => {
            if (child.isMesh) {
                meshesToHighlight.push(child);
            }
        });
    }

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
    // console.log("Removing all highlights. Meshes with stored original materials:", originalMeshMaterials.size);
    originalMeshMaterials.forEach((originalMaterialOrArray, meshUuid) => {
        const mesh = scene.getObjectByProperty('uuid', meshUuid);
        if (mesh && mesh.isMesh) {
            mesh.material = originalMaterialOrArray;
        }
    });
    originalMeshMaterials.clear();
};

window.addEventListener('click', (event) => {
    if (!loadedObjectModelRoot) {
        console.log("OBJ model not yet loaded.");
        return;
    }
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);
    let newlyClickedTarget = null;
    if (intersects.length > 0) {
        let intersectedObject = intersects[0].object;
        // console.log("Directly intersected:", intersectedObject.name || "unnamed_object", intersectedObject.type, intersectedObject.uuid);
        // if (intersectedObject.parent) {
        //     console.log("Parent of intersected:", intersectedObject.parent.name || "unnamed_parent", intersectedObject.parent.type, intersectedObject.parent.uuid);
        // }
        if (intersectedObject.isMesh && intersectedObject.name && intersectedObject.parent === loadedObjectModelRoot) {
            newlyClickedTarget = intersectedObject;
        }
        else if (intersectedObject.isMesh &&
                 intersectedObject.parent &&
                 intersectedObject.parent.isGroup &&
                 intersectedObject.parent.name &&
                 intersectedObject.parent.parent === loadedObjectModelRoot) {
            newlyClickedTarget = intersectedObject.parent;
        }
        else if (intersectedObject.isGroup && intersectedObject.name && intersectedObject.parent === loadedObjectModelRoot) {
            newlyClickedTarget = intersectedObject;
        }
        // if (newlyClickedTarget) {
        //     console.log("SUCCESS: Identified target object/group:", newlyClickedTarget.name);
        // } else {
        //     console.warn("WARNING: Could not identify a distinct named target for the clicked mesh/object under the loaded OBJ root.");
        // }
    }
    removeAllHighlights();
    if (newlyClickedTarget) {
        if (!selectedObjGroup || selectedObjGroup.uuid !== newlyClickedTarget.uuid) {
            selectedObjGroup = newlyClickedTarget;
            applyHighlight(selectedObjGroup, highlightColorSingle); // Use generic applyHighlight
        } else {
            selectedObjGroup = null;
        }
    } else {
        selectedObjGroup = null;
    }
    updateInfoPanel();
});



if (lastUnderscoreHalf !== -1 && lastUnderscoreFull !== -1) {
            splitIndex = Math.max(lastUnderscoreHalf, lastUnderscoreFull);
        } else if (lastUnderscoreHalf !== -1) {
            splitIndex = lastUnderscoreHalf;
        } else if (lastUnderscoreFull !== -1) {
            splitIndex = lastUnderscoreFull;
        }
        if (splitIndex > 0 && splitIndex < rawName.length - 1) {
            displayName = rawName.substring(0, splitIndex);
            displayId = rawName.substring(splitIndex + 1);
        } else {
            displayName = rawName;
        }
        
        // --- Window Resize ---
// ... (Your existing resize listener) ...
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// --- Animation Loop ---
// ... (Your existing animate function) ...
function animate() {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
}




import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';
import { OBJLoader } from './library/controls/OBJLoader.js';

// --- Get Loader Elements ---
const loaderElement = document.getElementById('loader');
const loaderTextElement = document.getElementById('loader-text');

// --- Global variables for parsed OBJ header info ---
let parsedWSCenID = "";
let parsedPJNo = "";

// --- Scene Setup ---
// ... (rest of your scene, camera, renderer, lighting, controls setup remains the same)
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xcccccc);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 20000);
camera.position.set(10, 10, 10);
camera.lookAt(scene.position);

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
// ... (shadow properties)
scene.add(directionalLight);

const hemiLight = new THREE.HemisphereLight( 0xffffff, 0x8d8d8d, 1.5 );
hemiLight.position.set( 0, 50, 0 );
scene.add( hemiLight );

const controls = new OrbitControls(camera, renderer.domElement);
// ... (controls properties)


// --- Global variable to store the loaded OBJ object ---
let loadedObjectModelRoot = null;

// --- Function to parse the first line of OBJ for WSCenID and PJNo ---
async function parseObjHeader(filePath) {
    try {
        const response = await fetch(filePath);
        if (!response.ok) {
            console.error(`Failed to fetch OBJ for header parsing: ${response.statusText}`);
            return;
        }
        const text = await response.text();
        const lines = text.split(/\r?\n/); // Split into lines

        if (lines.length > 0) {
            const firstLine = lines[0].trim();
            console.log("First line of OBJ:", firstLine); // Debugging

            if (firstLine.startsWith("# ")) {
                const content = firstLine.substring(2).trim(); // Remove "# "
                
                // Pattern 1: "d1cbc31e-a482-4876-9e3a-48775895e671_SA00000011"
                // WSCenID = d1cbc31e-a482-4876-9e3a-48775895e671
                // PJNo = SA00000011
                const pattern1Match = content.match(/^([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})_([a-zA-Z0-9]+)$/);
                if (pattern1Match) {
                    parsedWSCenID = pattern1Match[1];
                    parsedPJNo = pattern1Match[2];
                    console.log(`Pattern 1 Matched: WSCenID=${parsedWSCenID}, PJNo=${parsedPJNo}`);
                    return; // Successfully parsed
                }

                // Pattern 2: "ワークシェアリングされてない_PJNo無し" (or similar with full-width underscore)
                // WSCenID = "" (blank)
                // PJNo = "" (blank)
                // We can check for the specific Japanese text or a more general pattern if needed.
                // For now, let's assume if it doesn't match pattern 1, and contains "PJNo無し" it's pattern 2.
                if (content.includes("ワークシェアリングされてない") && (content.includes("_PJNo無し") || content.includes("＿PJNo無し"))) {
                    parsedWSCenID = "";
                    parsedPJNo = "";
                    console.log(`Pattern 2 Matched (ワークシェアリング無し): WSCenID=${parsedWSCenID}, PJNo=${parsedPJNo}`);
                    return; // Successfully parsed as pattern 2
                }
            }
        }
        console.log("First line did not match expected patterns for WSCenID/PJNo.");
    } catch (error) {
        console.error("Error fetching or parsing OBJ header:", error);
    }
}


// --- OBJ Loading ---
const objLoader = new OBJLoader();
const objPath = './objFiles/standard_testing.obj'; //  <-- MAKE SURE THIS IS YOUR CORRECT OBJ FILE PATH

// Show loader before starting
if (loaderElement) loaderElement.style.display = 'block';
if (loaderTextElement) loaderTextElement.style.display = 'block';

// Parse header first, then load OBJ
parseObjHeader(objPath).then(() => {
    // Now that header parsing is attempted, proceed to load the OBJ with OBJLoader
    objLoader.load(
        objPath,
        (object) => { // Success callback
            loadedObjectModelRoot = object;
            // ... (rest of your OBJ processing and camera adjustment logic from previous correct version)
            const initialBox = new THREE.Box3().setFromObject(object);
            const initialCenter = initialBox.getCenter(new THREE.Vector3());
            object.traverse((child) => { /* ... */ });
            object.position.set(0, 0, 0);
            // ... scaling ...
            const desiredMaxDimension = 50;
            // ...
            object.rotation.x = -Math.PI / 2; 
            object.rotation.y = -Math.PI / 2;
            scene.add(object);
            console.log("OBJ loaded and processed by OBJLoader:", object);
            // ... camera adjustment ...
            controls.update();
            updateInfoPanel(); // This will now also have access to parsedWSCenID and parsedPJNo

            if (loaderElement) loaderElement.style.display = 'none';
            if (loaderTextElement) loaderTextElement.style.display = 'none';
        },
        (xhr) => { /* ... progress callback ... */ },
        (error) => { /* ... error callback ... */ }
    );
});


// --- Object Selection Logic ---
// ... (Your existing selection logic: selectedObjGroup, originalMeshMaterials, highlightedMeshesUuids,
//      raycaster, mouse, highlightColorSingle, applyHighlight, removeAllHighlights, click listener) ...
// Make sure this section is using the latest corrected version from our previous discussions.
let selectedObjGroup = null;
// ... etc.


// --- Info Panel Update ---
// Modify updateInfoPanel to include parsedWSCenID and parsedPJNo if available
function updateInfoPanel() {
    const infoPanel = document.getElementById('objectInfo');
    if (!infoPanel) return;

    let headerInfo = "";
    if (parsedWSCenID || parsedPJNo) { // Only show if we have something
        headerInfo = `WSCenID: ${parsedWSCenID || "N/A"}\nPJNo: ${parsedPJNo || "N/A"}\n----\n`;
    } else if (parsedWSCenID === "" && parsedPJNo === "") { // Specifically for "PJNo無し" case
        headerInfo = `WSCenID: \nPJNo: \n----\n`;
    }


    if (selectedObjGroup) {
        const pos = selectedObjGroup.getWorldPosition(new THREE.Vector3());
        let rawName = selectedObjGroup.name || "Unnamed Group/Object";
        let displayName = rawName;
        let displayId = "N/A";

        const lastUnderscoreHalf = rawName.lastIndexOf('_');
        const lastUnderscoreFull = rawName.lastIndexOf('＿');
        let splitIndex = -1;
        // ... (your name/ID splitting logic) ...
        if (lastUnderscoreHalf !== -1 && lastUnderscoreFull !== -1) { /* ... */ }

        const selectionInfo = `名前: ${displayName}\n  ID: ${displayId}\n  x: ${pos.x.toFixed(2)}, y: ${pos.y.toFixed(2)}, z: ${pos.z.toFixed(2)}`;
        infoPanel.textContent = headerInfo + selectionInfo;
    } else {
        infoPanel.textContent = headerInfo + 'None Selected';
    }
}

// --- Window Resize ---
// ... (Your existing resize listener) ...

// --- Animation Loop ---
// ... (Your existing animate function) ...

// --- Start ---
animate();
