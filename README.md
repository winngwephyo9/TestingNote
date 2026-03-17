import os

import torch

import numpy as np

import open3d.ml.torch as ml3d

import open3d.ml as ml

import laspy



def main():

    base_path = "/mnt/d/PointCloudLibrary/Open3D-ML"

    cfg_path = os.path.join(base_path, "ml3d/configs/randlanet_semantickitti.yml")

    ckpt_path = os.path.join(base_path, "randlanet_semantickitti_202201071330utc.pth")

    input_las = os.path.join(base_path, "3d_point_data/tokyo_data.las")

    output_las = os.path.join(base_path, "3d_point_data/tokyo_segmented_5.las")



    # --- Step 1: モデル準備 ---

    cfg = ml.utils.Config.load_from_file(cfg_path)

    model = ml3d.models.RandLANet(**cfg.model)

    pipeline = ml3d.pipelines.SemanticSegmentation(model)

    pipeline.load_ckpt(ckpt_path)

    print(f"--- Step 1: モデル準備 ---")

    # --- Step 2: データ読み込み ---

    las = laspy.read(input_las)

    

    # XYZを一度に取り出す (Scale/Offset適用済み)

    points = np.asarray(las.xyz, dtype=np.float32)

    

    # 推論用に中心化

    mean_offset = np.mean(points, axis=0)

    points_normalized = points - mean_offset



    input_dict = {

        'point': points_normalized,

        'feat': np.zeros((len(points), 0), dtype=np.float32), 

        'label': np.zeros((len(points),), dtype=np.int32)

    }

    print(f"--- Step 2: データ読み込み ---")

    # --- Step 3: AI推論 ---

    pipeline.model.eval()

    with torch.no_grad():

        result = pipeline.run_inference(input_dict)

    labels = result['predict_labels']

    print(f"--- Step 3: AI推論 ---")

    # --- Step 4: 結果の保存 (RecursionError対策) ---

    # ヘッダーをコピーして新しいLASを作成

    # las.header.version を維持することでPotreeとの互換性を保つ

    new_las = laspy.create(point_format=las.header.point_format, file_version=las.header.version)

    

    # ヘッダーのオフセットとスケールを元データと完全に一致させる

    new_las.header.offsets = las.header.offsets

    new_las.header.scales = las.header.scales

    

    # 座標を直接コピー (整数値として正確に引き継ぐ)

    # これによりPotreeで「形が歪む」「浮く」問題が解消されます

    new_las.x = las.x

    new_las.y = las.y

    new_las.z = las.z



    # ラベルを適用

    new_las.classification = labels.astype(np.uint8)



    # SemanticKITTI カラーマップ (16bit対応)

    sem_kitti_cmap = np.array([

        [0, 0, 0], [100, 150, 245], [100, 230, 245], [30, 60, 150], [80, 30, 180],

        [100, 80, 250], [255, 30, 30], [255, 40, 200], [150, 30, 90], [255, 0, 255],

        [255, 150, 255], [75, 0, 75], [75, 0, 175], [0, 200, 255], [50, 120, 255],

        [0, 175, 0], [0, 60, 135], [80, 240, 150], [150, 240, 255], [0, 0, 255]

    ], dtype=np.uint16)



    valid_labels = np.clip(labels, 0, len(sem_kitti_cmap) - 1)

    rgb_colors = sem_kitti_cmap[valid_labels]



    # Potree用にカラーを16bitで書き込む

    new_las.red = rgb_colors[:, 0] << 8

    new_las.green = rgb_colors[:, 1] << 8

    new_las.blue = rgb_colors[:, 2] << 8



    # 保存

    new_las.write(output_las)

    print(f"修正完了: {output_las}")



if __name__ == "__main__":

    main()



.html file

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

        

        Potree.loadPointCloud("pointclouds/index/cloud.js", "index", e => {

            let pointcloud = e.pointcloud;

            let material = pointcloud.material;

            viewer.scene.addPointCloud(pointcloud);

            //material.activeAttributeName = "rgba"; //[rgba, intensity, classification, ...]

            material.size = 1;

            material.pointSizeType = Potree.PointSizeType.ADAPTIVE;

            material.shape = Potree.PointShape.SQUARE;

            viewer.fitToScreen();

        });

        

    </script>



tokyo metroの点群データを.lasファイルで出力して、そのコードでclassificationして、web上に表示しました。その時に画像に表示通りにattributeがrgbaの時に建物の部分は緑色がついていますが、attributeがclassificationの時に建物の部分はdefaultの色が付いています。そのため、どうしたら正しくclassificationできるのか、filterのclassificationにbuilding,roadなどをselect/unselectしたら表示・非表示したいです。学習済のrandlanet_semantickittiを使ってclassificationしましたが、画像を見て、正しくclassificationしていますか。

完璧のコードで修正してください。




<img width="1765" height="985" alt="image" src="https://github.com/user-attachments/assets/79a94227-5acc-426a-a65d-a5b0b1130578" />
