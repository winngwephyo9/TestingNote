  <?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Exception;
use GuzzleHttp\Client;

class FileController extends Controller
{
    /**
     * NEW FUNCTION: Lists the sub-folders (projects) within a main Box folder.
     * This will populate the dropdown menu.
     */
    public function getProjectList(Request $request)
    {
        $mainFolderId = $request->input('folderId'); // The ID of your "MainFolder"
        $accessToken = $request->input('accessToken'); // You need to provide the Box access token

        if (empty($mainFolderId) || empty($accessToken)) {
            return response()->json(['error' => 'Folder ID and Access Token are required.'], 400);
        }

        try {
            $client = new Client(['verify' => false]); // Consider setting 'verify' to true in production with proper SSL certs
            $requestURL = "https://api.box.com/2.0/folders/" . $mainFolderId . "/items?fields=id,name";
            $header = [
                "Authorization" => "Bearer " . $accessToken,
                "Accept" => "application/json"
            ];

            $response = $client->request('GET', $requestURL, ['headers' => $header]);
            $items = json_decode($response->getBody()->getContents())->entries;
            
            $projects = [];
            foreach ($items as $item) {
                if ($item->type == "folder") {
                    $projects[] = [
                        'id' => $item->id,       // The Folder ID (e.g., "12345" for "GF")
                        'name' => $item->name,   // The Folder Name (e.g., "GF")
                    ];
                }
            }

            return response()->json($projects);

        } catch (Exception $e) {
            return response()->json(['error' => "Failed to read project list from Box: " . $e->getMessage()], 500);
        }
    }

    /**
     * MODIFIED FUNCTION: Lists OBJ/MTL files within a specific project folder on Box.
     */
    public function getObjList(Request $request)
    {
        $projectFolderId = $request->input('folderId'); // The ID of the selected project folder (e.g., "GF")
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

            // Find OBJ files and look for their corresponding MTL files
            foreach ($allFiles as $fileName => $fileId) {
                if (strtolower(pathinfo($fileName, PATHINFO_EXTENSION)) === 'obj') {
                    $mtlFileName = pathinfo($fileName, PATHINFO_FILENAME) . '.mtl';
                    if (array_key_exists($mtlFileName, $allFiles)) {
                        $objFiles[] = [
                            'obj' => ['id' => $fileId, 'name' => $fileName],
                            'mtl' => ['id' => $allFiles[$mtlFileName], 'name' => $mtlFileName]
                        ];
                    }
                }
            }

            return response()->json($objFiles);

        } catch (Exception $e) {
            return response()->json(['error' => "Failed to read OBJ list from Box: " . $e->getMessage()], 500);
        }
    }
}



// ... (Imports, UI Elements, Global variables, Scene, Camera, etc. remain the same) ...

// --- Model Configuration ---
// This is no longer needed, as we will fetch the list dynamically
// const models = { ... };

// --- NEW: Global variables for Box integration ---
const BOX_ACCESS_TOKEN = "YOUR_DEVELOPER_TOKEN_HERE"; // IMPORTANT: Replace with a real token
const BOX_MAIN_FOLDER_ID = "YOUR_MAIN_FOLDER_ID_HERE"; // IMPORTANT: Replace with the ID of "MainFolder"

// --- Helper Functions ---
// ... (parseObjHeader, frameObject, buildAndPopulateCategorizedTree, etc. are OK) ...

// --- Data Fetching Functions (Modified for Box) ---

// Fetches a file's content directly from Box using its file ID
async function fetchBoxFileContent(fileId) {
    const url = `https://api.box.com/2.0/files/${fileId}/content`;
    const headers = { 'Authorization': `Bearer ${BOX_ACCESS_TOKEN}` };
    const response = await fetch(url, { headers });
    if (!response.ok) {
        throw new Error(`Failed to fetch file ${fileId} from Box: ${response.statusText}`);
    }
    return await response.text();
}

