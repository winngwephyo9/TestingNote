<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Exception;
use GuzzleHttp\Client;
use Illuminate\Support\Facades\Log;

class FileController extends Controller
{
    // getProjectList function remains the same...

    /**
     * MODIFIED FUNCTION: Lists ALL OBJ/MTL files within a specific project folder on Box,
     * handling pagination automatically.
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
            $header = [
                "Authorization" => "Bearer " . $accessToken,
                "Accept" => "application/json"
            ];

            $limit = 1000; // Max items per API call
            $offset = 0;
            $totalCount = 0;
            $allFiles = [];

            do {
                $requestURL = "https://api.box.com/2.0/folders/" . $projectFolderId . "/items?fields=id,name&limit=" . $limit . "&offset=" . $offset;
                
                $response = $client->request('GET', $requestURL, ['headers' => $header]);
                $body = json_decode($response->getBody()->getContents());

                if ($offset === 0) {
                    $totalCount = $body->total_count;
                }

                foreach ($body->entries as $item) {
                    if ($item->type == "file") {
                        $allFiles[$item->name] = $item->id;
                    }
                }

                $offset += $limit;

            } while (count($allFiles) < $totalCount);

            
            $objFiles = [];
            foreach ($allFiles as $fileName => $fileId) {
                if (strtolower(pathinfo($fileName, PATHINFO_EXTENSION)) === 'obj') {
                    $mtlFileName = pathinfo($fileName, PATHINFO_FILENAME) . '.mtl';
                    if (array_key_exists($mtlFileName, $allFiles)) {
                        $objFiles[] = [
                            'obj' => ['id' => $fileId, 'name' => $fileName],
                            'mtl' => ['id' => $allFiles[$mtlFileName], 'name' => $mtlFileName]
                        ];
                    } else {
                        Log::warning("MTL file not found for: " . $fileName);
                    }
                }
            }

            return response()->json($objFiles);

        } catch (Exception $e) {
            return response()->json(['error' => "Failed to read OBJ list from Box: " . $e->getMessage()], 500);
        }
    }
}



import * as THREE from './library/three.module.js';
import { OrbitControls } from './library/controls/OrbitControls.js';
import { OBJLoader } from './library/loaders/OBJLoader.js';
import { MTLLoader } from './library/loaders/MTLLoader.js';

// --- Get UI Elements & Global variables ---
// ... (This section remains the same) ...

// --- Scene Setup & Helpers ---
// ... (This section remains the same, no changes needed for Scene, Camera, Renderer,
//      frameObject, buildAndPopulateCategorizedTree, createCategoryNode, createObjectNode,
//      highlighting, isolation, updateInfoPanel, etc.) ...


// --- Data Fetching Functions (Modified for Performance) ---

async function parseObjHeader(objContent) {
    // This function now only parses and returns the values, not setting globals.
    try {
        const lines = objContent.split(/\r?\n/);
        if (lines.length > 0) {
            const firstLine = lines[0].trim();
            if (firstLine.startsWith("# ")) {
                const content = firstLine.substring(2).trim();
                const pattern1Match = content.match(/^([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})_([a-zA-Z0-9]+)$/);
                if (pattern1Match) {
                    return { wscenId: pattern1Match[1], pjNo: pattern1Match[2] };
                }
                if (content.includes("ワークシェアリングされてない")) {
                    return { wscenId: "", pjNo: "" };
                }
            }
        }
    } catch (error) {
        console.error("Error parsing OBJ header:", error);
    }
    return { wscenId: null, pjNo: null }; // Return nulls if no match
}


async function fetchBoxFileContent(fileId) {
    // ... (This function remains the same)
}

async function fetchAllCategoryData(wscenId, allElementIds) {
    // ... (This function remains the same)
}


// --- REFACTORED Main Application Flow ---
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
    parsedWSCenID = ""; // Reset header info
    parsedPJNo = "";    // Reset header info
    updateInfoPanel();

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list for ${projectName}...`;
        
        // 1. Fetch the list of OBJ/MTL file pairs from the server
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
            throw new Error(`No valid OBJ/MTL pairs found in project "${projectName}".`);
        }

        // 2. **PERFORMANCE FIX**: Create an array of promises to download all file contents in parallel.
        if (loaderTextElement) loaderTextElement.textContent = `Downloading 3D files (0/${fileList.length * 2})...`;
        let downloadedCount = 0;
        
        const fileContentPromises = fileList.map(filePair => {
            const mtlPromise = fetchBoxFileContent(filePair.mtl.id).then(content => {
                downloadedCount++;
                if (loaderTextElement) loaderTextElement.textContent = `Downloading 3D files (${downloadedCount}/${fileList.length * 2})...`;
                return { type: 'mtl', content, name: filePair.mtl.name };
            });
            const objPromise = fetchBoxFileContent(filePair.obj.id).then(content => {
                downloadedCount++;
                if (loaderTextElement) loaderTextElement.textContent = `Downloading 3D files (${downloadedCount}/${fileList.length * 2})...`;
                return { type: 'obj', content, name: filePair.obj.name };
            });
            return Promise.all([mtlPromise, objPromise]);
        });
        
        const allFilePairsContent = await Promise.all(fileContentPromises);

        // 3. **HEADER FIX**: Parse the header from the FIRST downloaded OBJ file only.
        if(allFilePairsContent.length > 0) {
            const firstObjContent = allFilePairsContent[0].find(f => f.type === 'obj').content;
            const headerData = await parseObjHeader(firstObjContent);
            if (headerData) {
                parsedWSCenID = headerData.wscenId;
                parsedPJNo = headerData.pjNo;
            }
        }
        
        // 4. Parse all models from the downloaded content.
        if (loaderTextElement) loaderTextElement.textContent = `Processing geometry...`;
        const mtlLoader = new MTLLoader();
        const objLoader = new OBJLoader();
        const loadedObjects = allFilePairsContent.map(pair => {
            const mtlContent = pair.find(f => f.type === 'mtl').content;
            const objContent = pair.find(f => f.type === 'obj').content;
            
            const materialsCreator = mtlLoader.parse(mtlContent, '');
            materialsCreator.preload();
            
            objLoader.setMaterials(materialsCreator);
            return objLoader.parse(objContent);
        });

        // 5. Combine all loaded objects into a single group
        const combinedModel = new THREE.Group();
        loadedObjects.forEach(object => {
            while (object.children.length > 0) {
                combinedModel.add(object.children[0]);
            }
        });
        loadedObjectModelRoot = combinedModel;

        // 6. Process the COMBINED model (center, scale, rotate)
        // ... (This logic remains the same as before) ...

        // 7. Fetch category data for all parts
        const allIds = [];
        loadedObjectModelRoot.traverse(child => {
            if (child.name) {
                const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                if (splitIndex > 0) allIds.push(child.name.substring(splitIndex + 1));
            }
        });
        await fetchAllCategoryData(parsedWSCenID, [...new Set(allIds)]);
        
        // 8. Build the tree, add to scene, frame, and hide loader
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        if (loaderContainer) loaderContainer.style.display = 'none';

    } catch (error) {
        console.error(`Failed to load model for project ID ${projectFolderId}:`, error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model: ${projectName}. Check console.`;
    }
}


// --- Populates the dropdown and starts the initial load ---
async function populateProjectDropdown() {
    try {
        const projects = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getProjectList",
                data: { _token: CSRF_TOKEN, folderId: BOX_MAIN_FOLDER_ID, accessToken: BOX_ACCESS_TOKEN },
                success: resolve,
                error: reject
            });
        });

        modelSelector.innerHTML = '';
        if (projects && projects.length > 0) {
            projects.forEach(project => {
                const option = document.createElement('option');
                option.value = project.id;
                option.textContent = project.name;
                modelSelector.appendChild(option);
            });
            loadModel(projects[0].id, projects[0].name);
        } else {
            if (loaderTextElement) loaderTextElement.textContent = "No projects found in the main folder.";
        }
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
        if (loaderTextElement) loaderTextElement.textContent = "Error fetching project list from Box.";
    }
}

// --- Event Listeners and Animation Loop ---
// ... (Keep all existing event listeners and the animate function) ...

// --- Start Application ---
populateProjectDropdown();
animate();
