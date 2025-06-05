You're right to point that out! When your CSV files are already being managed by Box Sync (meaning they are in your local C:\Users\UHR757\Box\... folder and automatically synced to Box cloud), this simplifies the "getting the file" part of the Power Automate flow significantly.

Here's how you'd adapt the Power Automate solution, focusing on Box:

Adapted Solution: Automate with Microsoft Power Automate (Leveraging Box)
The core principle remains the same: use Power Automate to orchestrate the process. The key difference is how Power Automate accesses the files.

The most straightforward approach is to leverage the Box cloud connector directly. Since your local Box folder syncs to the Box cloud, the files effectively exist in the Box cloud. Power Automate can directly connect to the Box cloud.

Part 1: Ensure Box Sync is Working Reliably
Crucial: Make sure your Box Sync client on C:\Users\UHR757\ is always running and successfully syncing your TestingData folder to your Box cloud account. Power Automate will be reading from the Box cloud, not your local C:\Users\UHR757\Box\ folder directly.
Part 2: Building the Power Automate Flow (with Box as Source)
Go to Power Automate:

Open your web browser and go to flow.microsoft.com.
Sign in with your Microsoft account that has access to SharePoint and Microsoft Teams.
Create a New Flow:

On the left navigation pane, click on "My flows."
Click on "New flow" and then select "Scheduled cloud flow."
Schedule your Flow:

Flow name: Give it a descriptive name (e.g., "Box CSV to SharePoint & Teams Notify").
Start date: Set today's date.
Start time: Set it to 09:00 PM (9 PM JST, since you're in Osaka).
Repeat every: Set to 1 week.
On these days: Select Monday.
Click "Create."
Add Actions to your Flow:

Trigger (already set): The schedule you just configured.

Action 1: List files from Box:

Click "+ New step."
Search for "Box."
Select the action "List files in folder."
Folder: You'll be prompted to sign in to your Box account if you haven't already. After connecting, browse to the specific folder within your Box account where your CSV files are located (this will correspond to 個人データ\ウィングウェピョー\TestingData\ in your Box cloud).
Action 2: Apply to each file (if processing multiple CSVs):

Click "+ New step."
Search for "Apply to each."
Select the "value" dynamic content from the "List files in folder" step.
Inside the "Apply to each" loop, add the next action.
Action 3: Get file content from Box:

Inside the "Apply to each" loop (or directly if you're only expecting one file), click "+ Add an action."
Search for "Box."
Select "Get file content."
File: Use the "Id" dynamic content from the "List files in folder" step.
Action 4: Create file in SharePoint:

Inside the "Apply to each" loop (or directly), click "+ Add an action."
Search for "SharePoint."
Select "Create file."
Site Address: Select the SharePoint site where you want to upload the files.
Folder Path: Select the specific document library and folder within that site where you want to upload the CSVs.
File Name: Use the "Name" dynamic content from the "List files in folder" step (this should include the .csv extension).
File Content: Use the "File content" dynamic content from the "Get file content" (Box) step.
Action 5: Send a message to Microsoft Teams:

Click "+ New step" (outside the "Apply to each" loop if you want one message for all uploads, or inside if you want a message per file).
Search for "Microsoft Teams."
Select "Post message in a chat or channel."
Post as: Choose "Flow bot" or "User."
Post in: Choose "Channel."
Team: Select the specific Microsoft Team.
Channel: Select the channel where you want the notification.
Message: Compose your notification message.
Example: "CSV file uploaded from Box to SharePoint! File Name: [Dynamic Content: Name]"
Save and Test:

Click "Save" at the top right.
Once saved, you can click "Test" to run the flow manually. This is essential for verifying connections and paths.



use PhpOffice\PhpSpreadsheet\Spreadsheet;
use PhpOffice\PhpSpreadsheet\Writer\Xlsx;

$spreadsheet = new Spreadsheet();
$sheet = $spreadsheet->getActiveSheet();

// Let's assume $csvData is already UTF-8 decoded string
$rows = explode("\n", $csvData);
foreach ($rows as $rowIndex => $row) {
    $columns = str_getcsv($row);
    foreach ($columns as $colIndex => $value) {
        $sheet->setCellValueByColumnAndRow($colIndex + 1, $rowIndex + 1, $value);
    }
}

$fileName = $filePrefix . date('Ymd') . '.xlsx';
$writer = new Xlsx($spreadsheet);
$writer->save($filePath);

   
   private function handleCsvDownload($response, $filePath, string $filePrefix): void
    {
        if ($response->getStatusCode() === 200) {
            $contentType = $response->getHeaderLine('Content-Type');
            if (strpos($contentType, 'text/csv') !== false || strpos($contentType, 'application/octet-stream') !== false) {
                // $csvData = mb_convert_encoding((string) $response->getBody(), 'UTF-8', 'shift_jis');
                $rawCsvData = (string) $response->getBody();

                // Convert from CP932 to UTF-8
                // $csvData = mb_convert_encoding($rawCsvData, 'UTF-8', 'CP932');
                $csvData = mb_convert_encoding($rawCsvData, 'UTF-8', 'Shift_JIS');

                // --- ADD THIS LINE TO PREPEND THE UTF-8 BOM ---
                // The UTF-8 BOM is represented by these three bytes: EF BB BF
                $csvDataWithBom = "\xEF\xBB\xBF" . $csvData;
                $fileName = $filePrefix . date('Ymd') . '.csv';
                $filePath = $filePath . "\\" . $fileName;
                file_put_contents($filePath, $csvDataWithBom);
                dump("{$fileName} downloaded and uploaded to Box successfully.");
            } else {
                throw new \Exception("Expected CSV data in response, but got: " . $contentType);
            }
        } else {
            throw new \Exception("Error submitting search on ADDR102.aspx: " . $response->getStatusCode());
        }
    }

![image](https://github.com/user-attachments/assets/fff32f8a-c8e7-44db-8ae3-3f25e6f05b2d)


