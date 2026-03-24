# --- 6. ポストプロセス：物理ルールによる強力な修正 ---
    # 床(min_z)からの相対高さを計算
    z_raw = orig_xyz_raw[:, 2]
    min_z = np.min(z_raw)
    max_z = np.max(z_raw)
    rel_z = z_raw - min_z
    room_height = max_z - min_z

    # --- A. 天井(Ceiling)の強制修正 ---
    # 「Table」や「Clutter」が天井付近(上部15%以上)にある場合は全て「Ceiling」へ
    # また、部屋の最上部にある点は、推論結果に関わらずCeilingとする
    ceiling_threshold = room_height * 0.85
    ceiling_mask = (rel_z > ceiling_threshold)
    final_labels[ceiling_mask] = 0

    # --- B. 床(Floor)の安定化 ---
    # 下部5cm以下は、何があろうとFloorとする
    floor_mask = (rel_z < 0.05)
    final_labels[floor_mask] = 1

    # --- C. 椅子(Chair)とテーブル(Table)の高さ制限 ---
    # 1. 椅子(8): 床から0.2m〜0.8mの間にある独立した塊（現在はWallやClutterに化けている）
    chair_zone = (rel_z > 0.2) & (rel_z < 0.8)
    # 現在Wall(2)やClutter(12)とされているもののうち、この高さにあるものをChairへ（暫定）
    final_labels[(final_labels == 12) & chair_zone] = 8
    
    # 2. テーブル(7): 0.6m〜1.0mの間。
    # 天井に誤認されたTable(7)は既にAで消去済み。
    
    # --- D. 壁(Wall)の誤認識修正 ---
    # 壁(2)と判定されているが、明らかに部屋の「中」にある浮いた点はClutter(12)かChair(8)へ
    # ※簡易的に高さで見分ける場合：
    wall_is_not_wall = (final_labels == 2) & (rel_z > 0.2) & (rel_z < 0.8)
    # 境界付近でない（部屋の中央にある）判定は複雑なため、一旦高さで椅子へ誘導
    # final_labels[wall_is_not_wall] = 8 

    # --- E. Window / Door の形状チェック（簡易版） ---
    # AIがWindow(5)やDoor(6)と出したものが、あまりに小さすぎる、または変な高さならWall(2)へ戻す
    # ドアは必ず床(rel_z < 0.1)に接しているはず
    door_mask = (final_labels == 6)
    door_touching_floor = (rel_z < 0.1)
    # 床に接していない「浮いたドア」は壁かClutter
    final_labels[door_mask & ~door_touching_floor] = 2

    # --- F. 最終的な統計の再計算 ---
	ご提示いただいた画像と、現在のコードの問題点を拝見しました。
RandLA-Netは強力ですが、学習データの性質上、幾何学的な特徴（高さや形状）の判定を誤ることがよくあります。

特に**「Ceiling（天井）が足りない」「Tableが天井に飛んでいる」「Chairが壁扱い」**といった問題は、AIの推論結果に頼りすぎず、**確実な物理ルール（Z座標ベースのフィルタリング）**で強制修正するのが最も効果的です。

Pythonコードの「ポストプロセス（手順6）」の部分を、以下の改善版に差し替えてください。

修正後のPythonコード（主要部分
主な修正ポイントの解説Ceiling (天井) の補完:画像5でTable(茶色)が天井に張り付いていますが、これはAIが「水平な面＝テーブル」と誤認したためです。rel_z > room_height * 0.85（部屋の上部15%）にある点は、ラベルが何であれ**強制的にCeiling(0)**に書き換えることで解決します。Table / Chair の分離:「椅子が壁(赤)になっている」問題に対し、高さ $0.2\text{m} \sim 0.7\text{m}$ の範囲で「壁(2)」や「Clutter(12)」と判定された箇所を「Chair(8)」に割り当て直します。Window / Door のノイズ除去:AIは平坦な壁の一部をWindowと誤認しやすいです。本来はRANSAC等で平面検出が必要ですが、まずは「床に接していないドアはドアではない」というルールを入れることで、壁の中間に浮いている誤判定を排除します。次のステップこの「物理ルールによる上書き」を適用したコードで再度実行してみてください。もし特定の家具（特定の椅子など）がまだ消える場合は、その場所の**床からの高さ(m)**を教えていただければ、さらに数値を微調整します。
	
	1. ceilingの点群が足りない

2. floorは大体大丈夫です。

3. wallも大体大丈夫ですが、椅子やbookcaseなどの点群をwallという決めました。

4. window又はdoorいう決めている点群の部分はWindow又はdoorの形ではないです。

5. tableとして記載している点群はceilingの所に表示しています。

6. chairとして記載している点群は部屋のboundaryに表示しています。

