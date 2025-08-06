// --- Event Listeners and Animation Loop ---

// REVISED: Listen for clicks only within the 3D viewer canvas.
viewerContainer.addEventListener('click', (event) => {
    // Exit if the 3D model isn't loaded yet.
    if (!loadedObjectModelRoot) return;

    // Get the position and size of the canvas container on the page.
    const rect = viewerContainer.getBoundingClientRect();

    // Calculate the mouse position relative to the container, normalized to [-1, 1].
    // This makes the raycasting accurate within the canvas bounds.
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

    // Update the raycaster with the new, accurate mouse position and camera.
    raycaster.setFromCamera(mouse, camera);

    // Find all objects intersected by the ray.
    const intersects = raycaster.intersectObjects(loadedObjectModelRoot.children, true);

    let newlyClickedTarget = null;
    // If the ray hit something...
    if (intersects.length > 0) {
        // ...get the first object hit.
        let current = intersects[0].object;
        // Walk up the object's hierarchy until we find the main group/object
        // that is a direct child of the loaded model root. This ensures you select
        // the whole component (e.g., "Beam_12345") instead of a tiny piece of its geometry.
        while (current && current.parent !== loadedObjectModelRoot && current !== loadedObjectModelRoot && current.parent !== scene) {
            current = current.parent;
        }
        // If we found a valid top-level object, it's our target.
        if (current) newlyClickedTarget = current;
    }

    // Pass the found target (or null if nothing was clicked) to the selection handler.
    handleSelection(newlyClickedTarget);
});


if (closeModelTreeBtn) closeModelTreeBtn.addEventListener('click', () => { if (modelTreePanel) modelTreePanel.style.display = 'none'; });

// ... (The rest of your code remains the same)
