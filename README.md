// ... (Imports and UI Element selection) ...

// --- Global variables ---
// ... (keep parsedWSCenID, parsedPJNo, etc.) ...
const ghostedObjects = new Map(); // NEW: To store original materials of ghosted objects

// --- NEW: Ghost Material Definition ---
const ghostMaterial = new THREE.MeshBasicMaterial({
    color: 0xffffff,
    transparent: true,
    opacity: 0.15, // 白抜きの濃さ (0.1 ~ 0.3程度がおすすめ)
    depthWrite: false, // 他のオブジェクトの裏にあっても見えるようにする
});


// --- Scene Setup, Camera, Renderer, Lighting, Controls ---
// ... (This section remains unchanged) ...

// --- Helper Functions ---
// ... (parseObjHeader, fetchAllCategoryData, frameObject, etc. remain unchanged) ...

// --- Model Tree Logic (MODIFIED) ---

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

    const visibilityToggle = document.createElement('span');
    visibilityToggle.className = 'visibility-toggle visible-icon';
    visibilityToggle.title = 'Hide';
    itemContent.appendChild(visibilityToggle);

    const subList = document.createElement('ul');
    toggler.addEventListener('click', (e) => {
        e.stopPropagation();
        const isCollapsed = subList.style.display === 'none';
        subList.style.display = isCollapsed ? 'block' : 'none';
        toggler.textContent = isCollapsed ? '▼' : '▶';
    });
    
    // Visibility toggle for the ENTIRE CATEGORY
    visibilityToggle.addEventListener('click', (e) => {
        e.stopPropagation();
        // Check if the first object in the category is ghosted to determine the state
        const isCurrentlyGhosted = ghostedObjects.has(objectsInCategory[0].uuid);
        objectsInCategory.forEach(object => {
            if (isCurrentlyGhosted) {
                setObjectVisibility(object, true); // Show all
            } else {
                setObjectVisibility(object, false); // Ghost all
            }
        });
        // Update the category icon based on the new state
        const isNowGhosted = ghostedObjects.has(objectsInCategory[0].uuid);
        visibilityToggle.classList.toggle('visible-icon', !isNowGhosted);
        visibilityToggle.classList.toggle('hidden-icon', isNowGhosted);
        visibilityToggle.title = isNowGhosted ? 'Show' : 'Hide';
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
    visibilityToggle.className = 'visibility-toggle';
    // Initial state check
    const isGhosted = ghostedObjects.has(object.uuid);
    visibilityToggle.classList.add(isGhosted ? 'hidden-icon' : 'visible-icon');
    visibilityToggle.title = isGhosted ? 'Show' : 'Hide';
    itemContent.appendChild(visibilityToggle);
    listItem.appendChild(itemContent);
    parentULElement.appendChild(listItem);
    itemContent.addEventListener('click', () => handleSelection(object));
    
    // MODIFIED Visibility Logic
    visibilityToggle.addEventListener('click', (e) => {
        e.stopPropagation();
        const isCurrentlyGhosted = ghostedObjects.has(object.uuid);
        setObjectVisibility(object, isCurrentlyGhosted); // Toggle visibility
        
        // Update UI
        const isNowGhosted = ghostedObjects.has(object.uuid);
        visibilityToggle.classList.toggle('visible-icon', !isNowGhosted);
        visibilityToggle.classList.toggle('hidden-icon', isNowGhosted);
        visibilityToggle.title = isNowGhosted ? 'Show' : 'Hide';
    });
}

// --- NEW Function to handle visibility (ghosting) ---
function setObjectVisibility(targetObject, makeVisible) {
    if (makeVisible) {
        // Restore original material if it was ghosted
        if (ghostedObjects.has(targetObject.uuid)) {
            const originalMaterial = ghostedObjects.get(targetObject.uuid);
            targetObject.traverse(child => {
                if (child.isMesh) child.material = originalMaterial;
            });
            ghostedObjects.delete(targetObject.uuid);
        }
        targetObject.visible = true; // Ensure it's visible
    } else {
        // Ghost the object if it's not already
        if (!ghostedObjects.has(targetObject.uuid)) {
            // We assume for simplicity that all meshes in a group share the same material.
            // A more complex model might require storing material per mesh.
            let originalMaterial = null;
            targetObject.traverse(child => {
                if (child.isMesh) {
                    if (!originalMaterial) originalMaterial = child.material;
                    child.material = ghostMaterial;
                }
            });
            if (originalMaterial) {
                ghostedObjects.set(targetObject.uuid, originalMaterial);
            }
        }
        targetObject.visible = true; // Keep it visible, but with ghost material
    }
}


// --- Main Application Flow and other functions ---
// The rest of your script (main, animate, selection logic, etc.) remains the same.
// Just ensure you have the full code from the previous response for all other functions.
async function main() { /* ... */ }
function handleSelection(target) { /* ... */ }
function updateInfoPanel() { /* ... */ }
// etc.

// --- Start Application ---
main();
animate();
