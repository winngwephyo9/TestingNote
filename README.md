// --- Object Selection Logic (inside script.js) ---
let selectedObjGroup = null; // Will store the THREE.Group corresponding to a 'g' tag
const originalMeshMaterials = new Map(); // Stores original materials for individual MESHES
const highlightedMeshesUuids = new Set();    // Stores UUIDs of meshes that are currently highlighted

const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const highlightColorSingle = new THREE.Color(0xffff00);

// Applies highlight ONLY to meshes that are direct children or grandchildren (etc.)
// of the provided 'groupNode'
const applyHighlightToMeshesInGroup = (groupNode, color) => {
    if (!groupNode) return;
    groupNode.traverse((childMesh) => {
        if (childMesh.isMesh && childMesh.material) {
            // Ensure we are only acting on meshes truly within this groupNode's hierarchy
            let isDescendant = false;
            let ascendant = childMesh;
            while (ascendant) {
                if (ascendant === groupNode) {
                    isDescendant = true;
                    break;
                }
                if (ascendant === scene) break; // Stop if we reach the scene
                ascendant = ascendant.parent;
            }

            if (isDescendant) {
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
                    else if (childMesh.material.emissive) child.material.emissive.set(color);
                }
                highlightedMeshesUuids.add(childMesh.uuid);
            }
        }
    });
};

// Removes highlights from all meshes currently tracked
const removeAllHighlights = () => {
    highlightedMeshesUuids.forEach(meshUuid => {
        const mesh = scene.getObjectByProperty('uuid', meshUuid);
        if (mesh && mesh.isMesh && originalMeshMaterials.has(meshUuid)) {
            const originalMatSet = originalMeshMaterials.get(meshUuid);
            // Restore material
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
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(scene.children, true);

    let clickedMesh = null;
    if (intersects.length > 0) {
        clickedMesh = intersects[0].object;
    }

    let newSelectedGroup = null;
    if (clickedMesh && clickedMesh.isMesh) {
        // Try to find the named THREE.Group that corresponds to the OBJ 'g' tag
        let parent = clickedMesh.parent;
        while (parent && parent !== scene) {
            if (parent.isGroup && parent.name) { // Found a named group
                newSelectedGroup = parent;
                break;
            }
            parent = parent.parent;
        }
        // If no named group parent was found, and the mesh itself has a name,
        // it might be a simple object not from a 'g' tag.
        // For this specific problem, we are interested in 'g' groups.
        // If newSelectedGroup is still null here, it means we clicked a mesh not under a named group.
        // Or if the top-level loaded object is the one with the 'g' name.
        if (!newSelectedGroup && clickedMesh.parent && clickedMesh.parent.isGroup && clickedMesh.parent.name && clickedMesh.parent.parent === scene) {
             // Case where the OBJ's top-level object is the named group
            newSelectedGroup = clickedMesh.parent;
        } else if (!newSelectedGroup && intersects[0].object.name && intersects[0].object.isGroup) {
            // Case where the intersection itself is the named group.
            newSelectedGroup = intersects[0].object;
        }

    } else if (intersects.length > 0 && intersects[0].object.isGroup && intersects[0].object.name) {
        // Clicked directly on a named group (less likely as groups usually don't have geometry for raycasting)
        newSelectedGroup = intersects[0].object;
    }


    // Always remove previous highlights
    removeAllHighlights();

    if (newSelectedGroup) {
        if (!selectedObjGroup || selectedObjGroup.uuid !== newSelectedGroup.uuid) {
            // New group selected, or first selection
            selectedObjGroup = newSelectedGroup;
            applyHighlightToMeshesInGroup(selectedObjGroup, highlightColorSingle);
        } else {
            // Clicked the same group again, deselect it (highlights already removed)
            selectedObjGroup = null;
        }
    } else {
        // Clicked on empty space or an unidentifiable part
        selectedObjGroup = null;
    }

    updateInfoPanel(); // This function will now use selectedObjGroup
});


// --- Info Panel Update ---
// Modify updateInfoPanel to use selectedObjGroup
function updateInfoPanel() {
    const infoPanel = document.getElementById('objectInfo');
    if (!infoPanel) return;

    if (selectedObjGroup) { // Use selectedObjGroup
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
