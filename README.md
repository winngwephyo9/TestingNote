// 'model_files/project123/file.txt' に 'Hello World' という内容を書き込む
Storage::disk('local')->put('model_files/project123/file.txt', 'Hello World');


// 'model_files/project123/file.txt' の中身を取得
$content = Storage::disk('local')->get('model_files/project123/file.txt');

if (Storage::disk('local')->exists('model_files/project123/file.txt')) {
    // ...
}
