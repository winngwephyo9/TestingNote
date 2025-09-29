-- もしcontentカラムが残っていれば削除
-- ALTER TABLE model_file_cache DROP COLUMN content;

-- file_pathカラムがなければ追加
-- ALTER TABLE model_file_cache ADD COLUMN `file_path` VARCHAR(512) NULL DEFAULT NULL AFTER `file_type`;
