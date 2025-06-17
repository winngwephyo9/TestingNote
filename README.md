![image](https://github.com/user-attachments/assets/b74ece0e-51d8-41a5-bc47-fb062c3c7ad9)



window.addEventListener('click', (event) => {
    if (!loadedObjectModelRoot) {
        console.log("OBJ model not yet loaded.");
        return;
    }

    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true); // Intersect within the loaded model

    let newlyClickedTargetGroup = null;

    if (intersects.length > 0) {
        let intersectedMesh = intersects[0].object; // This is the THREE.Mesh that was clicked

        console.log("Directly intersected mesh:", intersectedMesh.name || "unnamed_mesh", intersectedMesh.uuid);

        if (intersectedMesh.parent) {
            console.log("Parent of mesh:", intersectedMesh.parent.name || "unnamed_group", intersectedMesh.parent.type, intersectedMesh.parent.uuid);
            if (intersectedMesh.parent.parent) {
                 console.log("Grandparent of mesh:", intersectedMesh.parent.parent.name || "unnamed_grandparent", intersectedMesh.parent.parent.type, intersectedMesh.parent.parent.uuid);
            }
        }


        // The 'g' groups from OBJLoader are usually direct children of the 'loadedObjectModelRoot'
        // and these 'g' groups contain the actual meshes.
        // So, the parent of the intersectedMesh should be our target group.
        if (intersectedMesh.parent && 
            intersectedMesh.parent.isGroup && 
            intersectedMesh.parent.name && // Check if the parent group has a name
            intersectedMesh.parent.parent === loadedObjectModelRoot) { // Check if this named group is a child of our main loaded object
            
            newlyClickedTargetGroup = intersectedMesh.parent;

        } else if (intersectedMesh.isGroup && intersectedMesh.name && intersectedMesh.parent === loadedObjectModelRoot) {
            // Less common: If the ray somehow intersected the named group itself directly (e.g. it had a bounding box)
            // and this group is a child of loadedObjectModelRoot.
            newlyClickedTargetGroup = intersectedMesh;
        }
        // Further fallback if the structure is even more nested or different
        // This part might be less necessary if the above covers most OBJ outputs
        else {
            let current = intersectedMesh;
            while(current && current !== scene && current !== loadedObjectModelRoot.parent /* Should not happen */){
                 // Check if 'current' itself is one of the direct children of loadedObjectModelRoot and is a named group
                if(current.parent === loadedObjectModelRoot && current.isGroup && current.name) {
                    newlyClickedTargetGroup = current;
                    break;
                }
                // Or if 'current' is a mesh, and its parent is a named group child of loadedObjectModelRoot
                if(current.isMesh && current.parent && current.parent.isGroup && current.parent.name && current.parent.parent === loadedObjectModelRoot) {
                    newlyClickedTargetGroup = current.parent;
                    break;
                }
                current = current.parent;
            }
        }


        if (newlyClickedTargetGroup) {
            console.log("SUCCESS: Identified target group:", newlyClickedTargetGroup.name);
        } else {
            console.warn("WARNING: Could not identify a named target group for the clicked mesh. Clicked object might not be part of a named 'g' group directly under the loaded OBJ root, or ray intersected something else.");
        }
    }

    // --- The rest of the logic remains the same ---
    removeAllHighlights();

    if (newlyClickedTargetGroup) {
        if (!selectedObjGroup || selectedObjGroup.uuid !== newlyClickedTargetGroup.uuid) {
            selectedObjGroup = newlyClickedTargetGroup;
            applyHighlightToMeshesInGroup(selectedObjGroup, highlightColorSingle);
        } else {
            // Clicked the same group again, deselect it (highlights already removed by removeAllHighlights)
            selectedObjGroup = null;
        }
    } else {
        // Clicked on empty space or an unidentifiable part
        selectedObjGroup = null;
    }

    updateInfoPanel();
});


