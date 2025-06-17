// --- Object Selection Logic (inside script.js) ---
let selectedHierarchyObject = null; // The main selected object/group
let isShiftDown = false; // Kept for potential future multi-select
const originalMeshMaterials = new Map(); // Stores original materials for individual MESHES

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

const highlightColorSingle = new THREE.Color(0xffff00); // Yellow for single select

// Helper to apply highlight to MESHES within a given object/group
const applyHighlightToMeshes = (objectToHighlightRoot, color) => {
    if (!objectToHighlightRoot) return;
    objectToHighlightRoot.traverse((child) => {
        if (child.isMesh && child.material) {
            if (!originalMeshMaterials.has(child.uuid)) {
                // Store original material(s) for this specific mesh
                if (Array.isArray(child.material)) {
                    originalMeshMaterials.set(child.uuid, child.material.map(m => m.clone()));
                } else {
                    originalMeshMaterials.set(child.uuid, child.material.clone());
                }
            }

            // Apply highlight color
            if (Array.isArray(child.material)) {
                child.material.forEach(m => {
                    if (m.color) m.color.set(color);
                    else if (m.emissive) m.emissive.set(color);
                });
            } else {
                if (child.material.color) child.material.color.set(color);
                else if (child.material.emissive) child.material.emissive.set(color);
            }
        }
    });
};

// Helper to remove highlight from MESHES within a given object/group
const removeHighlightFromMeshes = (objectToUnhighlightRoot) => {
    if (!objectToUnhighlightRoot) return;
    objectToUnhighlightRoot.traverse((child) => {
        if (child.isMesh && child.material) {
            if (originalMeshMaterials.has(child.uuid)) {
                const originalMatSet = originalMeshMaterials.get(child.uuid);
                // Restore original material(s) for this specific mesh
                if (Array.isArray(child.material) && Array.isArray(originalMatSet)) {
                    child.material = originalMatSet.map(m => m.clone());
                } else if (!Array.isArray(child.material) && !Array.isArray(originalMatSet) && originalMatSet) {
                    child.material = originalMatSet.clone();
                }
                originalMeshMaterials.delete(child.uuid); // Clean up stored material for this mesh
            }
        }
    });
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

    let newlyClickedObject = null;

    if (intersects.length > 0) {
        let intersectTarget = intersects[0].object;
        // Traverse up to find the most logical "selectable" object.
        // This is often the direct child of the scene or a named group.
        while (intersectTarget.parent && intersectTarget.parent !== scene) {
            if (intersectTarget.name || intersectTarget.parent.type === 'Scene' || (intersectTarget.type === 'Group' && intersectTarget.children.some(c => c.isMesh))) {
                // Stop if current object has a name, or its parent is the scene,
                // or it's a group with mesh children (likely an OBJ group)
                break;
            }
            intersectTarget = intersectTarget.parent;
        }
        newlyClickedObject = intersectTarget;
    }

    // If an object was previously selected, remove its highlight.
    if (selectedHierarchyObject) {
        removeHighlightFromMeshes(selectedHierarchyObject);
    }
    // After removing highlights, always clear the map for a clean state if we're not multi-selecting.
    // For strict single select, we clear it regardless.
    originalMeshMaterials.clear();


    if (newlyClickedObject) {
        if (selectedHierarchyObject && selectedHierarchyObject.uuid === newlyClickedObject.uuid) {
            // Clicked the same object again: deselect it.
            selectedHierarchyObject = null;
            // originalMeshMaterials is already cleared above or will be empty.
        } else {
            // Clicked a new object.
            selectedHierarchyObject = newlyClickedObject;
            applyHighlightToMeshes(selectedHierarchyObject, highlightColorSingle);
        }
    } else {
        // Clicked on empty space: deselect any current object.
        selectedHierarchyObject = null;
        // originalMeshMaterials is already cleared above or will be empty.
    }

    updateInfoPanel();
});


// --- Info Panel Update ---
// (Keep your existing updateInfoPanel function, but it will now use selectedHierarchyObject)
function updateInfoPanel() {
    const infoPanel = document.getElementById('objectInfo');
    if (!infoPanel) return;

    if (selectedHierarchyObject) { // Changed from selectedObject
        const pos = selectedHierarchyObject.getWorldPosition(new THREE.Vector3());
        let rawName = selectedHierarchyObject.name || "Unnamed Group/Object";

        // Simplified naming for now, can be refined if OBJ structure is complex
        // if (selectedHierarchyObject.isMesh && selectedHierarchyObject.parent && selectedHierarchyObject.parent.name && selectedHierarchyObject.parent !== scene) {
        //      if (selectedHierarchyObject.parent.children.length > 1 || selectedHierarchyObject.parent.name.toLowerCase().includes("group")) {
        //         rawName = selectedHierarchyObject.parent.name;
        //     }
        // }

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