// Fetches category data (this function doesn't change, it still talks to your server)
async function fetchAllCategoryData(wscenId, allElementIds) { /* ... keep this function as is ... */ }

// --- REFACTORED Main Application Flow ---
async function loadModel(projectFolderId) {
    // 0. Reset scene and state
    if (loadedObjectModelRoot) scene.remove(loadedObjectModelRoot);
    loadedObjectModelRoot = null;
    // ... (clear all other maps and UI state) ...
    modelTreeList.innerHTML = '';
    if (modelTreePanel) modelTreePanel.style.display = 'none';
    updateInfoPanel();

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        
        // 1. Fetch the list of OBJ/MTL file pairs from the server (which gets it from Box)
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list...`;
        const fileList = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getObjList", // New Box API route
                data: { _token: CSRF_TOKEN, folderId: projectFolderId, accessToken: BOX_ACCESS_TOKEN },
                success: resolve,
                error: reject
            });
        });

        if (!fileList || fileList.length === 0) {
            throw new Error(`No valid OBJ/MTL pairs found in Box folder ID "${projectFolderId}".`);
        }
        
        // 2. Create an array of promises to load all models
        const loadingPromises = fileList.map((filePair, index) => {
            return new Promise(async (resolve, reject) => {
                try {
                    // Manually fetch MTL and OBJ content from Box
                    const mtlContent = await fetchBoxFileContent(filePair.mtl.id);
                    const objContent = await fetchBoxFileContent(filePair.obj.id);

                    // Parse the materials and then the object
                    const mtlLoader = new MTLLoader();
                    const materialsCreator = mtlLoader.parse(mtlContent, ''); // Path can be empty as textures are not used
                    materialsCreator.preload();

                    const objLoader = new OBJLoader();
                    objLoader.setMaterials(materialsCreator);
                    const object = objLoader.parse(objContent); // Use .parse() for content, .load() for URL
                    
                    if (loaderTextElement) {
                         loaderTextElement.textContent = `Loading Geometry... (${index + 1}/${fileList.length})`;
                    }
                    
                    resolve(object);
                } catch (error) {
                    console.error(`Failed to load ${filePair.obj.name} from Box:`, error);
                    reject(error);
                }
            });
        });

        // 3. Wait for ALL models to load and be parsed
        const loadedObjects = await Promise.all(loadingPromises);

        // 4. Combine all loaded objects into a single group
        // ... (This logic remains the same as before) ...
        
        // 5. Process the COMBINED model (center, scale, rotate)
        // ... (This logic remains the same as before) ...

        // 6. Fetch category data for all parts
        // ... (This logic remains the same as before) ...
        
        // 7. Build the tree, add to scene, frame, and hide loader
        // ... (This logic remains the same as before) ...
        
    } catch (error) {
        console.error('Failed to load model:', error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model. Check console.`;
    }
}

// --- NEW Function to populate the dropdown ---
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

        modelSelector.innerHTML = ''; // Clear existing options
        if (projects && projects.length > 0) {
            projects.forEach(project => {
                const option = document.createElement('option');
                option.value = project.id; // The value is the FOLDER ID
                option.textContent = project.name; // The text is the FOLDER NAME
                modelSelector.appendChild(option);
            });
            // Trigger the loading of the first model in the list
            loadModel(projects[0].id);
        } else {
            if (loaderTextElement) loaderTextElement.textContent = "No projects found in the main folder.";
        }
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
        if (loaderTextElement) loaderTextElement.textContent = "Error fetching project list from Box.";
    }
}

// --- Event Listeners and Animation Loop ---
// ... (Keep your existing click, close, search, and resize listeners) ...

// --- MODIFIED Event listener for the model selector dropdown ---
if (modelSelector) {
    modelSelector.addEventListener('change', (event) => {
        const selectedProjectFolderId = event.target.value;
        loadModel(selectedProjectFolderId);
    });
}

// --- Start Application ---
populateProjectDropdown(); // Start by populating the dropdown
animate();












