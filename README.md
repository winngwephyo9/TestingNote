import numpy as np
import laspy

# --- パス設定 ---
bin_path = "/mnt/d/PointCloudLibrary/data/ob_testing_point_cloud_data/dataset/sequences/00/velodyne/000000.bin"
label_path = "/mnt/d/PointCloudLibrary/Open3D-ML/test/sequences/00/predictions/000000.label"
output_laz = "/mnt/d/PointCloudLibrary/Open3D-ML/3d_point_data/ob_downsampled_final.laz"

# 1. データの読み込み
points = np.fromfile(bin_path, dtype=np.float32).reshape(-1, 4)
xyz = points[:, :3].astype(np.float64)
labels = (np.fromfile(label_path, dtype=np.uint32) & 0xFFFF).astype(np.uint8)

# 2. ボクセルダウンサンプリング (1cm単位)
voxel_size = 0.01
voxel_indices = np.floor(xyz / voxel_size).astype(np.int64)
_, unique_indices = np.unique(voxel_indices, axis=0, return_index=True)

down_xyz = xyz[unique_indices]
down_labels = labels[unique_indices]

print(f"Original: {len(xyz)} -> Downsampled: {len(down_xyz)}")

# 3. LASヘッダーの設定 (重要: ここを修正)
header = laspy.LasHeader(point_format=7, version="1.4")

# Potree Converter 2.x系で精度を保つためのスケール (0.001 = 1mm精度)
header.scales = [0.001, 0.001, 0.001]

# 手動で引くのではなく、ヘッダーにオフセットを定義する
header.offsets = np.min(down_xyz, axis=0)

las = laspy.LasData(header)

# 座標の代入: 元の値をそのまま入れる (laspyが内部で offset を考慮して処理します)
las.x = down_xyz[:, 0]
las.y = down_xyz[:, 1]
las.z = down_xyz[:, 2]

# 4. 属性の代入
las.classification = down_labels
las.return_number = np.ones(len(down_xyz), dtype=np.uint8)
las.number_of_returns = np.ones(len(down_xyz), dtype=np.uint8)

# 5. 色の割り当て (16bitカラー対応)
def get_color_255(label):
    colors = {
        0: [100, 100, 100], 10: [0, 0, 255], 11: [77, 179, 255],
        13: [0, 77, 128], 15: [255, 128, 0], 18: [128, 51, 26],
        20: [77, 77, 153], 30: [255, 51, 51], 40: [255, 0, 255],
        50: [255, 204, 0], 70: [0, 255, 0], 80: [0, 255, 255], 81: [255, 255, 0]
    }
    return colors.get(label, [255, 0, 0]) 

rgb = np.array([get_color_255(l) for l in down_labels], dtype=np.uint16)
# LASは各チャンネル16bit(0-65535)なので、256倍する
las.red = rgb[:, 0] * 256
las.green = rgb[:, 1] * 256
las.blue = rgb[:, 2] * 256

# 6. 保存
las.update_header()
las.write(output_laz)
print(f"--- 完了! {output_laz} ---")








import numpy as np
import laspy
import os

# --- パス設定 ---
bin_path = "/mnt/d/PointCloudLibrary/data/ob_testing_point_cloud_data/dataset/sequences/00/velodyne/000000.bin"
label_path = "/mnt/d/PointCloudLibrary/Open3D-ML/test/sequences/00/predictions/000000.label"
output_laz = "/mnt/d/PointCloudLibrary/Open3D-ML/3d_point_data/ob_downsampled_final.laz"

# 1. データの読み込み
points = np.fromfile(bin_path, dtype=np.float32).reshape(-1, 4)
xyz = points[:, :3].astype(np.float64)
# 下位16bitをラベルとして取得
labels = (np.fromfile(label_path, dtype=np.uint32) & 0xFFFF).astype(np.uint8)

