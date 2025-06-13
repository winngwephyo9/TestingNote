import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';
import { OBJLoader } from './library/controls/OBJLoader.js';

// --- Scene Setup ---
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xcccccc); // Light gray background

// --- Camera Setup ---
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 20000); // Increased far plane
camera.position.set(10, 10, 10); // Initial camera position, will be adjusted after OBJ load
camera.lookAt(scene.position);

// --- Renderer Setup ---
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true; // Enable shadow mapping
renderer.shadowMap.type = THREE.PCFSoftShadowMap; // Softer shadows
document.body.appendChild(renderer.domElement);

// --- Lighting Setup ---
const ambientLight = new THREE.AmbientLight(0x606060, 2); // Softer ambient light
scene.add(ambientLight);

const directionalLight = new THREE.DirectionalLight(0xFFFFFF, 2.5);
directionalLight.position.set(50, 100, 75); // Adjusted light position
directionalLight.castShadow = true;
directionalLight.shadow.mapSize.width = 2048; // Higher shadow map resolution
directionalLight.shadow.mapSize.height = 2048;
directionalLight.shadow.camera.near = 0.5;
directionalLight.shadow.camera.far = 500;
directionalLight.shadow.camera.left = -100;
directionalLight.shadow.camera.right = 100;
directionalLight.shadow.camera.top = 100;
directionalLight.shadow.camera.bottom = -100;
scene.add(directionalLight);

const hemiLight = new THREE.HemisphereLight( 0xffffff, 0x8d8d8d, 1.5 );
hemiLight.position.set( 0, 50, 0 );
scene.add( hemiLight );

// --- Controls Setup ---
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05; // Smoother damping
controls.screenSpacePanning = false;
controls.minDistance = 1;
controls.maxDistance = 10000;

// --- Demo Cube (Optional) ---
const cubeGeometry = new THREE.BoxGeometry();
const cubeMaterial = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
const cube = new THREE.Mesh(cubeGeometry, cubeMaterial);
cube.name = "Demo Cube";
cube.castShadow = true;
cube.receiveShadow = true;
cube.position.x = -15; // Place it a bit away from the OBJ
scene.add(cube);

// --- OBJ Loading ---
const objLoader = new OBJLoader();
const objPath = './objFiles/standard_testing.obj'; // Make sure MTL is in the same folder

objLoader.load(
    objPath,
    (object) => {
        // 1. Center the object's internal geometry BEFORE scaling and world placement.
        const initialBox = new THREE.Box3().setFromObject(object);
        const initialCenter = initialBox.getCenter(new THREE.Vector3());
        object.traverse((child) => {
            if (child.isMesh) {
                child.geometry.translate(-initialCenter.x, -initialCenter.y, -initialCenter.z);
                child.castShadow = true; // Enable shadows for OBJ parts
                child.receiveShadow = true;
            }
        });
        object.position.set(0, 0, 0); // Place the group at origin

        // 2. Scale the object to a desired maximum dimension in world units.
        const scaledBox = new THREE.Box3().setFromObject(object); // Recalculate after translation
        const scaledSize = scaledBox.getSize(new THREE.Vector3());
        const maxDim = Math.max(scaledSize.x, scaledSize.y, scaledSize.z);

        const desiredMaxDimension = 50; // Target size for the object's longest side
        if (maxDim > 0) {
            const scale = desiredMaxDimension / maxDim;
            object.scale.set(scale, scale, scale);
        }

        // 3. Apply desired rotation
        object.rotation.y = -Math.PI / 2; // Rotate -90 degrees around Y-axis

        scene.add(object);
        console.log("OBJ loaded, centered, scaled, and rotated:", object);

        // 4. Adjust camera to frame the object
        const finalWorldBox = new THREE.Box3().setFromObject(object);
        const finalWorldCenter = finalWorldBox.getCenter(new THREE.Vector3());
        const finalWorldSize = finalWorldBox.getSize(new THREE.Vector3());

        controls.target.copy(finalWorldCenter);

        const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
        const aspect = camera.aspect;
        const effectiveSizeDimension = Math.max(finalWorldSize.y, finalWorldSize.x / aspect);
        
        let cameraDistance = effectiveSizeDimension / (2 * Math.tan(fovInRadians / 2));
        const zoomOutFactor = 1.5; // Adjust for more or less initial zoom
        cameraDistance *= zoomOutFactor;
        cameraDistance = Math.max(cameraDistance, Math.max(finalWorldSize.x, finalWorldSize.y, finalWorldSize.z) * 0.75);

        const cameraDirection = new THREE.Vector3(1, 0.6, 1).normalize();
        camera.position.copy(finalWorldCenter).addScaledVector(cameraDirection, cameraDistance);

        camera.lookAt(finalWorldCenter);
        controls.update();
        updateInfoPanel(); // Update info panel after object load
    },
    (xhr) => {
        console.log((xhr.loaded / xhr.total * 100) + '% loaded');
    },
    (error) => {
        console.error('An error happened while loading the OBJ file:', error);
        const errorDiv = document.createElement('div');
        errorDiv.textContent = `Error loading OBJ: ${objPath}. Check console, file path, and MTL file.`;
        errorDiv.style.cssText = `
            color: red; position: absolute; top: 30px; left: 10px;
            background-color: white; padding: 10px; z-index: 100; border-radius: 5px;
        `;
        document.body.appendChild(errorDiv);
    }
);

