// --- NEW Data Fetching Function with Batching/Chunking ---
async function fetchAllCategoryData(wscenId, allElementIds) {
    if (allElementIds.length === 0) {
        console.log("No element IDs to fetch data for.");
        return; // Nothing to fetch
    }

    // Set the maximum number of IDs to send in a single request
    const batchSize = 900; // Keep safely below the 1000 limit
    console.log(`Total unique IDs to fetch: ${allElementIds.length}. Fetching in batches of ${batchSize}.`);

    // Loop through the IDs in chunks
    for (let i = 0; i < allElementIds.length; i += batchSize) {
        const batch = allElementIds.slice(i, i + batchSize);
        
        console.log(`Fetching batch ${i / batchSize + 1}... (${batch.length} IDs)`);
        if (loaderTextElement) {
            loaderTextElement.textContent = `Fetching Categories... (${i + batch.length}/${allElementIds.length})`;
        }

        // Use a promise to wait for each batch to complete
        await new Promise((resolve, reject) => {
            $.ajax({
                type: "post",
                url: url_prefix + "/DLDWH/getDatas", // The batch route
                data: { 
                    _token: CSRF_TOKEN, 
                    WSCenID: wscenId, 
                    ElementIds: batch // Send the current chunk of IDs
                },
                success: function (data) {
                    // console.log(`Batch ${i / batchSize + 1} data received.`, data);
                    // Add the received data to our global map
                    for (const elementId in data) {
                        elementIdDataMap.set(elementId, data[elementId]);
                    }
                    resolve(); // Resolve the promise for this batch
                },
                error: function (err) {
                    console.error(`Error fetching batch starting at index ${i}:`, err);
                    reject(err); // Reject the promise if a batch fails
                }
            });
        });
    }
    console.log("All category data fetched successfully.");
}