function updatePartnerCompanyInfo($folderId, $access_token)
    {
        try {
            $client = new \GuzzleHttp\Client();
            $client = new \GuzzleHttp\Client(['verify' => false]);
            $requestURL = "https://api.box.com/2.0/folders/" . $folderId . "/items/";
            $header = [
                "Authorization" => "Bearer " . $access_token,
                "Accept" => "application/json"
            ];
            $response = $client->request('GET', $requestURL, ['headers' => $header]);
            $items = $response->getBody()->getContents();
            $items = json_decode($items)->entries;
            foreach ($items as $item) {
                if ($item->type == "file") {
                    $fileId = $item->id;
                    $fileName = $item->name; //パートナー会社管理表
                    if (strstr($fileName, "パートナー会社管理表") == true) {
                        $requestURL = "https://api.box.com/2.0/files/" . $fileId . "/content/";
                        $response = $client->request('GET', $requestURL, ['headers' => $header]);
                        $file_content = $response->getBody()->getContents();
                        $filePath = public_path() . "/Download/allstore_excel.xlsx";
                        file_put_contents($filePath, $file_content);
                        $aaa = $this->LoadExcelData($filePath);
                        return $aaa;
                    }
                }
            }
        } catch (Exception $e) {
            return back()->with("error", "failed when reading　allstore info from box");
        }
    }
    
    
    
    
    <?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\File;

class FileController extends Controller
{
    public function getObjList(Request $request)
    {
        $projectName = $request->input('projectName'); // "GF", "BIKEN" など

        if (empty($projectName)) {
            return response()->json(['error' => 'Project name is required.'], 400);
        }

        // ここではサーバーのローカルパスを想定しています。
        // BOXから取得する場合は、BOX APIを使ってファイルリストを取得するロジックに置き換えます。
        $basePath = public_path('objFiles/' . $projectName); // 例: public/objFiles/GF/

        if (!File::isDirectory($basePath)) {
            return response()->json(['error' => 'Project directory not found.'], 404);
        }

        $files = File::files($basePath);
        $objFiles = [];

        foreach ($files as $file) {
            if (strtolower($file->getExtension()) === 'obj') {
                // ファイル名と、対応するMTLファイル名（拡張子を.mtlに変えたもの）をセットにする
                $objFileName = $file->getFilename();
                $mtlFileName = pathinfo($objFileName, PATHINFO_FILENAME) . '.mtl';
                
                // 対応するMTLファイルが存在するか確認
                if (File::exists($basePath . '/' . $mtlFileName)) {
                    $objFiles[] = [
                        'obj' => $objFileName,
                        'mtl' => $mtlFileName
                    ];
                } else {
                    // MTLがない場合は、mtlプロパティをnullにするか、リストに含めない
                    // ここでは含めない方針
                    \Log::warning("MTL file not found for: " . $objFileName);
                }
            }
        }

        return response()->json($objFiles);
    }
}]






// ... (Imports, UI Elements, Global variables, Scene, Camera, etc. remain the same) ...

// --- Model Configuration ---
// This is now just a list of project keys and their display names
const models = {
    'GF': {
        name: 'GF本社移転',
        path: '/ccc/public/objFiles/GF/' // Base path for this project's files
    },
    'BIKEN': {
        name: 'BIKEN (Placeholder)',
        path: '/ccc/public/objFiles/BIKEN/'
    },
    // Add other projects here
};