# 2. NumPyによるボクセルダウンサンプリング (1cm単位)
voxel_size = 0.01
# 各座標をボクセルインデックスに変換
voxel_indices = np.floor(xyz / voxel_size).astype(np.int64)
# 重複を除去して、各ボクセルから最初の1点のインデックスを取得
_, unique_indices = np.unique(voxel_indices, axis=0, return_index=True)

# ダウンサンプル後のデータを抽出
down_xyz = xyz[unique_indices]
down_labels = labels[unique_indices]

print(f"Original points: {len(xyz)} -> Downsampled points: {len(down_xyz)}")

# 3. LASヘッダーの設定
header = laspy.LasHeader(point_format=7, version="1.4")
header.scales = [0.00001, 0.00001, 0.00001]
# 座標の最小値を引いて原点付近に移動（Potreeの精度維持のため）
offset_val = np.min(down_xyz, axis=0)
header.offsets = [0, 0, 0]

las = laspy.LasData(header)
# 座標の代入 (必ずダウンサンプル後のデータを使う)
las.x = (down_xyz[:, 0] - offset_val[0])
las.y = (down_xyz[:, 1] - offset_val[1])
las.z = (down_xyz[:, 2] - offset_val[2])

# 4. 属性の代入
las.classification = down_labels
las.return_number = np.ones(len(down_xyz), dtype=np.uint8)
las.number_of_returns = np.ones(len(down_xyz), dtype=np.uint8)

# 5. 色の割り当て
def get_color_255(label):
    colors = {
        0: [100, 100, 100], 10: [0, 0, 255], 11: [77, 179, 255],
        13: [0, 77, 128], 15: [255, 128, 0], 18: [128, 51, 26],
        20: [77, 77, 153], 30: [255, 51, 51], 40: [255, 0, 255],
        50: [255, 204, 0], 70: [0, 255, 0], 80: [0, 255, 255], 81: [255, 255, 0]
    }
    return colors.get(label, [255, 0, 0]) 

# 必ずダウンサンプル後のラベル(down_labels)に対して色を計算
rgb = np.array([get_color_255(l) for l in down_labels], dtype=np.uint16)
las.red = rgb[:, 0] * 256
las.green = rgb[:, 1] * 256
las.blue = rgb[:, 2] * 256

# 6. 保存
las.update_header()
las.write(output_laz)
print(f"--- 完了! ファイルを保存しました: {output_laz} ---")

<img width="1607" height="763" alt="image" src="https://github.com/user-attachments/assets/6392b888-6651-4b35-8df9-0802925f2115" />

index.html
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
	<!-- INCLUDE SETTINGS HERE -->
	
	<div class="potree_container" style="position: absolute; width: 100%; height: 100%; left: 0px; top: 0px; ">
		<div id="potree_render_area" style="background-image: url('../build/potree/resources/images/background.jpg');"></div>
		<div id="potree_sidebar_container"> </div>
	</div>
	
	<script>
	
		window.viewer = new Potree.Viewer(document.getElementById("potree_render_area"));
		
		viewer.setEDLEnabled(true);
		viewer.setFOV(60);
		viewer.setPointBudget(2_000_000);
		<!-- INCLUDE SETTINGS HERE -->
		viewer.loadSettingsFromURL();
		
		viewer.setDescription("");
		
		viewer.loadGUI(() => {
			viewer.setLanguage('en');
			$("#menu_appearance").next().show();
			$("#menu_tools").next().show();
			$("#menu_clipping").next().show();
			viewer.toggleSidebar();
		});
		
		

		Potree.loadPointCloud("./pointclouds/index/metadata.json", "index", e => {
			let scene = viewer.scene;
			let pointcloud = e.pointcloud;
			
			let material = pointcloud.material;
			material.size = 1;
			material.pointSizeType = Potree.PointSizeType.ADAPTIVE;
			material.shape = Potree.PointShape.SQUARE;
			material.activeAttributeName = "rgba";
			
			scene.addPointCloud(pointcloud);
			
			viewer.fitToScreen();
		});

		
		
	</script>
	
	
  </body>
</html>


