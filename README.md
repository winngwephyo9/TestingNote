<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schema;

class SearchValueInDatabase extends Command
{
    protected $signature = 'db:search-value {value}';
    protected $description = 'Search a value in all tables and columns of the database';

    public function handle()
    {
        $valueToSearch = $this->argument('value');
        $databaseName = DB::getDatabaseName();
        $tables = DB::select("SHOW TABLES");

        $tableKey = "Tables_in_$databaseName";

        foreach ($tables as $table) {
            $tableName = $table->$tableKey;
            $columns = Schema::getColumnListing($tableName);

            foreach ($columns as $column) {
                try {
                    $results = DB::table($tableName)
                        ->where($column, $valueToSearch)
                        ->limit(1)
                        ->get();

                    if ($results->isNotEmpty()) {
                        $this->info("Found in table: `$tableName`, column: `$column`");
                    }
                } catch (\Exception $e) {
                    // Handle exceptions like non-searchable columns (JSON, etc.)
                    $this->warn("Skipped $tableName.$column due to: " . $e->getMessage());
                }
            }
        }

        $this->info('Search complete.');
    }
}


php artisan db:search-value yourValueHere
