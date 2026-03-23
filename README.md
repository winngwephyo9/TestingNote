import open3d as o3d
import open3d.ml.torch as ml3d
import torch
import numpy as np
import laspy
import os

def main():
    base_path = "/mnt/d/PointCloudLibrary/Open3D-ML"
    ckpt_path = os.path.join(base_path, "randlanet_s3dis_202201071330utc.pth") 
    input_las = os.path.join(base_path, "3d_point_data/Zuiko314.las")
    output_las = os.path.join(base_path, "3d_point_data/Zuiko314_segmented.las")
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

    # --- 1. モデル準備 (本来の6次元入力に設定) ---
    model_cfg = {
        'name': 'RandLANet', 'num_neighbors': 16, 'num_layers': 5, 'num_classes': 13,
        'dim_input': 6,        # XYZ + RGB の6次元に戻す
        'dim_features': 8, 
        'dim_output': [16, 64, 128, 256, 512],
        'sub_sampling_ratio': [4, 4, 4, 4, 4], 'grid_size': 0.04
    }
    model = ml3d.models.RandLANet(**model_cfg)
    
    # 重みのロード（次元カットせずそのまま読み込む）
    checkpoint = torch.load(ckpt_path, map_location=device)
    state_dict = checkpoint['model_state_dict'] if 'model_state_dict' in checkpoint else checkpoint
    model.load_state_dict(state_dict)
    
    pipeline = ml3d.pipelines.SemanticSegmentation(model, device=device)
    pipeline.model.to(device)
    pipeline.model.eval()

    # --- 2. データ読み込みと6次元化 ---
    las = laspy.read(input_las)
    xyz = np.stack([las.x, las.y, las.z], axis=1).astype(np.float32)
    
    # RGBの取得と正規化 (S3DISモデルは 0.0-255.0 の範囲を想定)
    if hasattr(las, 'red'):
        # laspyのRGBは16bit(0-65535)の場合があるため、8bit(0-255)に変換
        red = np.array(las.red) / (256 if np.max(las.red) > 255 else 1)
        green = np.array(las.green) / (256 if np.max(las.green) > 255 else 1)
        blue = np.array(las.blue) / (256 if np.max(las.blue) > 255 else 1)
        rgb = np.stack([red, green, blue], axis=1).astype(np.float32)
    else:
        print("Warning: No RGB found. Using dummy gray color.")
        rgb = np.full((len(xyz), 3), 128, dtype=np.float32)

    # 座標の正規化（中心を0に寄せる）
    xyz_min = np.min(xyz, axis=0)
    xyz_norm = xyz - xyz_min

    # 特徴量：[X, Y, Z, R, G, B]
    features = np.concatenate([xyz_norm, rgb], axis=1)

    inference_data = {
        'point': xyz_norm,
        'feat': features, 
        'label': np.zeros(len(xyz_norm), dtype=np.int32)
    }

    # --- 3. 推論実行 ---
    print(f"Running Inference with RGB-Aware Model (Points: {len(xyz)})...")
    with torch.no_grad():
        results = pipeline.run_inference(inference_data)
        labels = results['predict_labels']

    # --- 4. LAS保存 ---
    new_las = laspy.create(point_format=las.header.point_format, file_version=las.header.version)
    new_las.header.scales = las.header.scales
    new_las.header.offsets = las.header.offsets
    new_las.x, new_las.y, new_las.z = las.x, las.y, las.z
    new_las.classification = labels.astype(np.uint8)
    if hasattr(las, 'red'):
        new_las.red, new_las.green, new_las.blue = las.red, las.green, las.blue
    
    new_las.write(output_las)
    print(f"Saved: {output_las}")

if __name__ == "__main__":
    main()


