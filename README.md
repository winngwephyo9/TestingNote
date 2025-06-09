 // Convert raw data from CP932 to UTF-8.
            // We will NOT add BOM here, as SharePoint/Power Automate is stripping it anyway.
            $csvData = mb_convert_encoding($rawCsvData, 'UTF-8', 'CP932');

            // *** NEW STRATEGY: Prepend Excel-specific "SEP=," line ***
            // This is a non-standard Excel hack that often helps Excel correctly interpret
            // UTF-8 CSVs when the BOM is missing, especially in non-UTF-8 default locales.
            // The "\n" ensures the actual CSV data starts on the next line.
            $excelFriendlyCsvData = "SEP=,\n" . $csvData; 