// --- REFACTORED Main Application Flow ---
async function loadModel(modelKey) {
    // 0. Reset scene and state
    if (loadedObjectModelRoot) scene.remove(loadedObjectModelRoot);
    loadedObjectModelRoot = null;
    selectedObjectOrGroup = null;
    // ... (clear all other maps and UI state) ...
    modelTreeList.innerHTML = '';
    if (modelTreePanel) modelTreePanel.style.display = 'none';
    updateInfoPanel();

    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';
        
        const modelConfig = models[modelKey];
        if (!modelConfig) throw new Error(`Model key "${modelKey}" not found.`);

        // 1. Fetch the list of OBJ/MTL files from the server
        if (loaderTextElement) loaderTextElement.textContent = `Fetching file list...`;
        const fileList = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/files/getObjList",
                data: { _token: CSRF_TOKEN, projectName: modelKey },
                success: resolve,
                error: reject
            });
        });

        if (!fileList || fileList.length === 0) {
            throw new Error(`No OBJ files found for project "${modelKey}".`);
        }

        // 2. Create an array of promises to load all models
        const loadingPromises = fileList.map((filePair, index) => {
            return new Promise(async (resolve, reject) => {
                try {
                    const mtlLoader = new MTLLoader();
                    mtlLoader.setPath(modelConfig.path);
                    const materialsCreator = await mtlLoader.loadAsync(filePair.mtl);
                    materialsCreator.preload();

                    const objLoader = new OBJLoader();
                    objLoader.setMaterials(materialsCreator);
                    const object = await objLoader.loadAsync(modelConfig.path + filePair.obj);
                    
                    // Update progress on the main loader text
                    if (loaderTextElement) {
                         loaderTextElement.textContent = `Loading Geometry... (${index + 1}/${fileList.length})`;
                    }
                    
                    resolve(object);
                } catch (error) {
                    console.error(`Failed to load ${filePair.obj}:`, error);
                    reject(error);
                }
            });
        });

        // 3. Wait for ALL models to load
        const loadedObjects = await Promise.all(loadingPromises);

        // 4. Combine all loaded objects into a single group
        const combinedModel = new THREE.Group();
        loadedObjects.forEach(object => {
            // OBJLoader can return a group with children. We add the children directly
            // to avoid an extra layer of grouping.
            while(object.children.length > 0) {
                combinedModel.add(object.children[0]);
            }
        });
        loadedObjectModelRoot = combinedModel;

        // 5. Process the COMBINED model (center, scale, rotate)
        const initialBox = new THREE.Box3().setFromObject(loadedObjectModelRoot);
        const initialCenter = initialBox.getCenter(new THREE.Vector3());
        loadedObjectModelRoot.position.sub(initialCenter); // Center the group
        
        const scaledBox = new THREE.Box3().setFromObject(loadedObjectModelRoot);
        const maxDim = Math.max(scaledBox.getSize(new THREE.Vector3()).x, scaledBox.getSize(new THREE.Vector3()).y, scaledBox.getSize(new THREE.Vector3()).z);
        const desiredMaxDimension = 150;
        if (maxDim > 0) {
            const scale = desiredMaxDimension / maxDim;
            loadedObjectModelRoot.scale.set(scale, scale, scale);
        }
        loadedObjectModelRoot.rotation.x = -Math.PI / 2;

        // 6. Fetch category data for all parts of the combined model
        const allIds = [];
        loadedObjectModelRoot.traverse(child => {
            if (child.name) { // Any named object could have an ID
                const splitIndex = Math.max(child.name.lastIndexOf('_'), child.name.lastIndexOf('＿'));
                if (splitIndex > 0) allIds.push(child.name.substring(splitIndex + 1));
            }
        });
        await fetchAllCategoryData(parsedWSCenID, [...new Set(allIds)]);
        
        // 7. Build the tree, add to scene, frame, and hide loader
        await buildAndPopulateCategorizedTree();
        scene.add(loadedObjectModelRoot);
        frameObject(loadedObjectModelRoot);
        if (loaderContainer) loaderContainer.style.display = 'none';

    } catch (error) {
        console.error('Failed to initialize the viewer:', error);
        if (loaderContainer) loaderContainer.style.display = 'flex';
        if (loaderTextElement) loaderTextElement.textContent = `Error loading model: ${modelKey}. Check console.`;
    }
}

// --- Start Application ---
// Load the default model ("GF") when the page first loads
loadModel('GF');
animate();
