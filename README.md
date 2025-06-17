import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';
import { OBJLoader } from './library/controls/OBJLoader.js';

// --- Scene Setup ---
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xcccccc);

// --- Camera Setup ---
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 20000);
camera.position.set(10, 10, 10);
camera.lookAt(scene.position);

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

// --- OBJ Loading ---
const objLoader = new OBJLoader();
const objPath = './objFiles/standard_testing.obj'; //  <-- MAKE SURE THIS IS YOUR CORRECT OBJ FILE PATH

objLoader.load(
    objPath,
    (object) => {
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

        object.rotation.x = -Math.PI / 2; // Stand up
        object.rotation.y = -Math.PI / 2; // Orient front

        scene.add(object);
        console.log("OBJ loaded, centered, scaled, and rotated:", object);

        const finalWorldBox = new THREE.Box3().setFromObject(object);
        const finalWorldCenter = finalWorldBox.getCenter(new THREE.Vector3());
        const finalWorldSize = finalWorldBox.getSize(new THREE.Vector3());

        controls.target.copy(finalWorldCenter);

        const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
        const aspect = camera.aspect;
        const effectiveSizeDimension = Math.max(finalWorldSize.y, finalWorldSize.x / aspect);
        
        let cameraDistance = effectiveSizeDimension / (2 * Math.tan(fovInRadians / 2));
        const zoomOutFactor = 1.5;
        cameraDistance *= zoomOutFactor;
        cameraDistance = Math.max(cameraDistance, Math.max(finalWorldSize.x, finalWorldSize.y, finalWorldSize.z) * 0.75);

        const cameraDirection = new THREE.Vector3(1, 0.6, 1).normalize();
        camera.position.copy(finalWorldCenter).addScaledVector(cameraDirection, cameraDistance);

        camera.lookAt(finalWorldCenter);
        controls.update();
        updateInfoPanel();
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
let selectedHierarchyObject = null;
const originalMeshMaterials = new Map();
const highlightedMeshes = new Set();

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const highlightColorSingle = new THREE.Color(0xffff00);

const applyHighlightToMeshesOfObject = (objectToHighlightRoot, color) => {
    if (!objectToHighlightRoot) return;
    objectToHighlightRoot.traverse((childMesh) => {
        if (childMesh.isMesh && childMesh.material) {
            if (!highlightedMeshes.has(childMesh.uuid)) {
                if (Array.isArray(childMesh.material)) {
                    originalMeshMaterials.set(childMesh.uuid, childMesh.material.map(m => m.clone()));
                } else {
                    originalMeshMaterials.set(childMesh.uuid, childMesh.material.clone());
                }

                if (Array.isArray(childMesh.material)) {
                    childMesh.material.forEach(m => {
                        if (m.color) m.color.set(color);
                        else if (m.emissive) m.emissive.set(color);
                    });
                } else {
                    if (childMesh.material.color) childMesh.material.color.set(color);
                    else if (childMesh.material.emissive) child.material.emissive.set(color);
                }
                highlightedMeshes.add(childMesh.uuid);
            }
        }
    });
};

const removeAllHighlights = () => {
    highlightedMeshes.forEach(meshUuid => {
        const mesh = scene.getObjectByProperty('uuid', meshUuid);
        if (mesh && mesh.isMesh && originalMeshMaterials.has(meshUuid)) {
            const originalMatSet = originalMeshMaterials.get(meshUuid);
            if (Array.isArray(mesh.material) && Array.isArray(originalMatSet)) {
                mesh.material = originalMatSet.map(m => m.clone());
            } else if (!Array.isArray(mesh.material) && !Array.isArray(originalMatSet) && originalMatSet) {
                mesh.material = originalMatSet.clone();
            }
        }
    });
    highlightedMeshes.clear();
    originalMeshMaterials.clear();
};

window.addEventListener('click', (event) => {
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(scene.children, true);

    let newlyClickedObjectRoot = null;

    if (intersects.length > 0) {
        let intersectTarget = intersects[0].object;
        while (intersectTarget.parent && intersectTarget.parent !== scene) {
            if (intersectTarget.parent.type === 'Group' && intersectTarget.parent.name) {
                intersectTarget = intersectTarget.parent;
                break;
            }
            if (intersectTarget.parent === scene && (intersectTarget.name || intersectTarget.type === 'Group')) {
                break;
            }
            if (intersectTarget.name && (!intersectTarget.parent.name || intersectTarget.parent === scene)) {
                break;
            }
            intersectTarget = intersectTarget.parent;
        }
        newlyClickedObjectRoot = intersectTarget;
    }

    removeAllHighlights();

    if (newlyClickedObjectRoot) {
        if (!selectedHierarchyObject || selectedHierarchyObject.uuid !== newlyClickedObjectRoot.uuid) {
            selectedHierarchyObject = newlyClickedObjectRoot;
            applyHighlightToMeshesOfObject(selectedHierarchyObject, highlightColorSingle);
        } else {
            selectedHierarchyObject = null;
        }
    } else {
        selectedHierarchyObject = null;
    }
    updateInfoPanel();
});

// --- Info Panel Update ---
function updateInfoPanel() {
    const infoPanel = document.getElementById('objectInfo');
    if (!infoPanel) return;

    if (selectedHierarchyObject) {
        const pos = selectedHierarchyObject.getWorldPosition(new THREE.Vector3());
        let rawName = selectedHierarchyObject.name || "Unnamed Group/Object";

        let displayName = rawName;
        let displayId = "N/A";

        const lastUnderscoreHalf = rawName.lastIndexOf('_');
        const lastUnderscoreFull = rawName.lastIndexOf('＿');
        let splitIndex = -1;

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
        
        const info = `名前: ${displayName}\n  ID: ${displayId}\n  x: ${pos.x.toFixed(2)}, y: ${pos.y.toFixed(2)}, z: ${pos.z.toFixed(2)}`;
        infoPanel.textContent = info;
    } else {
        infoPanel.textContent = 'None';
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

// --- Start ---
animate();