// --- Object Selection Logic ---
let selectedObjects = [];
let isShiftDown = false;
const originalMaterials = new Map(); // Stores original materials/colors for selected objects

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

const highlightColorSingle = new THREE.Color(0xffff00); // Yellow for single select
const highlightColorMulti = new THREE.Color(0xff0000); // Red for multi-select

// Helper to apply highlight
const applyHighlight = (obj, color) => {
    if (!obj.isMesh || !obj.material) return;

    if (!originalMaterials.has(obj.uuid)) {
        if (Array.isArray(obj.material)) {
            originalMaterials.set(obj.uuid, obj.material.map(m => m.clone()));
        } else {
            originalMaterials.set(obj.uuid, obj.material.clone());
        }
    }

    if (Array.isArray(obj.material)) {
        obj.material.forEach(m => {
            if(m.color) m.color.set(color);
            else if (m.emissive) m.emissive.set(color);
        });
    } else {
        if (obj.material.color) obj.material.color.set(color);
        else if (obj.material.emissive) obj.material.emissive.set(color);
    }
};

// Helper to remove highlight
const removeHighlight = (obj) => {
    if (!obj.isMesh || !obj.material) return;
    if (originalMaterials.has(obj.uuid)) {
        const originalMatSet = originalMaterials.get(obj.uuid);
        if (Array.isArray(obj.material) && Array.isArray(originalMatSet)) {
            obj.material = originalMatSet.map(m => m.clone());
        } else if (!Array.isArray(obj.material) && !Array.isArray(originalMatSet) && originalMatSet) {
            obj.material = originalMatSet.clone();
        }
        // Do not delete from originalMaterials here if you want to toggle selection
        // It will be cleared on deselect all or new single selection
    }
};

window.addEventListener('keydown', (event) => {
    if (event.key === 'Shift') isShiftDown = true;
});
window.addEventListener('keyup', (event) => {
    if (event.key === 'Shift') isShiftDown = false;
});

window.addEventListener('click', (event) => {
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(scene.children, true);

    if (intersects.length > 0) {
        let targetObject = intersects[0].object;
        // Traverse up to find the main group if the clicked part is a child mesh of a loaded OBJ
        while (targetObject.parent && targetObject.parent !== scene && !targetObject.name && !(targetObject.userData.isSelectableRoot)) {
            targetObject = targetObject.parent;
        }
        // If still no name and it's part of an OBJ group, try one more parent (often the group loaded by OBJLoader)
        if (!targetObject.name && targetObject.parent && targetObject.parent !== scene && targetObject.parent.type === "Group") {
             if (targetObject.parent.children.some(child => child === targetObject)) { // check if it's a direct child of a group
                targetObject = targetObject.parent;
            }
        }


        if (targetObject.isMesh || targetObject.isGroup) { // Allow selecting groups too
            if (isShiftDown) {
                const index = selectedObjects.findIndex(selObj => selObj.uuid === targetObject.uuid);
                if (index === -1) {
                    selectedObjects.push(targetObject);
                    targetObject.traverse(child => { if (child.isMesh) applyHighlight(child, highlightColorMulti); });
                } else {
                    selectedObjects[index].traverse(child => { if (child.isMesh) removeHighlight(child); });
                    selectedObjects.splice(index, 1);
                }
            } else {
                selectedObjects.forEach(obj => obj.traverse(child => { if (child.isMesh) removeHighlight(child); }));
                originalMaterials.clear(); // Clear for new single selection state

                selectedObjects = [targetObject];
                targetObject.traverse(child => { if (child.isMesh) applyHighlight(child, highlightColorSingle); });
            }
        }
    } else {
        selectedObjects.forEach(obj => obj.traverse(child => { if (child.isMesh) removeHighlight(child); }));
        selectedObjects = [];
        originalMaterials.clear();
    }
    updateInfoPanel();
});

// --- Info Panel Update ---
function updateInfoPanel() {
    const infoPanel = document.getElementById('objectInfo');
    if (!infoPanel) return;

    const info = selectedObjects.map(obj => {
        const pos = obj.getWorldPosition(new THREE.Vector3());
        let name = obj.name || "Unnamed Group/Object";
        if (obj.isMesh && obj.parent && obj.parent.name && obj.parent !== scene) {
            name = `${obj.parent.name} -> ${obj.name || 'Unnamed Mesh'}`;
        }
        return `${name}\n  x: ${pos.x.toFixed(2)}, y: ${pos.y.toFixed(2)}, z: ${pos.z.toFixed(2)}`;
    }).join('\n\n');
    infoPanel.textContent = info || 'None';
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
    cube.rotation.x += 0.005; // Slowed down cube rotation
    cube.rotation.y += 0.005;
    controls.update();
    renderer.render(scene, camera);
}

// --- Start ---
animate();
