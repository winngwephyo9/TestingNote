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
