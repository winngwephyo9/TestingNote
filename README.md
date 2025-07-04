function zoomToAndIsolate(targetObject) {
    if (!targetObject) return;
    deIsolateAllObjects();
    isIsolateModeActive = true;

    const box = new THREE.Box3().setFromObject(targetObject);
    if (box.isEmpty()) { isIsolateModeActive = false; return; }
    
    // ... (Camera positioning logic remains the same) ...
    const center = box.getCenter(new THREE.Vector3());
    const sphere = box.getBoundingSphere(new THREE.Sphere());
    const radius = sphere.radius;
    const fovInRadians = THREE.MathUtils.degToRad(camera.fov);
    const distance = radius / Math.sin(fovInRadians / 2);
    const finalDistance = distance * 1.5; // Zoom factor
    const offsetDirection = camera.position.clone().sub(controls.target).normalize();
    if (offsetDirection.lengthSq() === 0) offsetDirection.set(0.5, 0.5, 1).normalize();
    camera.position.copy(center).addScaledVector(offsetDirection, finalDistance);
    controls.target.copy(center);
    controls.update();


    loadedObjectModelRoot.traverse((object) => {
        if (object.isMesh) {
            let isPartOfSelectedTarget = false;
            let temp = object;
            while (temp) {
                if (temp === targetObject) {
                    isPartOfSelectedTarget = true;
                    break;
                }
                temp = temp.parent;
            }

            if (!isPartOfSelectedTarget) {
                if (!originalObjectPropertiesForIsolate.has(object.uuid)) {
                    originalObjectPropertiesForIsolate.set(object.uuid, {
                        material: object.material,
                        visible: object.visible
                    });
                }
                if (object.visible) {
                    const materials = Array.isArray(object.material) ? object.material : [object.material];
                    const newMaterials = materials.map(mat => {
                        const newMat = mat.clone();
                        newMat.transparent = true;
                        newMat.opacity = 0.1; // Make other objects very transparent
                        return newMat;
                    });
                    object.material = Array.isArray(object.material) ? newMaterials : newMaterials[0];
                }
            } else {
                // This part ensures the selected object is fully opaque, even if its original MTL material was transparent
                const materials = Array.isArray(object.material) ? object.material : [object.material];
                materials.forEach(mat => {
                    // We don't restore from originalObjectPropertiesForIsolate because it might have been
                    // made transparent in a previous isolation. We just force it to be opaque.
                    // The 'removeAllHighlights' and 'applyHighlight' will handle its actual material.
                    if (originalMeshMaterials.has(object.uuid)) {
                        // If it's highlighted, it already has a cloned, opaque material. Do nothing.
                    } else {
                        // If not highlighted, ensure its base material is opaque.
                        mat.transparent = false; // Or restore its original transparency if needed
                        mat.opacity = 1.0;
                    }
                });
                object.visible = true;
            }
        }
    });
}

function deIsolateAllObjects() {
    if (!isIsolateModeActive) return;
    originalObjectPropertiesForIsolate.forEach((props, uuid) => {
        const object = scene.getObjectByProperty('uuid', uuid);
        if (object && object.isMesh) {
            // Restore the original material and visibility
            object.material = props.material;
            object.visible = props.visible;
        }
    });
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
}
