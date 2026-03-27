import os
import numpy as np
import open3d as o3d
import torch
import laspy
import open3d.ml.torch as ml3d
from scipy.spatial import cKDTree

class S3DISPredictor:
    def __init__(self, checkpoint_path):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        model_cfg = {
            "name": "RandLANet",
            "num_neighbors": 16,
            "num_layers": 5,
            "num_classes": 13,
            "dim_input": 6, 
            "dim_feature": 8,
            "dim_output": [16, 64, 128, 256, 512],
            "sub_sampling_ratio": [4, 4, 4, 4, 4],
            "ign_label": -1
        }
        
        self.model = ml3d.models.RandLANet(**model_cfg)
        self.model.fc0 = torch.nn.Linear(6, 8, bias=True)
        self.model.device = self.device
        
        print(f"Loading model: {checkpoint_path}")
        ckpt = torch.load(checkpoint_path, map_location=self.device)
        state_dict = ckpt['model_state_dict'] if 'model_state_dict' in ckpt else ckpt
        self.model.load_state_dict(state_dict)
        self.model.to(self.device).eval()

    def predict(self, pcd):
        xyz = np.asarray(pcd.points).astype(np.float32)
        
        # 1. 前処理 (まずOpen3D-MLにダウンサンプリング等を任せる)
        # ラベルはダミーを渡します
        data = {
            'point': xyz, 
            'feat': np.zeros((len(xyz), 6), dtype=np.float32), # 6次元の枠だけ作る
            'label': np.zeros(len(xyz), dtype=np.int32)
        }
        
        proc = self.model.preprocess(data, {'split': 'test'})
        inputs = self.model.transform(proc, {'split': 'test'})

        # 2. サンプリングされた後の形状に合わせて 6次元データを注入する
        # inputs['coords'][0] にはサンプリング後の座標が入っています
        sampled_xyz = inputs['coords'][0] # [N_sampled, 3]
        
        # サンプリング後の座標から擬似色を作る (これで次元を6に合わせる)
        z_s = sampled_xyz[:, 2]
        z_norm_s = (z_s - z_s.min()) / (z_s.max() - z_s.min() + 1e-6)
        pseudo_rgb_s = np.tile(z_norm_s[:, None], (1, 3))
        
        # [N_sampled, 6] を作成
        sampled_features = np.concatenate([sampled_xyz, pseudo_rgb_s], axis=1).astype(np.float32)
        
        # 3. テンソル化してモデルへ
        model_inputs = {
            'features': torch.from_numpy(sampled_features).unsqueeze(0).to(self.device),
            'coords': [torch.from_numpy(c).unsqueeze(0).to(self.device) for c in inputs['coords']],
            'neighbor_indices': [torch.from_numpy(n).unsqueeze(0).to(self.device) for n in inputs['neighbor_indices']],
            'sub_idx': [torch.from_numpy(s).unsqueeze(0).to(self.device) for s in inputs['sub_idx']],
            'interp_idx': [torch.from_numpy(i).unsqueeze(0).to(self.device) for i in inputs['interp_idx']]
        }

        print(f"推論開始: サンプリング後形状 {model_inputs['features'].shape}")
        
        with torch.no_grad():
            results = self.model(model_inputs)
            
        logits = results['predict'] if isinstance(results, dict) else results
        pred_sampled = logits.squeeze(0).argmax(dim=-1).cpu().numpy()
        
        # 4. 元の点数 (405741点) にラベルを戻す
        # 1層目のサンプリング後の座標を使って最近傍探索
        tree = cKDTree(inputs['coords'][0])
        _, idx = tree.query(xyz)
        return pred_sampled[idx].astype(np.uint8)

def save_labeled_las(xyz, labels, output_path):
    s3dis_colors = np.array([
        [255, 0, 0], [0, 255, 0], [0, 0, 255], [255, 255, 0], [255, 0, 255], 
        [0, 255, 255], [128, 255, 255], [255, 128, 0], [0, 128, 255], 
        [128, 0, 255], [255, 0, 128], [128, 128, 128], [0, 0, 0]
    ], dtype=np.uint8)

    header = laspy.LasHeader(point_format=3, version="1.2")
    las = laspy.LasData(header)
    las.x, las.y, las.z = xyz.T
    las.classification = labels 
    
    clamped_labels = np.clip(labels, 0, 12)
    rgb = s3dis_colors[clamped_labels]
    las.red, las.green, las.blue = rgb[:, 0]*256, rgb[:, 1]*256, rgb[:, 2]*256
    las.write(output_path)
    print(f"結果を保存しました: {output_path}")

def main():
    base_dir = "/mnt/d/PointCloudLibrary/Open3D-ML"
    input_path = os.path.join(base_dir, "3d_point_data/cloud_bin_0.ply")
    model_path = os.path.join(base_dir, "randlanet_s3dis_202201071330utc.pth")
    output_path = os.path.join(base_dir, "3d_point_data/s3dis_analyzed_room_0.las")

    pcd = o3d.io.read_point_cloud(input_path)
    predictor = S3DISPredictor(model_path)
    labels = predictor.predict(pcd)

    save_labeled_las(np.asarray(pcd.points), labels, output_path)

if __name__ == "__main__":
    main()

<!DOCTYPE html>
<html lang="en">