![image](https://github.com/user-attachments/assets/99472d28-3a35-4426-ade8-3bb63ac49027)


![image](https://github.com/user-attachments/assets/1704d9f3-b2b4-482c-8fc5-4fc03fb469e7)




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

// --- Global variable to store the loaded OBJ object ---
let loadedObjectModelRoot = null;

// --- OBJ Loading ---
const objLoader = new OBJLoader();
// MAKE SURE to use your *NEW* OBJ file path if you changed it
const objPath = './objFiles/standard_testing.obj'; // Or your new file path

objLoader.load(
    objPath,
    (object) => {
        loadedObjectModelRoot = object; // Store the root of the loaded OBJ

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
        console.log("OBJ loaded and processed:", object);

        const finalWorldBox = new THREE.Box3().setFromObject(object);
        const finalWorldCenter = finalWorldBox.getCenter(new THREE.Vector3());
        controls.target.copy(finalWorldCenter);

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
    },
    (xhr) => {
        console.log((xhr.loaded / xhr.total * 100) + '% loaded');
    },
    (error) => {
        console.error('An error happened while loading the OBJ file:', error);
        const errorDiv = document.createElement('div');
        errorDiv.textContent = `Error loading OBJ: ${objPath}. Check console, file path, and MTL file. Make sure the OBJ is not corrupted (e.g., has NaN issues).`;
        errorDiv.style.cssText = `
            color: red; position: absolute; top: 30px; left: 10px;
            background-color: white; padding: 10px; z-index: 100; border-radius: 5px;
        `;
        document.body.appendChild(errorDiv);
    }
);

// --- Object Selection Logic ---
let selectedObjGroup = null;
const originalMeshMaterials = new Map();
const highlightedMeshesUuids = new Set();

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const highlightColorSingle = new THREE.Color(0xffff00);

const applyHighlightToMeshesInGroup = (groupNode, color) => {
    if (!groupNode) return;
    console.log("Applying highlight to group:", groupNode.name);
    groupNode.traverse((childMesh) => {
        if (childMesh.isMesh && childMesh.material) {
            // Check if this childMesh is a direct descendant of the groupNode
            // This check is important if groupNode itself might be a mesh (though less likely for OBJs 'g')
            let isDirectDescendant = false;
            let p = childMesh.parent;
            while(p && p !== scene) {
                if (p === groupNode) {
                    isDirectDescendant = true;
                    break;
                }
                p = p.parent;
            }
            // If groupNode is the mesh itself
            if (childMesh === groupNode) isDirectDescendant = true;


            if (isDirectDescendant) {
                if (!originalMeshMaterials.has(childMesh.uuid)) {
                    if (Array.isArray(childMesh.material)) {
                        originalMeshMaterials.set(childMesh.uuid, childMesh.material.map(m => m.clone()));
                    } else {
                        originalMeshMaterials.set(childMesh.uuid, childMesh.material.clone());
                    }
                }

                if (Array.isArray(childMesh.material)) {
                    childMesh.material.forEach(m => {
                        if (m.color) m.color.set(color);
                        else if (m.emissive) m.emissive.set(color);
                    });
                } else {
                    if (childMesh.material.color) childMesh.material.color.set(color);
                    else if (childMesh.material.emissive) childMesh.material.emissive.set(color);
                }
                highlightedMeshesUuids.add(childMesh.uuid);
            }
        }
    });
};

const removeAllHighlights = () => {
    console.log("Removing all highlights. Currently highlighted:", highlightedMeshesUuids.size);
    highlightedMeshesUuids.forEach(meshUuid => {
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
    highlightedMeshesUuids.clear();
    originalMeshMaterials.clear();
};

window.addEventListener('click', (event) => {
    if (!loadedObjectModelRoot) { // Don't do anything if OBJ hasn't loaded
        console.log("OBJ model not yet loaded.");
        return;
    }

    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

    raycaster.setFromCamera(mouse, camera);
    // Crucially, intersect with the children of the loadedObjectModelRoot
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);

    let newlyClickedTargetGroup = null;

    if (intersects.length > 0) {
        let intersectedMesh = intersects[0].object; // This must be a THREE.Mesh

        console.log("Directly intersected mesh:", intersectedMesh.name || "unnamed", intersectedMesh.uuid);
        if(intersectedMesh.parent) console.log("Parent:", intersectedMesh.parent.name || "unnamed", intersectedMesh.parent.uuid, "Parent of Parent:", intersectedMesh.parent.parent ? (intersectedMesh.parent.parent.name || "unnamed") : "N/A");


        // The 'g' groups from OBJLoader are usually direct children of the 'object' passed to the load callback
        // So, the parent of the intersectedMesh should be our target group, if it has a name.
        if (intersectedMesh.parent && intersectedMesh.parent.isGroup && intersectedMesh.parent.name && intersectedMesh.parent.parent === loadedObjectModelRoot) {
            newlyClickedTargetGroup = intersectedMesh.parent;
        } else if (intersectedMesh.parent && intersectedMesh.parent.isGroup && intersectedMesh.parent.name && intersectedMesh.parent === loadedObjectModelRoot){
            // This case might happen if the OBJ file has only one 'g' group which becomes the loadedObjectModelRoot itself
             newlyClickedTargetGroup = intersectedMesh.parent;
        }
         else {
            // Fallback or more complex hierarchy: try to find named group upwards
            let current = intersectedMesh;
            while(current && current !== scene){
                if(current.isGroup && current.name){
                    newlyClickedTargetGroup = current;
                    break;
                }
                current = current.parent;
            }
        }


        if (newlyClickedTargetGroup) {
            console.log("Identified target group:", newlyClickedTargetGroup.name);
        } else {
            console.log("Could not identify a named target group for the clicked mesh.");
        }
    }

    removeAllHighlights();

    if (newlyClickedTargetGroup) {
        if (!selectedObjGroup || selectedObjGroup.uuid !== newlyClickedTargetGroup.uuid) {
            selectedObjGroup = newlyClickedTargetGroup;
            applyHighlightToMeshesInGroup(selectedObjGroup, highlightColorSingle);
        } else {
            selectedObjGroup = null;
        }
    } else {
        selectedObjGroup = null;
    }
    updateInfoPanel();
});

// --- Info Panel Update ---
function updateInfoPanel() {
    const infoPanel = document.getElementById('objectInfo');
    if (!infoPanel) return;

    if (selectedObjGroup) {
        const pos = selectedObjGroup.getWorldPosition(new THREE.Vector3());
        let rawName = selectedObjGroup.name || "Unnamed Group/Object";
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
