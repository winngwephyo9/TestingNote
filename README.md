![image](https://github.com/user-attachments/assets/b616e460-21bd-4079-ada7-82bc9b0f9f2b)
![image](https://github.com/user-attachments/assets/3d3db647-df3f-42d4-9b8a-6494b692dece)

// --- Main Application Flow (NEW - Using explicit MTLLoader) ---
async function main() {
    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        
        // Define paths and filenames
        const assetPath = '/ccc/public/objFiles/';
        const objFileName = '240324_GF本社移転_2022_20250618.obj';
        const mtlFileName = '240324_GF本社移転_2022_20250627.mtl'; // Manually specify the MTL filename
        const fullObjPath = assetPath + objFileName;
        
        // Step 1: Parse header info from the OBJ file
        await parseObjHeader(fullObjPath);

        // Step 2: Manually load the materials first using MTLLoader
        if (loaderTextElement) loaderTextElement.textContent = `Loading Materials...`;
        
        const materialsCreator = await new Promise((resolve, reject) => {
            const mtlLoader = new MTLLoader();
            mtlLoader.setPath(assetPath); // Set base path for any textures inside the MTL
            mtlLoader.load(mtlFileName, (materials) => {
                materials.preload();
                resolve(materials);
            }, undefined, reject);
        });

        console.log("Materials loaded successfully:", materialsCreator);
        
        // Step 3: Load the OBJ model and provide the pre-loaded materials
        if (loaderTextElement) loaderTextElement.textContent = `Loading 3D Geometry...`;

        const object = await new Promise((resolve, reject) => {
            const objLoader = new OBJLoader();
            objLoader.setMaterials(materialsCreator); // <-- CRUCIAL STEP
            objLoader.load(fullObjPath, resolve, (xhr) => {
                if (loaderTextElement) {
                    const percent = Math.round(xhr.loaded / xhr.total * 100);
                    loaderTextElement.textContent = isFinite(percent) && percent < 100 ?
                        `Loading 3D Geometry: ${percent}%` : `Processing Geometry...`;
                }
            }, reject);
        });

        loadedObjectModelRoot = object;
        
        // Step 4: Process the loaded model (center, scale, rotate)
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
        const maxDim = Math.max(scaledBox.getSize(new THREE.Vector3()).x, scaledBox.getSize(new THREE.Vector3()).y, scaledBox.getSize(new THREE.Vector3()).z);
        const desiredMaxDimension = 150;
        if (maxDim > 0) {
            const scale = desiredMaxDimension / maxDim;
            object.scale.set(scale, scale, scale);
        }
        object.rotation.x = -Math.PI / 2;
        object.rotation.y = -Math.PI / 2;

        // Step 5: Fetch category data and build the tree
        const allIds = [];
        loadedObjectModelRoot.traverse(child => {
            if (child.name && child.parent === loadedObjectModelRoot) {
                const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                if (splitIndex > 0) allIds.push(child.name.substring(splitIndex + 1));
            }
        });
        await fetchAllCategoryData(parsedWSCenID, [...new Set(allIds)]);
        await buildAndPopulateCategorizedTree();
        
        // Step 6: Add model to scene, frame it, and hide loader
        scene.add(object);
        frameObject(object);
        
        if (loaderContainer) loaderContainer.style.display = 'none';

    } catch (error) {
        console.error('Failed to initialize the viewer:', error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = 'Error during initialization.';
    }
}

