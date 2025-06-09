{
    "statusCode": 200,
    "headers": {
        "Cache-Control": "no-store, max-age=0, private",
        "Transfer-Encoding": "chunked",
        "Vary": "Origin,Accept-Encoding",
        "Set-Cookie": "ARRAffinity=6e180e87d531777cf79d40ef64331d02957fd3d9c40299698b75d8bc77b9102e;Path=/;HttpOnly;Secure;Domain=sharepointonline-je.azconn-je-001.p.azurewebsites.net,ARRAffinitySameSite=6e180e87d531777cf79d40ef64331d02957fd3d9c40299698b75d8bc77b9102e;Path=/;HttpOnly;SameSite=None;Secure;Domain=sharepointonline-je.azconn-je-001.p.azurewebsites.net",
        "Strict-Transport-Security": "max-age=31536000",
        "X-NetworkStatistics": "0,4194444,0,0,54006,48625,48625,3048",
        "X-SharePointHealthScore": "2",
        "X-SP-SERVERSTATE": "ReadOnly=0",
        "DATASERVICEVERSION": "3.0",
        "SPClientServiceRequestDuration": "698",
        "IsOCDI": "0",
        "X-DataBoundary": "NONE",
        "X-1DSCollectorUrl": "https://mobile.events.data.microsoft.com/OneCollector/1.0/",
        "X-AriaCollectorURL": "https://browser.pipe.aria.microsoft.com/Collector/3.0/",
        "SPRequestGuid": "7553a6a1-0017-5000-4df1-9e4c7239cc0e",
        "Request-Id": "7553a6a1-0017-5000-4df1-9e4c7239cc0e",
        "MS-CV": "oaZTdRcAAFBN8Z5McjnMDg.0",
        "SPLogId": "7553a6a1-0017-5000-4df1-9e4c7239cc0e",
        "X-Frame-Options": "SAMEORIGIN,DENY",
        "Content-Security-Policy": "frame-ancestors 'self' teams.microsoft.com *.teams.microsoft.com *.skype.com *.teams.microsoft.us local.teams.office.com teams.cloud.microsoft *.office365.com goals.cloud.microsoft *.powerapps.com *.powerbi.com *.yammer.com engage.cloud.microsoft word.cloud.microsoft excel.cloud.microsoft powerpoint.cloud.microsoft *.officeapps.live.com *.office.com *.microsoft365.com m365.cloud.microsoft *.cloud.microsoft *.stream.azure-test.net *.dynamics.com *.microsoft.com onedrive.live.com *.onedrive.live.com securebroker.sharepointonline.com;",
        "MicrosoftSharePointTeamServices": "16.0.0.26121",
        "X-Content-Type-Options": "nosniff,nosniff",
        "X-MS-InvokeApp": "1; RequireReadOnly",
        "P3P": "CP=\"ALL IND DSP COR ADM CONo CUR CUSo IVAo IVDo PSA PSD TAI TELo OUR SAMo CNT COM INT NAV ONL PHY PRE PUR UNI\"",
        "X-AspNet-Version": "4.0.30319",
        "X-Powered-By": "ASP.NET",
        "x-ms-request-id": "7553a6a1-0017-5000-4df1-9e4c7239cc0e",
        "x-ms-environment-id": "default-89c5ff22-fdba-4401-9f8f-7af329bedd42",
        "x-ms-tenant-id": "89c5ff22-fdba-4401-9f8f-7af329bedd42",
        "x-ms-dlp-re": "-|-",
        "x-ms-dlp-gu": "-|-",
        "x-ms-dlp-ef": "-|-/-|-|-",
        "Timing-Allow-Origin": "*",
        "x-ms-apihub-cached-response": "true",
        "x-ms-apihub-obo": "false",
        "Date": "Mon, 09 Jun 2025 04:54:45 GMT",
        "Content-Type": "application/json; odata=verbose; charset=utf-8",
        "Expires": "Sun, 25 May 2025 04:54:44 GMT",
        "Last-Modified": "Mon, 09 Jun 2025 04:54:44 GMT",
        "Content-Length": "7007"
    },
    "body": {
        "d": {


base64ToBinary(base64(outputs('Get_file_content_using_id_2')?['body']))


Unable to process template language expressions in action 'Compose_2' inputs at line '0' and column '0': 'The template language expression 'base64ToBinary(body('Get_file_content_using_id_2')?['$content'])' cannot be evaluated because property '$content' cannot be selected. Property selection is not supported on values of type 'String'. Please see https://aka.ms/logicexpressions for usage details.'.



// Convert raw data from CP932 to UTF-8.
            // We will NOT add BOM here, as SharePoint/Power Automate is stripping it anyway.
            $csvData = mb_convert_encoding($rawCsvData, 'UTF-8', 'CP932');

            // *** NEW STRATEGY: Prepend Excel-specific "SEP=," line ***
            // This is a non-standard Excel hack that often helps Excel correctly interpret
            // UTF-8 CSVs when the BOM is missing, especially in non-UTF-8 default locales.
            // The "\n" ensures the actual CSV data starts on the next line.
            $excelFriendlyCsvData = "SEP=,\n" . $csvData; 
