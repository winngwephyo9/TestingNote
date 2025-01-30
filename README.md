進捗報告をさせていただきます。

DL/DWHのBIに関しては、プロジェクトと左側の各フィルターをクリックすると、そのプロジェクトに関する場所が地図上にマークされるように開発しました。現在はこの作業を一旦停止し、プラトーの調査を行っております。

プラトーの調査報告ですが、プラトーにはAPIがありません。GitHubからソースコードをダウンロードし、ローカル環境に必要な開発環境を構築して起動しましたので、新しいプロジェクトとしてなっております。このドキュメントに従い、ローカル環境で地図の確認と日本の地図の表示まで調査が完了しました。現在、3D都市モデルと地形の表示について引き続き調査を行っております。

調査が終了したら、プラトーが新しいプロジェクトとしてCCCに埋め込むことが可能かどうかを確認する予定です。もしローカル環境でプラトープロジェクトがCCCに埋め込むことができましたら、本番環境でプラトープロジェクトとCCCプロジェクトの開発環境の構築が競合する可能性があります。

以上、ご報告させていただきます。よろしくお願いいたします。


but i want to modify  this project want to add in my existing php,laravel, js project
how to integrate terriaMap project to my existing laravel project
Integrating a TerriaMap project into an existing Laravel application involves several steps. TerriaMap is a JavaScript-based mapping application, so you'll need to ensure that your Laravel application can serve the necessary static files and handle routing appropriately. Here's a step-by-step guide to help you integrate TerriaMap into your Laravel project:

1. Prepare Your TerriaMap Project
Ensure your TerriaMap project is working independently (e.g., running on http://localhost:3001).

Build your TerriaMap project for production. This typically involves running a build command like npm run build or yarn build. This will generate static files (HTML, JS, CSS, etc.) in a dist or build directory.

2. Move TerriaMap Files to Laravel's Public Directory
Copy the contents of the TerriaMap dist or build directory into your Laravel project's public directory. You can create a subdirectory like public/terriamap to keep things organized.

For example:

Copy
cp -r /path/to/terriamap/dist/* /path/to/laravel/public/terriamap/
3. Serve TerriaMap via Laravel
You can serve the TerriaMap files directly from the public/terriamap directory. Laravel will serve static files from the public directory by default.

Access the TerriaMap interface via http://your-laravel-app/terriamap/index.html.

4. Integrate TerriaMap into Laravel Views
If you want to embed TerriaMap into a Laravel Blade view, you can include the TerriaMap HTML file or use an iframe.

Example of embedding via iframe in a Blade view (resources/views/terriamap.blade.php):

blade
Copy
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TerriaMap Integration</title>
</head>
<body>
    <h1>My Laravel Application with TerriaMap</h1>
    <iframe src="{{ asset('terriamap/index.html') }}" width="100%" height="600px"></iframe>
</body>
</html>
5. Set Up Laravel Routes
Create a route in routes/web.php to serve the TerriaMap view:

php
Copy
Route::get('/terriamap', function () {
    return view('terriamap');
});
Now, you can access the TerriaMap integration at http://your-laravel-app/terriamap.

6. Handle API Requests (if needed)
If TerriaMap makes API requests to your Laravel backend, ensure you have the necessary API routes and controllers set up in Laravel.

Example API route in routes/api.php:

php
Copy
Route::get('/terriamap-data', [TerriaMapController::class, 'getData']);
7. Adjust TerriaMap Configuration
If TerriaMap requires configuration files (e.g., config.json), ensure they are correctly placed in the public/terriamap directory and referenced properly in the TerriaMap build.

8. Test the Integration
Run your Laravel application (php artisan serve) and navigate to the TerriaMap route (e.g., http://localhost:8000/terriamap).

Ensure all assets (JS, CSS, images) are loading correctly and that the map functions as expected.

9. Deploy to Production
When deploying to production, ensure your web server (e.g., Apache, Nginx) is configured to serve static files from the public/terriamap directory.

If using Laravel Forge, Vapor, or another deployment tool, follow their guidelines for serving static files.

10. Optional: Use Laravel Mix for Asset Compilation
If you want to integrate TerriaMap's build process with Laravel Mix, you can modify webpack.mix.js to include TerriaMap's build steps. This is more advanced and may require custom configuration.

Example Directory Structure
Copy
laravel-project/
├── public/
│   └── terriamap/
│       ├── index.html
│       ├── assets/
│       └── config.json
├── resources/
│   └── views/
│       └── terriamap.blade.php
├── routes/
│   └── web.php
└── ...
By following these steps, you should be able to integrate TerriaMap into your existing Laravel project. Let me know if you encounter any issues!
