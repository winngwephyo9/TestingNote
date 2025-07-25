// In your FileController.php

public function getObjList(Request $request)
{
    $projectFolderId = $request->input('folderId');
    $accessToken = $request->input('accessToken');

    if (empty($projectFolderId) || empty($accessToken)) {
        return response()->json(['error' => 'Folder ID and Access Token are required.'], 400);
    }

    try {
        $client = new Client(['verify' => false]);
        $requestURL = "https://api.box.com/2.0/folders/" . $projectFolderId . "/items?fields=id,name";
        $header = [
            "Authorization" => "Bearer " . $accessToken,
            "Accept" => "application/json"
        ];
        
        $response = $client->request('GET', $requestURL, ['headers' => $header]);
        $items = json_decode($response->getBody()->getContents())->entries;
        
        $objFiles = [];
        $allFiles = [];
        foreach ($items as $item) {
            if ($item->type == "file") {
                $allFiles[$item->name] = $item->id;
            }
        }

        // --- MODIFIED LOGIC ---
        // Find all OBJ files and check if a corresponding MTL file exists.
        foreach ($allFiles as $fileName => $fileId) {
            if (strtolower(pathinfo($fileName, PATHINFO_EXTENSION)) === 'obj') {
                $mtlFileName = pathinfo($fileName, PATHINFO_FILENAME) . '.mtl';
                
                $mtlInfo = null; // Default to null
                if (array_key_exists($mtlFileName, $allFiles)) {
                    // If MTL file exists, create its info object
                    $mtlInfo = ['id' => $allFiles[$mtlFileName], 'name' => $mtlFileName];
                }

                $objFiles[] = [
                    'obj' => ['id' => $fileId, 'name' => $fileName],
                    'mtl' => $mtlInfo // This will be the mtl info object or null
                ];
            }
        }
        // --- END MODIFIED LOGIC ---

        return response()->json($objFiles);

    } catch (Exception $e) {
        return response()->json(['error' => "Failed to read OBJ list from Box: " . $e->getMessage()], 500);
    }
}





// In your script.js file, replace the loadModel function

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
            // This error now means no OBJ files were found at all, which is a valid error.
            throw new Error(`No OBJ files found in project "${projectName}".`);
        }
        
        // 2. Create an array of promises to load all models
        const loadingPromises = fileList.map((filePair, index) => {
            return new Promise(async (resolve, reject) => {
                try {
                    // --- MODIFIED LOGIC ---
                    let materialsCreator = null;
                    const objLoader = new OBJLoader();

                    // Step 2a: Load MTL only if it exists
                    if (filePair.mtl) {
                        const mtlContent = await fetchBoxFileContent(filePair.mtl.id);
                        const mtlLoader = new MTLLoader();
                        materialsCreator = mtlLoader.parse(mtlContent, '');
                        materialsCreator.preload();
                        objLoader.setMaterials(materialsCreator);
                    }
                    
                    // Step 2b: Load OBJ content
                    const objContent = await fetchBoxFileContent(filePair.obj.id);
                    
                    // Parse the object (it will use pre-set materials or a default)
                    const object = objLoader.parse(objContent);
                    
                    if (loaderTextElement) {
                         loaderTextElement.textContent = `Loading Geometry... (${index + 1}/${fileList.length})`;
                    }
                    
                    resolve(object);
                    // --- END MODIFIED LOGIC ---
                } catch (error) {
                    console.error(`Failed to load ${filePair.obj.name} from Box:`, error);
                    // Resolve with null so one failed file doesn't stop the whole process
                    resolve(null);
                }
            });
        });

        // 3. Wait for ALL models to load
        let loadedObjects = await Promise.all(loadingPromises);
        // Filter out any null results from failed loads
        loadedObjects = loadedObjects.filter(obj => obj !== null);

        if (loadedObjects.length === 0) {
            throw new Error("All file parts failed to load. Please check the console for errors.");
        }

        // 4. Combine all loaded objects into a single group
        const combinedModel = new THREE.Group();
        loadedObjects.forEach(object => {
            while(object.children.length > 0) {
                combinedModel.add(object.children[0]);
            }
        });
        loadedObjectModelRoot = combinedModel;

        // ... (The rest of the function: processing, data fetching, tree building, etc., remains the same)
        // 5. Process the COMBINED model
        // 6. Fetch category data
        // 7. Build the tree, add to scene, frame, and hide loader
        
        // ... (Paste the rest of the function from the previous full code response here)

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error: ${error.message}`;
    }
}
