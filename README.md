async function fetchBoxFileContent(fileId) {
    console.log("Field Id of fetchBoxFileContent()", fileId);
    try {
        const response = await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/box/getFileContents",
                data: { _token: CSRF_TOKEN, fileId: fileId },
                success: resolve,
                error: reject
            });
        });
        if (!response.ok) throw new Error(`Failed to fetch file ${fileId} from Box: ${response.statusText}`);
        return await response.text();
    } catch (error) {
        console.error("Failed to populate project dropdown:", error);
        if (loaderTextElement) loaderTextElement.textContent = "Error fetching project list from Box.";
    }
}