metadata.json
{
	"version": "2.0",
	"name": "ob_downsampled_final",
	"description": "",
	"points": 5460851,
	"projection": "",
	"hierarchy": {
		"firstChunkSize": 4444, 
		"stepSize": 4, 
		"depth": 9
	},
	"offset": [0, 0, 0],
	"scale": [1.0000000000000001e-05, 1.0000000000000001e-05, 1.0000000000000001e-05],
	"spacing": 2.3240771093750001,
	"boundingBox": {
		"min": [0, 0, 0], 
		"max": [297.48187000000001, 297.48187000000001, 297.48187000000001]
	},
	"encoding": "DEFAULT",
	"attributes": [
		{
			"name": "position",
			"description": "",
			"size": 12,
			"numElements": 3,
			"elementSize": 4,
			"type": "int32",
			"min": [0, 0, 0],
			"max": [297.48187000000001, 281.08412000000004, 64.65934],
			"scale": [1, 1, 1],
			"offset": [0, 0, 0]
		},{
			"name": "intensity",
			"description": "",
			"size": 2,
			"numElements": 1,
			"elementSize": 2,
			"type": "uint16",
			"min": [0],
			"max": [0],
			"scale": [1],
			"offset": [0]
		},{
			"name": "return number",
			"description": "",
			"size": 1,
			"numElements": 1,
			"elementSize": 1,
			"type": "uint8",
			"min": [1],
			"max": [1],
			"scale": [1],
			"offset": [0]
		},{
			"name": "number of returns",
			"description": "",
			"size": 1,
			"numElements": 1,
			"elementSize": 1,
			"type": "uint8",
			"min": [1],
			"max": [1],
			"scale": [1],
			"offset": [0]
		},{
			"name": "classification flags",
			"description": "",
			"size": 1,
			"numElements": 1,
			"elementSize": 1,
			"type": "uint8",
			"min": [0],
			"max": [0],
			"scale": [1],
			"offset": [0]
		},{
			"name": "classification",
			"description": "",
			"size": 1,
			"numElements": 1,
			"elementSize": 1,
			"type": "uint8",
			"histogram": [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 37930, 0, 0, 0, 0, 0, 0, 0, 0, 0, 24085, 0, 0, 0, 0, 0, 0, 0, 0, 0, 81, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1520075, 0, 0, 0, 70463, 0, 0, 0, 352953, 21825, 228445, 147135, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 174342, 2800865, 14337, 0, 0, 0, 0, 0, 0, 0, 49423, 18892, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0], 
			"min": [10],
			"max": [81],
			"scale": [1],
			"offset": [0]
		},{
			"name": "user data",
			"description": "",
			"size": 1,
			"numElements": 1,
			"elementSize": 1,
			"type": "uint8",
			"min": [0],
			"max": [0],
			"scale": [1],
			"offset": [0]
		},{
			"name": "scan angle",
			"description": "",
			"size": 2,
			"numElements": 1,
			"elementSize": 2,
			"type": "int16",
			"min": [0],
			"max": [0],
			"scale": [1],
			"offset": [0]
		},{
			"name": "point source id",
			"description": "",
			"size": 2,
			"numElements": 1,
			"elementSize": 2,
			"type": "uint16",
			"min": [0],
			"max": [0],
			"scale": [1],
			"offset": [0]
		},{
			"name": "gps-time",
			"description": "",
			"size": 8,
			"numElements": 1,
			"elementSize": 8,
			"type": "double",
			"min": [0],
			"max": [0],
			"scale": [1],
			"offset": [0]
		},{
			"name": "rgb",
			"description": "",
			"size": 6,
			"numElements": 3,
			"elementSize": 2,
			"type": "uint16",
			"min": [0, 0, 0],
			"max": [65280, 65280, 65280],
			"scale": [1, 1, 1],
			"offset": [0, 0, 0]
		}
	]
}

<img width="1102" height="965" alt="image" src="https://github.com/user-attachments/assets/f6539268-2aff-44b7-b026-bdd0321dd890" />

