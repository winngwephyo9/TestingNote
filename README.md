// ... (namespace, use statements, and getProjectList function remain the same) ...

/**
 * MODIFIED FUNCTION: Finds the SINGLE .mtl file and ALL .obj files in a folder.
 */
public function getObjList(Request $request)
{
    $projectFolderId = $request->input('folderId');
    $accessToken = $request->input('accessToken');

    if (empty($projectFolderId) || empty($accessToken)) {
        return response()->json(['error' => 'Folder ID and Access Token are required.'], 400);
    }

    try {
        $client = new Client(['verify' => false]);
        $header = ["Authorization" => "Bearer " . $accessToken, "Accept" => "application/json"];

        $limit = 1000;
        $offset = 0;
        $totalCount = 0;
        $allFolderItems = [];

        do {
            $requestURL = "https://api.box.com/2.0/folders/" . $projectFolderId . "/items?fields=id,name&limit=" . $limit . "&offset=" . $offset;
            $response = $client->request('GET', $requestURL, ['headers' => $header]);
            $body = json_decode($response->getBody()->getContents());

            if ($offset === 0) $totalCount = $body->total_count;
            $allFolderItems = array_merge($allFolderItems, $body->entries);
            $offset += $limit;
        } while (count($allFolderItems) < $totalCount);

        $objFiles = [];
        $mtlFile = null;

        // First, find the single MTL file in the folder
        foreach ($allFolderItems as $item) {
            if ($item->type == "file" && strtolower(pathinfo($item->name, PATHINFO_EXTENSION)) === 'mtl') {
                $mtlFile = ['id' => $item->id, 'name' => $item->name];
                break; // Assume there's only one
            }
        }

        if ($mtlFile === null) {
            return response()->json(['error' => 'No MTL file found in the specified folder.'], 404);
        }

        // Now, collect all OBJ files
        foreach ($allFolderItems as $item) {
            if ($item->type == "file" && strtolower(pathinfo($item->name, PATHINFO_EXTENSION)) === 'obj') {
                $objFiles[] = ['id' => $item->id, 'name' => $item->name];
            }
        }

        return response()->json([
            'mtl' => $mtlFile,
            'objs' => $objFiles
        ]);

    } catch (Exception $e) {
        return response()->json(['error' => "Failed to read file list from Box: " . $e->getMessage()], 500);
    }
}


import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';
import { OBJLoader } from './library/loaders/OBJLoader.js';
import { MTLLoader } from './library/loaders/MTLLoader.js';

// --- Get UI Elements, Global variables, Scene, Camera, etc. ---
// (This entire top section remains IDENTICAL to the previous full version)

// ...

// --- REFACTORED Main Application Flow ---
async function loadModel(projectFolderId, projectName) {
    // 0. Reset scene and state (this part remains the same)
    if (loadedObjectModelRoot) scene.remove(loadedObjectModelRoot);
    loadedObjectModelRoot = null;
    selectedObjectOrGroup = null;
    originalMeshMaterials.clear();
    originalObjectPropertiesForIsolate.clear();
    isIsolateModeActive = false;
    elementIdDataMap.clear();
    modelTreeList.innerHTML = '';
    if (modelTreePanel) modelTreePanel.style.display = 'none';
    parsedWSCenID = "";
    parsedPJNo = "";
    updateInfoPanel();

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list for ${projectName}...`;
        
        // 1. Fetch the list of OBJ files and the single MTL file
        const fileData = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getObjList",
                data: { _token: CSRF_TOKEN, folderId: projectFolderId, accessToken: BOX_ACCESS_TOKEN },
                success: resolve,
                error: reject
            });
        });

        if (!fileData || !fileData.mtl || !fileData.objs || fileData.objs.length === 0) {
            throw new Error(`Incomplete file list received for project "${projectName}".`);
        }

        const mtlFileInfo = fileData.mtl;
        const objFileInfoList = fileData.objs;

        // 2. Load the SINGLE MTL file ONCE
        if (loaderTextElement) loaderTextElement.textContent = `Loading Materials...`;
        const mtlContent = await fetchBoxFileContent(mtlFileInfo.id);
        const mtlLoader = new MTLLoader();
        const materialsCreator = mtlLoader.parse(mtlContent, '');
        materialsCreator.preload();

        // 3. Download all OBJ file contents in parallel for performance
        let downloadedCount = 0;
        if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (0/${objFileInfoList.length})...`;
        
        const objContentPromises = objFileInfoList.map(objInfo => 
            fetchBoxFileContent(objInfo.id).then(content => {
                downloadedCount++;
                if (loaderTextElement) loaderTextElement.textContent = `Downloading Geometry (${downloadedCount}/${objFileInfoList.length})...`;
                // Return content paired with its original info for header parsing
                return { content: content, info: objInfo }; 
            })
        );
        const allObjContents = await Promise.all(objContentPromises);

        // 4. Parse the header from the FIRST downloaded OBJ file
        if(allObjContents.length > 0) {
            const headerData = await parseObjHeader(allObjContents[0].content);
            if (headerData) {
                parsedWSCenID = headerData.wscenId;
                parsedPJNo = headerData.pjNo;
            }
        }
        
        // 5. Parse all OBJ geometries using the SAME material creator
        if (loaderTextElement) loaderTextElement.textContent = `Processing geometry...`;
        const objLoader = new OBJLoader();
        objLoader.setMaterials(materialsCreator);
        
        const loadedObjects = allObjContents.map(objData => {
            return objLoader.parse(objData.content);
        });

        // 6. Combine all loaded objects into a single group
        const combinedModel = new THREE.Group();
        loadedObjects.forEach(object => {
            while (object.children.length > 0) {
                combinedModel.add(object.children[0]);
            }
        });
        loadedObjectModelRoot = combinedModel;

        // 7. Process the COMBINED model (center, scale, rotate)
        // ... (This logic remains the same as before) ...

        // 8. Fetch category data, build tree, add to scene, frame, and hide loader
        // ... (This logic remains the same as before) ...
        await fetchAllCategoryData(parsedWSCenID, /* ... */);
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        if (loaderContainer) loaderContainer.style.display = 'none';

    } catch (error) {
        console.error(`Failed to load model for ${projectName}:`, error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model: ${projectName}. Check console.`;
    }
}

// --- Start Application ---
// The rest of the file (populateProjectDropdown, animate, all helpers and event listeners)
// remains IDENTICAL to the previous full version.
populateProjectDropdown();
animate();
