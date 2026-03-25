import os
import numpy as np
import open3d as o3d
import open3d.ml.torch as ml3d
import torch
import torch.nn as nn
import laspy
from scipy.spatial import cKDTree

# S3DIS Labels
CEILING, FLOOR, WALL, TABLE, CHAIR, CLUTTER = 0, 1, 2, 7, 8, 12

def build_hierarchy(sub_xyz, num_layers=5, k_n=16, device="cpu"):
    current_xyz = sub_xyz
    pts_l, neigh_l, sub_l, interp_l = [], [], [], []
    for i in range(num_layers):
        n = len(current_xyz)
        pcd = o3d.geometry.PointCloud(); pcd.points = o3d.utility.Vector3dVector(current_xyz)
        tree = o3d.geometry.KDTreeFlann(pcd)
        neigh = np.zeros((n, k_n), dtype=np.int64)
        for j in range(n):
            _, idx, _ = tree.search_knn_vector_3d(pcd.points[j], k_n)
            neigh[j] = np.array(idx)
        pts_l.append(torch.FloatTensor(current_xyz).unsqueeze(0).to(device))
        neigh_l.append(torch.LongTensor(neigh).unsqueeze(0).to(device))
        s_pts = max(n // 4, 1)
        s_idx = np.random.choice(n, s_pts, replace=False)
        sub_l.append(torch.LongTensor(neigh[s_idx]).unsqueeze(0).to(device))
        sub_xyz2 = current_xyz[s_idx]
        sub_pcd = o3d.geometry.PointCloud(); sub_pcd.points = o3d.utility.Vector3dVector(sub_xyz2)
        sub_tree = o3d.geometry.KDTreeFlann(sub_pcd)
        interp = np.array([sub_tree.search_knn_vector_3d(pcd.points[j], 1)[1][0] for j in range(n)])
        interp_l.append(torch.LongTensor(interp).unsqueeze(0).to(device))
        current_xyz = sub_xyz2
    return pts_l, neigh_l, sub_l, interp_l

def estimate_normals(xyz, radius):
    pcd = o3d.geometry.PointCloud()
    pcd.points = o3d.utility.Vector3dVector(xyz)
    pcd.estimate_normals(search_param=o3d.geometry.KDTreeSearchParamHybrid(radius=float(radius), max_nn=50))
    return np.asarray(pcd.normals)

def refine_to_s3dis_v2(xyz_m, ai_labels, spacing_m):
    print("\n" + "="*20 + " FINAL LOGIC: SEPARATING FLOOR & TABLE " + "="*20)
    
    z = xyz_m[:, 2]
    normals = estimate_normals(xyz_m, radius=spacing_m * 10.0)
    nz = np.abs(normals[:, 2])

    final_labels = ai_labels.copy()

    # 1. 【壁(2)】垂直面を最優先 (nz < 0.4)
    wall_mask = (nz < 0.4)
    final_labels[wall_mask] = 2

    # 2. 【床(1)】と【天井(0)】の特定
    horizontal_mask = (nz > 0.8) & (~wall_mask)
    if np.any(horizontal_mask):
        h_z = z[horizontal_mask]
        z_min_local = np.percentile(h_z, 5) # 下位5%を床の目安にする
        z_max_local = np.percentile(h_z, 95) # 上位5%を天井の目安にする

        # 床: 最も低いエリアの水平面
        floor_mask = horizontal_mask & (z < z_min_local + 0.2)
        final_labels[floor_mask] = 1

        # 天井: 最も高いエリアの水平面
        ceiling_mask = horizontal_mask & (z > z_max_local - 0.2)
        final_labels[ceiling_mask] = 0

        # 3. 【家具(7, 8)】床から浮いている水平面だけを抽出
        # 0.5m 〜 1.2m の高さにある「独立した」水平面を家具とする
        furn_candidate = horizontal_mask & (z > z_min_local + 0.4) & (z < z_min_local + 1.2)
        
        if np.any(furn_candidate):
            f_pcd = o3d.geometry.PointCloud()
            f_pcd.points = o3d.utility.Vector3dVector(xyz_m[furn_candidate])
            # クラスタリングで「浮いている塊」を特定
            clusters = np.array(f_pcd.cluster_dbscan(eps=0.10, min_points=20))
            
            # 一旦すべての家具候補をリセット（Clutterへ）
            final_labels[furn_candidate] = 12 

            for c_id in set(clusters):
                if c_id == -1: continue
                idx = np.where(furn_candidate)[0][clusters == c_id]
                c_xyz = xyz_m[idx]
                
                # 塊の広がり（幅と奥行き）を計算
                width = np.max(c_xyz[:, 0]) - np.min(c_xyz[:, 0])
                depth = np.max(c_xyz[:, 1]) - np.min(c_xyz[:, 1])
                bbox_diag = np.sqrt(width**2 + depth**2)

                # --- 厳しい判定基準 ---
                # テーブル(7): 対角線が 0.7m 以上、かつある程度の点数がある
                if bbox_diag > 0.7 and len(idx) > 500:
                    final_labels[idx] = 7
                # 椅子(8): それより小さい独立した塊
                elif bbox_diag < 0.7:
                    final_labels[idx] = 8

    # 4. その他のカテゴリ (AIの推論を維持)
    # AIが Sofa(9), Bookcase(10), Window(5), Door(6) と言った場所は、
    # 既に確定した Floor/Wall/Ceiling 以外であればそのまま採用する
    keep_ai = np.isin(ai_labels, [3, 4, 5, 6, 9, 10, 11])
    mask = keep_ai & ~np.isin(final_labels, [0, 1, 2])
    final_labels[mask] = ai_labels[mask]

    # 結果表示
    label_names = {0:"ceiling", 1:"floor", 2:"wall", 3:"beam", 4:"column", 5:"window", 6:"door", 7:"table", 8:"chair", 9:"sofa", 10:"bookcase", 11:"board", 12:"clutter"}
    print("\n[RESULT] Final S3DIS Classification Count:")
    for i in range(13):
        count = np.sum(final_labels == i)
        if count > 0:
            print(f"  Label {i:2d} ({label_names[i]:10s}): {count:8d} points")
            
    return final_labels.astype(np.uint8)

def main():
    base = "/mnt/d/PointCloudLibrary/Open3D-ML"
    in_ply = os.path.join(base, "3d_point_data/cloud_bin_0.ply")
    out_las = os.path.join(base, "3d_point_data/cloud_bin_0_fixed.las")
    ckpt = os.path.join(base, "randlanet_s3dis_202201071330utc.pth")
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

    pcd = o3d.io.read_point_cloud(in_ply)
    raw_xyz = np.asarray(pcd.points).astype(np.float32)
    xyz_m = raw_xyz * (0.001 if np.max(raw_xyz) > 100.0 else 1.0)

    model = ml3d.models.RandLANet(num_classes=13, num_layers=5, dim_input=6, dim_features=8, dim_output=[16, 64, 128, 256, 512], num_neighbors=16).to(device)
    model.device = device
    model.fc0 = nn.Linear(6, 8).to(device)
    sd = torch.load(ckpt, map_location=device)
    model.load_state_dict(sd["model_state_dict"] if "model_state_dict" in sd else sd)
    model.eval()

    pcd_d = pcd.voxel_down_sample(0.02)
    sub_xyz = np.asarray(pcd_d.points).astype(np.float32)
    sub_xyz_norm = sub_xyz - np.mean(sub_xyz, axis=0)
    feats = np.concatenate([sub_xyz_norm, np.asarray(pcd_d.colors) if pcd.has_colors() else np.zeros_like(sub_xyz_norm)], axis=1)
    
    pts, nei, sidx, iidx = build_hierarchy(sub_xyz_norm, device=device)
    inputs = {'features': torch.FloatTensor(feats).unsqueeze(0).to(device), 'points': pts, 'coords': pts, 'neighbor_indices': nei, 'sub_idx': sidx, 'interp_idx': iidx}

    with torch.no_grad():
        out = model(inputs)
        pred = torch.argmax(out["predict_scores"] if isinstance(out, dict) else out, dim=-1).squeeze().cpu().numpy()

    tree = cKDTree(sub_xyz)
    _, idx = tree.query(xyz_m, k=1)
    
    final_labels = refine_to_s3dis_v2(xyz_m, pred[idx], 0.005)

    h = laspy.LasHeader(point_format=3, version="1.2")
    h.offsets, h.scales = np.min(raw_xyz, axis=0), [0.001, 0.001, 0.001]
    las = laspy.LasData(h)
    las.x, las.y, las.z = raw_xyz[:,0], raw_xyz[:,1], raw_xyz[:,2]
    las.classification = final_labels
    las.write(out_las)
    print(f"\n[DONE] Saved to: {out_las}")

if __name__ == "__main__":
    main()
	<!DOCTYPE html>
<html lang="en">

<head>
	<meta charset="utf-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
	<title>Potree Viewer - S3DIS Classification</title>

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
		// 1. Viewerの初期化
		window.viewer = new Potree.Viewer(document.getElementById("potree_render_area"));

		viewer.setEDLEnabled(true);
		viewer.setFOV(60);
		viewer.setPointBudget(3_000_000);
		viewer.setBackground("gradient");
		viewer.loadSettingsFromURL();

		// 2. S3DIS専用の分類定義（ラベル名と色）を設定
		// これにより、Potree標準の「Ground」などの名称が「Wall」や「Floor」に書き換わります
		viewer.setClassifications({
			0: { visible: true, name: "Ceiling", color: [1.0, 0.0, 0.0, 1.0] },
			1: { visible: true, name: "Floor", color: [0.0, 1.0, 0.0, 1.0] },
			2: { visible: true, name: "Wall", color: [0.0, 0.0, 1.0, 1.0] },
			3: { visible: true, name: "Beam", color: [1.0, 1.0, 0.0, 1.0] },
			4: { visible: true, name: "Column", color: [1.0, 0.0, 1.0, 1.0] },
			5: { visible: true, name: "Window", color: [0.0, 1.0, 1.0, 1.0] },
			6: { visible: true, name: "Door", color: [0.5, 0.5, 0.0, 1.0] },
			7: { visible: true, name: "Table", color: [0.8, 0.4, 0.1, 1.0] },
			8: { visible: true, name: "Chair", color: [0.0, 0.5, 0.5, 1.0] },
			9: { visible: true, name: "Sofa", color: [0.8, 0.8, 0.0, 1.0] },
			10: { visible: true, name: "Bookcase", color: [0.5, 0.5, 0.5, 1.0] },
			11: { visible: true, name: "Board", color: [0.2, 0.5, 0.2, 1.0] },
			12: { visible: true, name: "Clutter", color: [0.3, 0.3, 0.3, 1.0] },
			"DEFAULT": { visible: true, name: "Default", color: [0.5, 0.5, 0.5, 1.0] }
		});

		// 3. GUIのロード
		viewer.loadGUI(() => {
			viewer.setLanguage('en');
			$("#menu_appearance").next().show();
			// フィルタメニューを最初から表示する
			$("#menu_filters").next().show();
			viewer.toggleSidebar();
		});

		// 4. 点群データのロード (cloud.js を使用)
		// パス "pointclouds/index/cloud.js" は環境に合わせて適宜調整してください
		Potree.loadPointCloud("pointclouds/index/cloud.js", "index", e => {
			let pointcloud = e.pointcloud;
			let material = pointcloud.material;

			viewer.scene.addPointCloud(pointcloud);

			// Classification（分類）モードをデフォルトの表示にする
			material.activeAttributeName = "classification";
			material.size = 1;
			material.pointSizeType = Potree.PointSizeType.ADAPTIVE;
			material.shape = Potree.PointShape.SQUARE;

			viewer.fitToScreen();
		});

	</script>
</body>

</html>


	<img width="1319" height="879" alt="image" src="https://github.com/user-attachments/assets/50d5d000-5f91-4a7e-9748-714f2a4bfb5d" />