Potree.loadPointCloud("pointclouds/index/cloud.js", "index", function (e) {
    let pointcloud = e.pointcloud;
    let material = pointcloud.material;

    // 分類表示をデフォルトにする
    material.activeAttributeName = "classification"; 
    material.pointSizeType = Potree.PointSizeType.ADAPTIVE;
    material.size = 1;

    // S3DISの13クラス定義に正確に合わせる
    viewer.setClassifications({
        0:  { visible: true, name: 'Ceiling',  color: [1.0, 1.0, 0.0, 1.0] }, // 黄
        1:  { visible: true, name: 'Floor',    color: [0.0, 1.0, 0.0, 1.0] }, // 緑
        2:  { visible: true, name: 'Wall',     color: [1.0, 0.0, 0.0, 1.0] }, // 赤
        3:  { visible: true, name: 'Beam',     color: [0.0, 1.0, 1.0, 1.0] }, // 水色
        4:  { visible: true, name: 'Column',   color: [0.5, 0.0, 0.5, 1.0] }, // 紫
        5:  { visible: true, name: 'Window',   color: [0.0, 0.0, 1.0, 1.0] }, // 青
        6:  { visible: true, name: 'Door',     color: [1.0, 0.5, 0.0, 1.0] }, // オレンジ
        7:  { visible: true, name: 'Table',    color: [0.7, 0.3, 0.1, 1.0] },
        8:  { visible: true, name: 'Chair',    color: [0.3, 0.3, 0.3, 1.0] },
        9:  { visible: true, name: 'Sofa',     color: [1.0, 0.7, 0.7, 1.0] },
        10: { visible: true, name: 'Bookcase', color: [0.5, 0.5, 0.0, 1.0] },
        11: { visible: true, name: 'Board',    color: [0.0, 0.5, 0.5, 1.0] },
        12: { visible: true, name: 'Clutter',  color: [0.7, 0.7, 0.7, 1.0] }  // グレー
    });

    viewer.scene.addPointCloud(pointcloud);
    viewer.fitToScreen();
});




import open3d as o3d
import open3d.ml.torch as ml3d
import torch
import torch.nn as nn
import numpy as np
import laspy
import os

def main():
    base_path = "/mnt/d/PointCloudLibrary/Open3D-ML"
    ckpt_path = os.path.join(base_path, "randlanet_s3dis_202201071330utc.pth") 
    input_las = os.path.join(base_path, "3d_point_data/Zuiko314.las")
    output_las = os.path.join(base_path, "3d_point_data/Zuiko314_segmented.las")
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

    # --- 1. モデル準備 (入力を3次元に設定) ---
    model_cfg = {
        'name': 'RandLANet', 'num_neighbors': 16, 'num_layers': 5, 'num_classes': 13,
        'dim_input': 3,        # 3次元(XYZ)のみを受け取るように変更
        'dim_features': 8, 
        'dim_output': [16, 64, 128, 256, 512],
        'sub_sampling_ratio': [4, 4, 4, 4, 4], 'grid_size': 0.04
    }
    model = ml3d.models.RandLANet(**model_cfg)
    model.device = device
    
    # 重みのロードと特殊な変換
    checkpoint = torch.load(ckpt_path, map_location=device)
    state_dict = checkpoint['model_state_dict'] if 'model_state_dict' in checkpoint else checkpoint
    
    # 【重要】元が6次元の重みから、XYZに対応する最初の3次元分だけを取り出す
    if state_dict['fc0.weight'].shape[1] == 6:
        print("Converting 6D weights to 3D (extracting XYZ components)...")
        original_weight = state_dict['fc0.weight'] # [8, 6]
        state_dict['fc0.weight'] = original_weight[:, :3] # [8, 3] に切り出し
        
    model.load_state_dict(state_dict)
    
    pipeline = ml3d.pipelines.SemanticSegmentation(model, device=device)
    pipeline.model.to(device)
    pipeline.model.eval()

    # --- 2. データ読み込み ---
    las = laspy.read(input_las)
    xyz_raw = np.stack([las.x, las.y, las.z], axis=1).astype(np.float32)
    
    # 座標の正規化
    xyz_min = np.min(xyz_raw, axis=0)
    xyz_norm = xyz_raw - xyz_min
    if np.max(xyz_norm[:, 2]) > 50.0:
        xyz_norm *= 0.001
    
    # Pipelineに渡すデータ (featはあえてXYZと同じにする)
    inference_data = {
        'point': xyz_norm,
        'feat': xyz_norm, # 3次元として渡す
        'label': np.zeros(len(xyz_norm), dtype=np.int32)
    }

    # --- 3. 推論実行 ---
    print(f"Final Height for AI: {np.max(xyz_norm[:, 2]):.2f}m")
    print("Running Inference with 3D-Adapted Model...")
    
    with torch.no_grad():
        results = pipeline.run_inference(inference_data)
        labels = results['predict_labels']

    # --- 4. 結果表示と保存 ---
    unique, counts = np.unique(labels, return_counts=True)
    labels_dict = {0:"ceiling", 1:"floor", 2:"wall", 3:"beam", 4:"column", 5:"window", 
                   6:"door", 7:"table", 8:"chair", 9:"sofa", 10:"bookcase", 11:"board", 12:"clutter"}
    
    print("\n--- AI Classification Stats ---")
    for u, c in zip(unique, counts):
        name = labels_dict.get(int(u), f"unknown({u})")
        print(f"ID {u:2} ({name:8}): {c:10} points")

    new_las = laspy.create(point_format=las.header.point_format, file_version=las.header.version)
    new_las.header.scales = las.header.scales
    new_las.header.offsets = las.header.offsets
    new_las.x, new_las.y, new_las.z = las.x, las.y, las.z
    new_las.classification = labels.astype(np.uint8)
    if hasattr(las, 'red'):
        new_las.red, new_las.green, new_las.blue = las.red, las.green, las.blue
    
    new_las.write(output_las)
    print(f"\nSaved successfully to: {output_las}")

