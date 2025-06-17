// --- Object Selection Logic (inside script.js) ---
let selectedHierarchyObject = null; // The main selected object/group
// let isShiftDown = false; // Not strictly needed for current single-select logic
const originalMeshMaterials = new Map(); // Stores original materials for individual MESHES that are CURRENTLY highlighted
const highlightedMeshes = new Set();    // Stores UUIDs of meshes that are currently highlighted

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

const highlightColorSingle = new THREE.Color(0xffff00); // Yellow for single select

// Helper to apply highlight to MESHES within a given object/group
const applyHighlightToMeshesOfObject = (objectToHighlightRoot, color) => {
    if (!objectToHighlightRoot) return;
    objectToHighlightRoot.traverse((childMesh) => {
        if (childMesh.isMesh && childMesh.material) {
            // Only store and highlight if not already highlighted (shouldn't happen if logic is correct, but good check)
            if (!highlightedMeshes.has(childMesh.uuid)) {
                // Store original material(s) for this specific mesh
                if (Array.isArray(childMesh.material)) {
                    originalMeshMaterials.set(childMesh.uuid, childMesh.material.map(m => m.clone()));
                } else {
                    originalMeshMaterials.set(childMesh.uuid, childMesh.material.clone());
                }

                // Apply highlight color
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

// Helper to remove highlight from ALL currently highlighted meshes
const removeAllHighlights = () => {
    highlightedMeshes.forEach(meshUuid => {
        const mesh = scene.getObjectByProperty('uuid', meshUuid); // Find the mesh in the scene
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


// window.addEventListener('keydown', (event) => { // Not needed for pure single select
//     if (event.key === 'Shift') isShiftDown = true;
// });
// window.addEventListener('keyup', (event) => { // Not needed for pure single select
//     if (event.key === 'Shift') isShiftDown = false;
// });

window.addEventListener('click', (event) => {
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(scene.children, true);

    let newlyClickedObjectRoot = null;

    if (intersects.length > 0) {
        let intersectTarget = intersects[0].object;
        // Traverse up to find the most logical "selectable" root object (often the Group loaded by OBJLoader).
        while (intersectTarget.parent && intersectTarget.parent !== scene) {
            // Heuristic: if a parent is a Group and has a name, or if it's a direct child of the scene,
            // it's a good candidate for the "root" of the selection.
            if (intersectTarget.parent.type === 'Group' && intersectTarget.parent.name) {
                intersectTarget = intersectTarget.parent;
                break;
            }
            if (intersectTarget.parent === scene && (intersectTarget.name || intersectTarget.type === 'Group')) {
                 // If the direct child of the scene is a group or has a name, use it.
                break;
            }
             // If the object itself has a name and no named group parent, select it directly.
            if (intersectTarget.name && (!intersectTarget.parent.name || intersectTarget.parent === scene)) {
                break;
            }
            intersectTarget = intersectTarget.parent;
        }
        newlyClickedObjectRoot = intersectTarget;
    }

    // Always remove all existing highlights before processing the new click
    removeAllHighlights();

    if (newlyClickedObjectRoot) {
        // If the newly clicked object is different from the previously selected one,
        // or if nothing was selected, then select and highlight the new one.
        if (!selectedHierarchyObject || selectedHierarchyObject.uuid !== newlyClickedObjectRoot.uuid) {
            selectedHierarchyObject = newlyClickedObjectRoot;
            applyHighlightToMeshesOfObject(selectedHierarchyObject, highlightColorSingle);
        } else {
            // Clicked the same object again, so it's now deselected (highlights already removed)
            selectedHierarchyObject = null;
        }
    } else {
        // Clicked on empty space, deselect.
        selectedHierarchyObject = null;
    }

    updateInfoPanel();
});


// --- Info Panel Update ---
// (Your existing updateInfoPanel function should work with selectedHierarchyObject)
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