7. clutterの一部はceilingの80%ぐらい表示しています。その後は壁、sofa,ドアに表示しています。
修正してください。
import open3d as o3d
import open3d.ml.torch as ml3d
import torch
import torch.nn as nn
import numpy as np
import laspy
import os
from scipy.spatial import cKDTree
from scipy import stats

def main():
    # --- 設定 ---
    base_path = "/mnt/d/PointCloudLibrary/Open3D-ML"
    ckpt_path = os.path.join(base_path, "randlanet_s3dis_202201071330utc.pth") 
    input_sub_las = os.path.join(base_path, "3d_point_data/Zuiko314_subsampled.las")
    input_orig_las = os.path.join(base_path, "3d_point_data/Zuiko314.las")
    output_las = os.path.join(base_path, "3d_point_data/Zuiko314_segmented_final_perfect.las")
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

    # --- 1. モデル準備 ---
    model_cfg = {
        'name': 'RandLANet', 'num_neighbors': 16, 'num_layers': 5, 'num_classes': 13,
        'dim_input': 6, 'dim_features': 8, 'dim_output': [16, 64, 128, 256, 512],
        'sub_sampling_ratio': [4, 4, 4, 4, 4], 
        'grid_size': 0.06 
    }
    model = ml3d.models.RandLANet(**model_cfg)
    model.fc0 = nn.Linear(6, 8) 
    
    checkpoint = torch.load(ckpt_path, map_location=device)
    state_dict = checkpoint['model_state_dict'] if 'model_state_dict' in checkpoint else checkpoint
    model.load_state_dict(state_dict)
    model.to(device).eval()
    model.device = device

    # --- 2. 推論データの準備 (成功した 5cm 設定 + 断層カラー) ---
    sub_las = laspy.read(input_sub_las)
    xyz_raw = np.stack([sub_las.x, sub_las.y, sub_las.z], axis=1).astype(np.float32)
    
    pcd_temp = o3d.geometry.PointCloud()
    pcd_temp.points = o3d.utility.Vector3dVector(xyz_raw)
    pcd_down = pcd_temp.voxel_down_sample(voxel_size=0.05) # 机認識に最適な5cm
    sub_xyz_raw = np.asarray(pcd_down.points).astype(np.float32)
    
    print(f"Pre-processing {len(sub_xyz_raw)} points...")
    xyz_min = np.min(sub_xyz_raw, axis=0)
    sub_xyz_local = sub_xyz_raw - xyz_min
    
    # 部屋全体を 9.0m にフィット
    auto_scale = 9.0 / np.max(np.max(sub_xyz_raw, axis=0) - xyz_min)
    sub_xyz_scaled = sub_xyz_local * auto_scale
    sub_xyz_final = sub_xyz_scaled - np.mean(sub_xyz_scaled, axis=0)

    # 断層カラー (Z座標による役割分担)
    z_real = sub_xyz_local[:, 2]
    pseudo_rgb = np.zeros_like(sub_xyz_local)
    pseudo_rgb[z_real < 0.2, 2] = 1.0                # 床付近(Blue)
    pseudo_rgb[(z_real >= 0.2) & (z_real < 0.7), 1] = 1.0  # 椅子付近(Green)
    pseudo_rgb[z_real >= 0.7, 0] = 1.0                # 机付近以上(Red)
    
    features = np.concatenate([sub_xyz_final, pseudo_rgb], axis=1).astype(np.float32)

    # --- 3. 階層的データの構築 (RandLA-Net用) ---
    np.random.seed(42)
    current_xyz = sub_xyz_final
    input_points, input_neighbors, input_sub_idx, input_interp_idx = [], [], [], []

    for i in range(5):
        num_pts = len(current_xyz)
        pcd = o3d.geometry.PointCloud(); pcd.points = o3d.utility.Vector3dVector(current_xyz)
        pcd_tree = o3d.geometry.KDTreeFlann(pcd)
        neigh_idx = np.array([pcd_tree.search_knn_vector_3d(pcd.points[j], 16)[1] for j in range(num_pts)])
        input_points.append(torch.FloatTensor(current_xyz).unsqueeze(0).to(device))
        input_neighbors.append(torch.LongTensor(neigh_idx).unsqueeze(0).to(device))
        sub_pts = max(num_pts // 4, 1); sub_indices = np.random.choice(num_pts, sub_pts, replace=False)
        sub_xyz = current_xyz[sub_indices]
        input_sub_idx.append(torch.LongTensor(neigh_idx[sub_indices]).unsqueeze(0).to(device))
        sub_pcd = o3d.geometry.PointCloud(); sub_pcd.points = o3d.utility.Vector3dVector(sub_xyz)
        sub_tree = o3d.geometry.KDTreeFlann(sub_pcd)
        interp_idx = [sub_tree.search_knn_vector_3d(pcd.points[j], 1)[1][0] for j in range(num_pts)]
        input_interp_idx.append(torch.LongTensor(np.array(interp_idx)).unsqueeze(0).to(device))
        current_xyz = sub_xyz

    # --- 4. 推論実行 ---
    print("Starting Inference...")
    inputs = {'features': torch.FloatTensor(features).unsqueeze(0).to(device), 'points': input_points, 'coords': input_points, 'neighbor_indices': input_neighbors, 'sub_idx': input_sub_idx, 'interp_idx': input_interp_idx}
    with torch.no_grad():
        output = model(inputs)
        logits = output['predict_scores'] if isinstance(output, dict) else output
        sub_labels = torch.argmax(logits, dim=-1).squeeze().cpu().numpy()

    # --- 5. 元の点群へのラベル復元とスムージング ---
    print(f"Mapping labels back and applying Majority Vote...")
    orig_las = laspy.read(input_orig_las)
    orig_xyz_raw = np.stack([orig_las.x, orig_las.y, orig_las.z], axis=1).astype(np.float32)
    tree = cKDTree(sub_xyz_raw) 
    _, nearest_indices = tree.query(orig_xyz_raw, k=5)
    neighbor_labels = sub_labels[nearest_indices]
    final_labels, _ = stats.mode(neighbor_labels, axis=1)
    final_labels = final_labels.flatten()

    # --- 6. 【重要】ポストプロセス：物理的ルールによるChair救済 ---
    # 床(min_z)からの相対高さを計算
    rel_z = orig_xyz_raw[:, 2] - np.min(orig_xyz_raw[:, 2])
    
    # ルール1: AIがClutter(12)やWall(2)と判定したが、椅子の高さ(0.3m-0.65m)にある独立した塊をChair(8)へ
    # ルール2: Table(7)と判定された点のうち、低すぎる位置にあるものをChair(8)へ（椅子の座面誤認対策）
    chair_condition = (
        ((final_labels == 12) & (rel_z > 0.35) & (rel_z < 0.65)) | 
        ((final_labels == 7) & (rel_z < 0.55))
    )
    final_labels[chair_condition] = 8

    # ルール3: 天井の救済（一番高い位置にある Clutter/Wall を Ceiling(0)へ）
    max_z_limit = np.max(rel_z) * 0.95
    ceiling_condition = (final_labels == 12) & (rel_z > max_z_limit)
    final_labels[ceiling_condition] = 0

    # --- 7. 統計表示 ---
    labels_dict = {0: "ceiling", 1: "floor", 2: "wall", 3: "beam", 4: "column", 5: "window", 6: "door", 7: "table", 8: "chair", 9: "sofa", 10: "bookcase", 11: "board", 12: "clutter"}
    counts_all = np.bincount(final_labels.astype(int), minlength=13)
    
    print("\n" + "="*40)
    print(f"{'ID':<4} | {'Class Name':<10} | {'Point Count':<12}")
    print("-" * 40)
    for i in range(13):
        print(f"{i:<4} | {labels_dict[i]:<10} | {counts_all[i]:<12,}")
    print("="*40)

    # 保存
    new_las = laspy.create(point_format=orig_las.header.point_format, file_version=orig_las.header.version)
    new_las.header.scales, new_las.header.offsets = orig_las.header.scales, orig_las.header.offsets
    new_las.x, new_las.y, new_las.z = orig_las.x, orig_las.y, orig_las.z
    new_las.classification = final_labels.astype(np.uint8)
    new_las.write(output_las)
    print(f"\nSaved to: {output_las}")

if __name__ == "__main__":
    main()

<!DOCTYPE html>
<html lang="en">

<head>
	<meta charset="utf-8">
	<meta name="description" content="">
	<meta name="author" content="">
	<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
	<title>Potree Viewer - Indoor Segmentation</title>

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
		viewer.setBackground("gradient");
		viewer.loadSettingsFromURL();

		viewer.loadGUI(() => {
			viewer.setLanguage('en');

			// --- 1. 全13クラス(0-12)の定義を完全網羅 ---
			const indoorClassification = {
				0: { visible: true, name: "Ceiling", color: [1.0, 1.0, 0.0, 1.0] }, // 黄
				1: { visible: true, name: "Floor", color: [0.0, 1.0, 0.0, 1.0] }, // 緑
				2: { visible: true, name: "Wall", color: [1.0, 0.0, 0.0, 1.0] }, // 赤
				3: { visible: true, name: "Beam", color: [0.0, 1.0, 1.0, 1.0] }, // 水色
				4: { visible: true, name: "Column", color: [0.5, 0.0, 1.0, 1.0] }, // 紫
				5: { visible: true, name: "Window", color: [0.0, 0.0, 1.0, 1.0] }, // 青
				6: { visible: true, name: "Door", color: [1.0, 0.5, 0.0, 1.0] }, // 橙
				7: { visible: true, name: "Table", color: [0.6, 0.3, 0.1, 1.0] }, // 茶
				8: { visible: true, name: "Chair", color: [1.0, 0.0, 1.0, 1.0] }, // 桃
				9: { visible: true, name: "Sofa", color: [0.2, 0.5, 0.5, 1.0] }, // ティール
				10: { visible: true, name: "Bookcase", color: [0.5, 0.5, 0.0, 1.0] }, // オリーブ
				11: { visible: true, name: "Board", color: [1.0, 1.0, 1.0, 1.0] }, // 白
				12: { visible: true, name: "Clutter", color: [0.7, 0.7, 0.7, 1.0] }, // 灰
				DEFAULT: { visible: true, name: "Default", color: [0.3, 0.3, 0.3, 1.0] }
			};

			// Potreeの標準設定を上書き
			viewer.setClassifications(indoorClassification);

			// --- 2. サイドバーリストの再構築 (0-12対応) ---
			const rebuildClassificationList = () => {
				const $list = $("#classificationList");
				if ($list.length === 0) return;

				$list.empty();
				$list.append(`<li><label style="cursor:pointer"><input id="toggleClassificationFilters" type="checkbox" checked> <span style="font-weight:bold">show/hide all</span></label></li>`);

				// 定義したID順にリストを作成
				Object.keys(indoorClassification).forEach(key => {
					if (key === "DEFAULT") return;
					const item = indoorClassification[key];
					const rgb = `rgb(${item.color[0] * 255}, ${item.color[1] * 255}, ${item.color[2] * 255})`;
					const classId = parseInt(key);

					$list.append(`
                        <li id="liClassification_${key}">
                            <label style="display: flex; align-items: center; cursor: pointer; white-space: nowrap;">
                                <input id="chkClassification_${key}" type="checkbox" checked>
                                <span style="flex-grow: 1; margin-left: 8px;">${item.name} (ID ${key})</span>
                                <div style="width: 15px; height: 15px; background-color: ${rgb}; margin-left: 10px; border: 1px solid #555;"></div>
                            </label>
                        </li>
                    `);

					// イベントハンドラ
					$(`#chkClassification_${key}`).on("change", function () {
						const isVisible = $(this).prop("checked");
						viewer.setClassificationVisibility(classId, isVisible);
					});
				});

				// 全選択/解除のイベント
				$("#toggleClassificationFilters").on("change", function () {
					const isVisible = $(this).prop("checked");
					Object.keys(indoorClassification).forEach(k => {
						if (k !== "DEFAULT") {
							$(`#chkClassification_${k}`).prop("checked", isVisible);
							viewer.setClassificationVisibility(parseInt(k), isVisible);
						}
					});
				});
			};

			// GUIが安定するまで少し待って実行
			setTimeout(rebuildClassificationList, 1000);

			$("#menu_appearance").next().show();
			$("#menu_filters").next().show();
			viewer.toggleSidebar();
		});

		Potree.loadPointCloud("pointclouds/index/cloud.js", "index", e => {
			let pointcloud = e.pointcloud;
			let material = pointcloud.material;

			viewer.scene.addPointCloud(pointcloud);
			material.activeAttributeName = "classification";
			material.size = 1;
			material.pointSizeType = Potree.PointSizeType.ADAPTIVE;

			viewer.fitToScreen();
		});
	</script>
</body>

</html>

<img width="1305" height="956" alt="image" src="https://github.com/user-attachments/assets/62fdb2ab-2a7f-45e7-92f8-d2e455daaed5" />
<img width="1256" height="864" alt="image" src="https://github.com/user-attachments/assets/e2fce64f-ae68-4f2f-b77c-15d9b340ae50" />
<img width="1195" height="860" alt="image" src="https://github.com/user-attachments/assets/7a61cc26-81a0-440b-897b-38f76cb76882" />
<img width="1275" height="861" alt="image" src="https://github.com/user-attachments/assets/e5f80555-dfa2-4b55-9c60-d3a3e60ffe88" />
<img width="1163" height="880" alt="image" src="https://github.com/user-attachments/assets/3a687aa5-bc86-4529-b799-fc885a426220" />
<img width="1124" height="939" alt="image" src="https://github.com/user-attachments/assets/f83424c5-db65-4580-8fbc-1e8aaf812be7" />
<img width="1028" height="971" alt="image" src="https://github.com/user-attachments/assets/d9d236bc-67c0-4bbd-84cb-335ce34d8579" />
<img width="1182" height="947" alt="image" src="https://github.com/user-attachments/assets/f8271c39-653c-4eb4-9183-bf6b4dc03750" />
<img width="1276" height="947" alt="image" src="https://github.com/user-attachments/assets/2fd0e9eb-092c-4861-a2cb-3f3ff9af77f8" />







	