if __name__ == "__main__":
    main()
    

    <!DOCTYPE html>
<html lang="en">

<head>
	<meta charset="utf-8">
	<meta name="description" content="">
	<meta name="author" content="">
	<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
	<title>Potree Viewer</title>

	<link rel="stylesheet" type="text/css" href="./libs/potree/potree.css">
	<link rel="stylesheet" type="text/css" href="./libs/jquery-ui/jquery-ui.min.css">
	<link rel="stylesheet" type="text/css" href="./libs/openlayers3/ol.css">
	<link rel="stylesheet" type="text/css" href="./libs/spectrum/spectrum.css">
	<link rel="stylesheet" type="text/css" href="./libs/jstree/themes/mixed/style.css">
</head>

<body>
	<script src="./libs/jquery/jquery-3.1.1.min.js"></script>
	<script src="./libs/spectrum/spectrum.js"></script>
	<script src="./libs/jquery-ui/jquery-ui.min.js"></script>
	<script src="./libs/three.js/build/three.min.js"></script>
	<script src="./libs/three.js/extra/lines.js"></script>
	<script src="./libs/other/BinaryHeap.js"></script>
	<script src="./libs/tween/tween.min.js"></script>
	<script src="./libs/d3/d3.js"></script>
	<script src="./libs/proj4/proj4.js"></script>
	<script src="./libs/openlayers3/ol.js"></script>
	<script src="./libs/i18next/i18next.js"></script>
	<script src="./libs/jstree/jstree.js"></script>
	<script src="./libs/potree/potree.js"></script>
	<script src="./libs/plasio/js/laslaz.js"></script>

	<!-- INCLUDE ADDITIONAL DEPENDENCIES HERE -->
	document.title = "";
	viewer.setEDLEnabled(false);
	viewer.setBackground("gradient"); // ["skybox", "gradient", "black", "white"];
	viewer.setDescription(``);

	<div class="potree_container" style="position: absolute; width: 100%; height: 100%; left: 0px; top: 0px; ">
		<div id="potree_render_area" style="background-image: url('./libs/potree/resources/images/background.jpg');">
		</div>
		<div id="potree_sidebar_container"> </div>
	</div>

	<script>

		window.viewer = new Potree.Viewer(document.getElementById("potree_render_area"));

		viewer.setEDLEnabled(true);
		viewer.setFOV(60);
		viewer.setPointBudget(2_000_000);
		document.title = "";
		viewer.setEDLEnabled(false);
		viewer.setBackground("gradient"); // ["skybox", "gradient", "black", "white"];
		viewer.setDescription(``);
		viewer.loadSettingsFromURL();

		viewer.setDescription("");

		viewer.loadGUI(() => {
			viewer.setLanguage('en');
			$("#menu_appearance").next().show();
			$("#menu_tools").next().show();
			$("#menu_clipping").next().show();
			viewer.toggleSidebar();
		});

		// Potree.loadPointCloud("pointclouds/index/cloud.js", "index", e => {
		// 	let pointcloud = e.pointcloud;
		// 	let material = pointcloud.material;
		// 	viewer.scene.addPointCloud(pointcloud);
		// 	//material.activeAttributeName = "rgba"; //[rgba, intensity, classification, ...]
		// 	material.size = 1;
		// 	material.pointSizeType = Potree.PointSizeType.ADAPTIVE;
		// 	material.shape = Potree.PointShape.SQUARE;
		// 	viewer.fitToScreen();
		// });

		Potree.loadPointCloud("pointclouds/index/cloud.js", "index", function (e) {
			let pointcloud = e.pointcloud;
			let material = pointcloud.material;

			// --- 1. 表示設定をRGBA（自分で塗った色）に固定する ---
			material.activeAttributeName = "rgba";
			material.pointSizeType = Potree.PointSizeType.ADAPTIVE;
			material.size = 1;

			// --- 2. Classification（左側メニュー）の日本語化と定義追加 ---
			// ここで番号(2, 5, 6, 8, 11)と名前を紐付けます
			viewer.setClassifications({
				// --- 既存の定義 ---
				0: { visible: true, name: '未分類', color: [0.5, 0.5, 0.5, 1.0] },
				2: { visible: true, name: '地面', color: [0.6, 0.4, 0.2, 1.0] },
				5: { visible: true, name: '高木(植物)', color: [0.0, 0.6, 0.0, 1.0] },
				6: { visible: true, name: '建物', color: [1.0, 0.0, 0.0, 1.0] },
				8: { visible: true, name: '車', color: [1.0, 1.0, 0.0, 1.0] },
				11: { visible: true, name: '道路', color: [0.4, 0.4, 0.4, 1.0] },

				// --- 追加可能なクラス ---
				3: { visible: true, name: '低木', color: [0.3, 1.0, 0.3, 1.0] },
				4: { visible: true, name: '中木', color: [0.0, 0.8, 0.0, 1.0] },
				7: { visible: true, name: 'ノイズ', color: [1.0, 0.0, 1.0, 1.0] }, // ピンク系
				9: { visible: true, name: '水面', color: [0.0, 0.0, 1.0, 1.0] }, // 青
				10: { visible: true, name: 'レール/軌道', color: [0.8, 0.8, 0.2, 1.0] },
				12: { visible: true, name: 'オーバーラップ', color: [1.0, 1.0, 0.0, 0.5] }, // 半透明
				13: { visible: true, name: '電線', color: [1.0, 0.5, 0.0, 1.0] }, // オレンジ
				14: { visible: true, name: '電柱/塔', color: [0.6, 0.3, 0.0, 1.0] },
				15: { visible: true, name: '橋梁', color: [0.0, 0.7, 0.7, 1.0] },
				17: { visible: true, name: '縁石(路肩)', color: [0.7, 0.7, 0.7, 1.0] }
			});

			viewer.scene.addPointCloud(pointcloud);
			viewer.fitToScreen();
		});

	</script>


</body>

</html>

これで大丈夫ですが、詳しく見るとdoorの色はdoorの部分だけでなくwallにも付いていると思ういます。Windowも同じWindowの部分だけでなく他の部分にも色が付いています。

きちんとclassificationを修正してください。
    
    <img width="1146" height="912" alt="image" src="https://github.com/user-attachments/assets/37270175-8ec1-4453-becc-3e62da3abf5d" />
<img width="1111" height="901" alt="image" src="https://github.com/user-attachments/assets/8e003755-ee21-419c-aaae-15c71ca103f0" />
