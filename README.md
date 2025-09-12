ALTER TABLE model_file_cache
    MODIFY file_name VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
    MODIFY base_name VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL,
    MODIFY content LONGTEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- テーブル全体のデフォルト設定も変更
ALTER TABLE model_file_cache CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
