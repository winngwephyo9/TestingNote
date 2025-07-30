
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