<head>
	<meta charset="utf-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
	<title>Potree S3DIS AI Analyzer - Complete</title>

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
		/**
		 * 1. S3DIS カテゴリと色の完全定義
		 */
		const s3dis_cfg = {
			0: { visible: true, name: "Ceiling", color: [1.0, 0.0, 0.0, 1.0] }, // 赤
			1: { visible: true, name: "Floor", color: [0.0, 1.0, 0.0, 1.0] }, // 緑
			2: { visible: true, name: "Wall", color: [0.0, 0.0, 1.0, 1.0] }, // 青
			3: { visible: true, name: "Beam", color: [1.0, 1.0, 0.0, 1.0] }, // 黄
			4: { visible: true, name: "Column", color: [1.0, 0.0, 1.0, 1.0] }, // 紫
			5: { visible: true, name: "Window", color: [0.0, 1.0, 1.0, 1.0] }, // 水
			6: { visible: true, name: "Door", color: [0.5, 0.5, 0.0, 1.0] }, // オリーブ
			7: { visible: true, name: "Table", color: [1.0, 0.6, 0.0, 1.0] }, // 橙
			8: { visible: true, name: "Chair", color: [0.0, 0.6, 0.6, 1.0] }, // ティール
			9: { visible: true, name: "Sofa", color: [0.8, 0.2, 0.5, 1.0] }, // ピンク
			10: { visible: true, name: "Bookcase", color: [0.5, 0.3, 0.0, 1.0] }, // 茶
			11: { visible: true, name: "Board", color: [0.2, 0.5, 0.2, 1.0] }, // 深緑
			12: { visible: true, name: "Clutter", color: [0.4, 0.4, 0.4, 1.0] }  // 灰
		};

		// --- エラー回避処理: 全256スロットをデフォルト色で埋める ---
		const full_classification = {};
		for (let i = 0; i <= 255; i++) {
			full_classification[i] = { visible: true, name: `Unclassified ${i}`, color: [0.5, 0.5, 0.5, 1.0] };
		}
		// S3DISの定義を上書きマージ
		Object.assign(full_classification, s3dis_cfg);
		Potree.Classification = full_classification;

		/**
		 * 2. ビューアーの初期化
		 */
		window.viewer = new Potree.Viewer(document.getElementById("potree_render_area"));

		viewer.setEDLEnabled(true);
		viewer.setFOV(60);
		viewer.setPointBudget(3_000_000);
		viewer.setBackground("gradient");
		viewer.setDescription("S3DIS AI Scene Analysis with Camera Telemetry");
		viewer.loadSettingsFromURL();

		/**
		 * 3. カメラ座標のリアルタイム・ログ
		 */
		viewer.addEventListener("update", () => {
			const pos = viewer.scene.view.position;
			const target = viewer.scene.view.getPivot();

			// 1秒ごとにコンソールへ出力
			if (!window.lastLogTime || (Date.now() - window.lastLogTime > 1000)) {
				console.clear();
				console.log("%c--- S3DIS ANALYZER LOG ---", "color: #3498db; font-weight: bold; font-size: 1.2em;");
				console.log(`[CAMERA POSITION] X: ${pos.x.toFixed(3)}, Y: ${pos.y.toFixed(3)}, Z: ${pos.z.toFixed(3)}`);
				console.log(`[CAMERA TARGET  ] X: ${target.x.toFixed(3)}, Y: ${target.y.toFixed(3)}, Z: ${target.z.toFixed(3)}`);
				console.log(`[POINT BUDGET   ] ${viewer.getPointBudget().toLocaleString()} points`);
				window.lastLogTime = Date.now();
			}
		});

		/**
		 * 4. GUIのロード
		 */
		viewer.loadGUI(() => {
			viewer.setLanguage('en');
			viewer.setClassifications(s3dis_cfg);
			$("#menu_appearance").next().show();
			$("#menu_filters").next().show();
			viewer.toggleSidebar();
		});

		/**
		 * 5. 点群のロードとマテリアル設定
		 */
		Potree.loadPointCloud("pointclouds/index/cloud.js", "index", e => {
			let pointcloud = e.pointcloud;
			let material = pointcloud.material;

			viewer.scene.addPointCloud(pointcloud);

			// 分類(Classification)表示に固定
			material.activeAttributeName = "classification";

			// マテリアル側の色の配列を全スロット分作成 (エラーの原因 recomputeClassification を防ぐ)
			material.classification = {};
			for (let id in full_classification) {
				const c = full_classification[id].color;
				material.classification[id] = new THREE.Vector4(c[0], c[1], c[2], c[3]);
			}

			material.needsUpdate = true;

			// 見た目の設定
			material.size = 1.0;
			material.pointSizeType = Potree.PointSizeType.ADAPTIVE;
			material.shape = Potree.PointShape.SQUARE;

			console.log("%cPoint Cloud Loaded Successfully.", "color: #2ecc71; font-weight: bold;");

			viewer.fitToScreen();
		});

	</script>
</body>

</html>

<img width="1633" height="926" alt="image" src="https://github.com/user-attachments/assets/70cb80c3-78cf-402b-b64d-9886d774d795" />
<img width="1480" height="787" alt="image" src="https://github.com/user-attachments/assets/8b934d49-de35-481a-a8d3-734d4e834383" />
全体としてはBookcaseとDoorの色が混ぜて付けています。
wallのpointsも床のところに色が付けています。
categoryごとに正しく色を付けてください。

