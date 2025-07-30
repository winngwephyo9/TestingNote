/objFiles/240324_GF%…2022_20250627.obj:1 
 Failed to load resource: the server responded with a status of 404 (Not Found)

objViewer.js:256 Failed to fetch OBJ for header parsing: Not Found
parseObjHeader	@	objViewer.js:256
/objFiles/240324_GF%…2022_20250627.mtl:1 
 Failed to load resource: the server responded with a status of 404 (Not Found)
objViewer.js:246 Failed to initialize the viewer: HttpError: fetch for "https://ohb-spoke-ccc-deployment.azurewebsites.net/objFiles/240324_GF%E6%9C%AC%E7%A4%BE%E7%A7%BB%E8%BB%A2_2022_20250627.mtl" responded with 404: Not Found
    at three.core.js:43923:12
loadModel	@	objViewer.js:246


<img width="1025" height="252" alt="image" src="https://github.com/user-attachments/assets/16fd9875-516a-4edf-bab0-b52838ebd12c" />

﻿
 GET https://ohb-spoke-ccc-deployment.azurewebsites.net/objFiles/240324_GF%E6%9C%AC%E7%A4%BE%E7%A7%BB%E8%BB%A2_2022_20250627.obj 404 (Not Found)

        const assetPath = '/ccc/public/objFiles/';
        const objFileName = modelFiles.obj;
        const mtlFileName = modelFiles.mtl;
        const fullObjPath = assetPath + objFileName;

        // Step 1: Parse header info
        await parseObjHeader(fullObjPath);

async function parseObjHeader(filePath) {
    try {
        const response = await fetch(filePath);
        if (!response.ok) {
            console.error(`Failed to fetch OBJ for header parsing: ${response.statusText}`);
            return;
        }
}
