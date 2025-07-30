    <script>
    window.objAssetPath = "https://ohb-spoke-ccc-deployment.azurewebsites.net/objFiles/";
    
    
    try {
        if (loaderContainer) loaderContainer.style.display = 'flex';

        const modelFiles = models[modelKey];
        if (!modelFiles) {
            throw new Error(`Model key "${modelKey}" not found in configuration.`);
        }

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
        const text = await response.text();
        const lines = text.split(/\r?\n/);
        if (lines.length > 0) {
            const firstLine = lines[0].trim();
            if (firstLine.startsWith("# ")) {
                const content = firstLine.substring(2).trim();
                const pattern1Match = content.match(/^([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})_([a-zA-Z0-9]+)$/);
                if (pattern1Match) {
                    parsedWSCenID = pattern1Match[1];
                    parsedPJNo = pattern1Match[2];
                    return;
                }
                if (content.includes("ワークシェアリングされてない") && (content.includes("_PJNo無し") || content.includes("＿PJNo無し"))) {
                    parsedWSCenID = "";
                    parsedPJNo = "";
                    return;
                }
            }
        }
    } catch (error) {
        console.error("Error fetching or parsing OBJ header:", error);
    }
}
