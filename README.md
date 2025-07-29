// --- REFACTORED Main Application Flow with Download Throttling ---
async function loadModel(projectFolderId, projectName) {
    // 0. Reset scene and state (this part remains the same)
    if (loadedObjectModelRoot) scene.remove(loadedObjectModelRoot);
    // ... (full reset logic) ...
    updateInfoPanel();

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list for ${projectName}...`;
        
        // 1. Fetch the list of OBJ/MTL files
        const fileData = await new Promise((resolve, reject) => { /* ... (ajax call as before) ... */ });
        if (!fileData || !fileData.mtl || !fileData.objs || fileData.objs.length === 0) {
            throw new Error(`Incomplete file list for project "${projectName}".`);
        }

        const mtlFileInfo = fileData.mtl;
        const objFileInfoList = fileData.objs;

        // 2. Load the SINGLE MTL file ONCE
        if (loaderTextElement) loaderTextElement.textContent = `Loading Materials...`;
        const mtlContent = await fetchBoxFileContent(mtlFileInfo.id);
        const mtlLoader = new MTLLoader();
        const materialsCreator = mtlLoader.parse(mtlContent, '');
        materialsCreator.preload();

        // 3. **PERFORMANCE & RATE-LIMIT FIX**: Download all OBJ file contents in controlled parallel batches.
        if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (0/${objFileInfoList.length})...`;
        
        const CONCURRENT_DOWNLOADS = 10; // Number of parallel downloads. Box recommends between 4 and 12.
        const allObjContents = [];
        let downloadedCount = 0;
        
        // Create a copy of the list to use as a queue
        const downloadQueue = [...objFileInfoList];

        const downloadBatch = async () => {
            const promises = [];
            // Start up to CONCURRENT_DOWNLOADS downloads
            while(downloadQueue.length > 0 && promises.length < CONCURRENT_DOWNLOADS) {
                const objInfo = downloadQueue.shift(); // Get next file from queue
                if (objInfo) {
                    const promise = fetchBoxFileContent(objInfo.id).then(content => {
                        downloadedCount++;
                        if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (${downloadedCount}/${objFileInfoList.length})...`;
                        return { content: content, info: objInfo };
                    });
                    promises.push(promise);
                }
            }
            // Wait for this batch of promises to complete
            const results = await Promise.all(promises);
            allObjContents.push(...results); // Add results to the main array

            // If there are more files in the queue, recurse to download the next batch
            if(downloadQueue.length > 0) {
                await downloadBatch();
            }
        };

        await downloadBatch(); // Start the first batch

        // 4. Parse the header from the FIRST downloaded OBJ file
        // ... (This logic remains the same) ...
        
        // 5. Parse all OBJ geometries using the SAME material creator
        // ... (This logic remains the same) ...

        // 6. Combine all loaded objects into a single group
        // ... (This logic remains the same) ...

        // 7. Process the COMBINED model (center, scale, rotate)
        // ... (This logic remains the same) ...

        // 8. Fetch category data, build tree, add to scene, frame, and hide loader
        // ... (This logic remains the same) ...

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model: ${projectName}. Check console.`;
    }
}
