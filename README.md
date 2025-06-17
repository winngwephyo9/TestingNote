// --- Object Selection Logic (inside script.js) ---
let selectedObject = null; // Changed from selectedObjects array to a single object
let isShiftDown = false; // Kept for potential future multi-select, but current logic focuses on single
const originalMaterials = new Map();

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

const highlightColorSingle = new THREE.Color(0xffff00); // Yellow for single select

// Helper to apply highlight to an object and its children
const applyHighlight = (objectToHighlight, color) => {
    objectToHighlight.traverse((child) => {
        if (child.isMesh && child.material) {
            if (!originalMaterials.has(child.uuid)) {
                if (Array.isArray(child.material)) {
                    originalMaterials.set(child.uuid, child.material.map(m => m.clone()));
                } else {
                    originalMaterials.set(child.uuid, child.material.clone());
                }
            }

            if (Array.isArray(child.material)) {
                child.material.forEach(m => {
                    if (m.color) m.color.set(color);
                    else if (m.emissive) m.emissive.set(color); // Fallback
                });
            } else {
                if (child.material.color) child.material.color.set(color);
                else if (child.material.emissive) child.material.emissive.set(color); // Fallback
            }
        }
    });
};

// Helper to remove highlight from an object and its children
const removeHighlight = (objectToUnhighlight) => {
    if (!objectToUnhighlight) return;
    objectToUnhighlight.traverse((child) => {
        if (child.isMesh && child.material) {
            if (originalMaterials.has(child.uuid)) {
                const originalMatSet = originalMaterials.get(child.uuid);
                if (Array.isArray(child.material) && Array.isArray(originalMatSet)) {
                    child.material = originalMatSet.map(m => m.clone());
                } else if (!Array.isArray(child.material) && !Array.isArray(originalMatSet) && originalMatSet) {
                    child.material = originalMatSet.clone();
                }
                originalMaterials.delete(child.uuid); // Remove after restoring
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
    const intersects = raycaster.intersectObjects(scene.children, true); // true for recursive

    let newlyClickedHierarchyObject = null;

    if (intersects.length > 0) {
        let intersectObject = intersects[0].object;
        // Traverse up to find a more meaningful object to select
        while (intersectObject.parent && intersectObject.parent !== scene && !intersectObject.name) {
            if (intersectObject.parent.type === "Group") {
                 // Check if this is likely the root of an OBJ loaded object
                if (intersectObject.parent.children.some(child => child.isMesh && child.geometry)) {
                    intersectObject = intersectObject.parent;
                    break;
                } else {
                    break;
                }
            } else {
                break;
            }
        }
        newlyClickedHierarchyObject = intersectObject;
    }

    // If there was a previously selected object, unhighlight it
    if (selectedObject) {
        removeHighlight(selectedObject);
        // originalMaterials will be cleared if a new object is selected, or if nothing is selected
    }

    if (newlyClickedHierarchyObject) {
        if (selectedObject && selectedObject.uuid === newlyClickedHierarchyObject.uuid) {
            // Clicked the same object again, so deselect it
            selectedObject = null;
            originalMaterials.clear(); // Clear as nothing is selected now
        } else {
            // Clicked a new object (or the first object)
            originalMaterials.clear(); // Clear previous state before highlighting new
            selectedObject = newlyClickedHierarchyObject;
            applyHighlight(selectedObject, highlightColorSingle);
        }
    } else {
        // Clicked on empty space
        selectedObject = null;
        originalMaterials.clear(); // Clear as nothing is selected now
    }

    updateInfoPanel();
});

// --- Info Panel Update ---
function updateInfoPanel() {
    const infoPanel = document.getElementById('objectInfo');
    if (!infoPanel) return;

    if (selectedObject) {
        const pos = selectedObject.getWorldPosition(new THREE.Vector3());
        let rawName = selectedObject.name || "Unnamed Group/Object";

        // If the selected object is a mesh and its parent has a name (likely an OBJ group),
        // prefer the parent's name for grouping context.
        // This part might need adjustment based on how deep your OBJs nest meaningful names.
        if (selectedObject.isMesh && selectedObject.parent && selectedObject.parent.name && selectedObject.parent !== scene) {
             if (selectedObject.parent.children.length > 1 || selectedObject.parent.name.toLowerCase().includes("group")) {
                rawName = selectedObject.parent.name; // Use group name if mesh is part of a larger named group
            }
        }


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
