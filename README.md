[2025-09-18 10:17:38] local.WARNING: Box token expired (401). Will attempt to refresh after this batch.  
[2025-09-18 10:17:51] local.ERROR: Exception occurred while refreshing Box token: Client error: `POST https://api.box.com/oauth2/token` resulted in a `400 Bad Request` response:
{"error":"invalid_grant","error_description":"Invalid refresh token"}
  
[2025-09-18 10:17:51] local.ERROR: Failed to refresh Box token. Aborting sync process.  
[2025-09-18 10:17:51] local.ERROR: Download aborted in chunk 23: Failed to refresh token.  
[2025-09-18 10:17:51] local.ERROR: Box Sync Failed: Download process was aborted: Failed to refresh token. {"exception":"[object] (Exception(code: 0): Download process was aborted: Failed to refresh token. at C:\\home\\site\\wwwroot\\ccc\\app\\Models\\DLDHWDataImportModel.php:5427)
[stacktrace]
#0 C:\\home\\site\\wwwroot\\ccc\\app\\Jobs\\SyncBoxProject.php(54): App\\Models\\DLDHWDataImportModel->syncAndGetModelData()
