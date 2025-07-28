async function loadModel(projectFolderId, projectName) {
    // 0. Reset scene and state
    if (loadedObjectModelRoot) scene.remove(loadedObjectModelRoot);
    loadedObjectModelRoot = null;
    selectedObjectOrGroup = null;
    originalMeshMaterials.clear();
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
    elementIdDataMap.clear();
    modelTreeList.innerHTML = '';
    if (modelTreePanel) modelTreePanel.style.display = 'none';
    updateInfoPanel();

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list for ${projectName}...`;
        
        const fileList = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getObjList",
                data: { _token: CSRF_TOKEN, folderId: projectFolderId, accessToken: BOX_ACCESS_TOKEN },
                success: resolve,
                error: reject
            });
        });

        if (!fileList || fileList.length === 0) {
            throw new Error(`No OBJ files found in project "${projectName}".`);
        }
        
        // --- MODIFIED LOADING LOGIC TO CATCH NaN ERRORS ---
        const loadedObjects = [];
        for (let i = 0; i < fileList.length; i++) {
            const filePair = fileList[i];
            if (loaderTextElement) {
                loaderTextElement.textContent = `Loading Geometry... (${i + 1}/${fileList.length})`;
            }

            try {
                let materialsCreator = null;
                const objLoader = new OBJLoader();

                if (filePair.mtl) {
                    const mtlContent = await fetchBoxFileContent(filePair.mtl.id);
                    const mtlLoader = new MTLLoader();
                    materialsCreator = mtlLoader.parse(mtlContent, '');
                    materialsCreator.preload();
                    objLoader.setMaterials(materialsCreator);
                }
                
                const objContent = await fetchBoxFileContent(filePair.obj.id);
                
                // This is the critical part. We wrap the parse in a try/catch.
                // The NaN error happens inside .parse()
                const object = objLoader.parse(objContent);
                loadedObjects.push(object);

            } catch (error) {
                console.error(`!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!`);
                console.error(`!!! CRITICAL ERROR parsing file: ${filePair.obj.name}`);
                console.error(`!!! This file is likely corrupted and caused the NaN error.`);
                console.error(`!!! Details:`, error);
                console.error(`!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!`);
                // We skip this file and continue loading the others.
            }
        }
        // --- END OF MODIFIED LOADING LOGIC ---

        if (loadedObjects.length === 0) {
            throw new Error("All file parts failed to load or were corrupted. Please check the console for errors.");
        }

        const combinedModel = new THREE.Group();
        loadedObjects.forEach(object => {
            while(object.children.length > 0) {
                combinedModel.add(object.children[0]);
            }
        });
        loadedObjectModelRoot = combinedModel;

        // ... (The rest of the function: processing, data fetching, tree building, etc., remains the same)

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error: ${error.message}`;
    }
}
