Unable to process template language expressions in action 'Compose_2' inputs at line '0' and column '0': 'The template language expression 'base64ToBinary(body('Get_file_content_using_id_2')?['$content'])' cannot be evaluated because property '$content' cannot be selected. Property selection is not supported on values of type 'String'. Please see https://aka.ms/logicexpressions for usage details.'.



// Convert raw data from CP932 to UTF-8.
            // We will NOT add BOM here, as SharePoint/Power Automate is stripping it anyway.
            $csvData = mb_convert_encoding($rawCsvData, 'UTF-8', 'CP932');

            // *** NEW STRATEGY: Prepend Excel-specific "SEP=," line ***
            // This is a non-standard Excel hack that often helps Excel correctly interpret
            // UTF-8 CSVs when the BOM is missing, especially in non-UTF-8 default locales.
            // The "\n" ensures the actual CSV data starts on the next line.
            $excelFriendlyCsvData = "SEP=,\n" . $csvData; 
