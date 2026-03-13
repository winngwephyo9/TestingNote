import numpy as np
import laspy

# --- パス設定などは変更なし ---

# 4. Classificationの変換マッピング (SemanticKITTI -> ASPRS LAS 1.4)
# Potreeの標準表示に合わせるための変換表
def map_kitti_to_asprs(labels):
    asprs_labels = np.zeros_like(labels)
    
    mapping = {
        0: 0,    # unclassified
        10: 1,   # car -> Processed
        11: 1,   # bicycle
        13: 1,   # bus
        15: 1,   # motorcycle
        18: 1,   # truck
        20: 1,   # other-vehicle
        30: 1,   # person
        31: 1,   # bicyclist
        32: 1,   # motorcyclist
        40: 11,  # road -> Road (ASPRS 11)
        44: 11,  # sidewalk
        48: 11,  # other-ground
        49: 11,  # terrain
        50: 6,   # building -> Building (ASPRS 6)
        51: 6,   # wall
        52: 6,   # fence
        70: 5,   # vegetation -> High Vegetation (ASPRS 5)
        71: 4,   # trunk -> Medium Veg
        72: 3,   # terrain -> Low Veg
        80: 1,   # pole
        81: 1,   # traffic-sign
    }
    
    for kitti_id, asprs_id in mapping.items():
        asprs_labels[labels == kitti_id] = asprs_id
    return asprs_labels

# --- メイン処理 ---
# 1.4 Format 6 を使用 (高精度・多機能)
header = laspy.LasHeader(point_format=6, version="1.4")
header.scales = [0.001, 0.001, 0.001]
header.offsets = np.min(down_xyz, axis=0)

las = laspy.LasData(header)
las.x = down_xyz[:, 0]
las.y = down_xyz[:, 1]
las.z = down_xyz[:, 2]

# ここでラベルを変換して代入
las.classification = map_kitti_to_asprs(down_labels).astype(np.uint8)

# 5. 色の割り当て (見た目をCloudCompareに合わせるためにRGBも保存)
cmap = {
    0: [100, 100, 100], 10: [0, 0, 255], 11: [77, 179, 255],
    40: [255, 0, 255], 50: [255, 204, 0], 70: [0, 255, 0], # 道路は紫、建物は黄色など
}
rgb = np.array([cmap.get(l, [255, 0, 0]) for l in down_labels], dtype=np.uint16)

las.red = rgb[:, 0] * 256
las.green = rgb[:, 1] * 256
las.blue = rgb[:, 2] * 256

las.update_header()
las.write(output_laz)




import numpy as np
import laspy

# ... (読み込み・ダウンサンプリング部分は同じ) ...

# 3. LASヘッダーの設定 (1.4を指定して0-255のIDを許可する)
header = laspy.LasHeader(point_format=6, version="1.4") # Format 6は基本的な1.4形式
header.scales = [0.001, 0.001, 0.001]
header.offsets = np.min(down_xyz, axis=0)

las = laspy.LasData(header)
las.x = down_xyz[:, 0]
las.y = down_xyz[:, 1]
las.z = down_xyz[:, 2]

# 4. Classificationの修正（ここが重要）
# Potreeの標準フィルタに邪魔されないよう、一旦すべて「0 (Created, never classified)」
# もしくは「1 (Unclassified)」に設定し、色はRGBで表現するのが最も安全です。
# もしPotreeのサイドメニューで名前（Roadなど）を出したい場合は、ここでマッピングを行います。
las.classification = down_labels.astype(np.uint8) 

# 5. 色の割り当て (RGBを優先させる)
def get_color_map():
    return {
        0: [100, 100, 100], 10: [0, 0, 255], 11: [77, 179, 255],
        13: [0, 77, 128], 15: [255, 128, 0], 18: [128, 51, 26],
        20: [77, 77, 153], 30: [255, 51, 51], 40: [255, 0, 255],
        50: [255, 204, 0], 70: [0, 255, 0], 80: [0, 255, 255], 81: [255, 255, 0]
    }

cmap = get_color_map()
rgb = np.array([cmap.get(l, [255, 0, 0]) for l in down_labels], dtype=np.float64)

# 16bitカラーへ変換
las.red = (rgb[:, 0] * 256).astype(np.uint16)
las.green = (rgb[:, 1] * 256).astype(np.uint16)
las.blue = (rgb[:, 2] * 256).astype(np.uint16)

las.update_header()
las.write(output_laz)



146000,283000,340000,696000





Traceback (most recent call last):

  File "/mnt/d/PointCloudLIbrary/Open3D-ML/1_ob_convert_to_laz.py", line 39, in <module>

    las.classification = down_labels

  File "/home/winngwephyo/miniconda3/envs/randla/lib/python3.9/site-packages/laspy/lasdata.py", line 403, in __setattr__

    self[key] = value

  File "/home/winngwephyo/miniconda3/envs/randla/lib/python3.9/site-packages/laspy/lasdata.py", line 435, in __setitem__

    self.points[key] = value

  File "/home/winngwephyo/miniconda3/envs/randla/lib/python3.9/site-packages/laspy/point/record.py", line 223, in __setitem__

    self[key][:] = value

  File "/home/winngwephyo/miniconda3/envs/randla/lib/python3.9/site-packages/laspy/point/dims.py", line 684, in __setitem__

    raise OverflowError(

OverflowError: value 81 is greater than allowed (max: 31)

このエラーが発生していますので、headerを変更しました。

header = laspy.LasHeader(point_format=7, version="1.4")

PotreeConverterで交換してweb上に確認してみるとこの形になっています。

classificationにはbuilding, low point(noise), keypoint, overlap, defaultしかない

pointの数もpoint budgetから10,000,000まで増えましたが、形が変わらないです。import numpy as np
import laspy
import os

# --- パス設定 ---
bin_path = "/mnt/d/PointCloudLibrary/data/ob_testing_point_cloud_data/dataset/sequences/00/velodyne/000000.bin"
label_path = "/mnt/d/PointCloudLibrary/Open3D-ML/test/sequences/00/predictions/000000.label"
output_laz = "/mnt/d/PointCloudLibrary/Open3D-ML/3d_point_data/ob_downsampled_final_11.laz"

# 1. データの読み込み
points = np.fromfile(bin_path, dtype=np.float32).reshape(-1, 4)
xyz = points[:, :3].astype(np.float64)
# SemanticKITTIのラベルを取得
labels = (np.fromfile(label_path, dtype=np.uint32) & 0xFFFF).astype(np.uint8)

# 2. ボクセルダウンサンプリング (0.02 = 2cm)
voxel_size = 0.02 
voxel_indices = np.floor(xyz / voxel_size).astype(np.int64)
_, unique_indices = np.unique(voxel_indices, axis=0, return_index=True)

down_xyz = xyz[unique_indices]
down_labels = labels[unique_indices]

# 3. LASヘッダーの設定
# Point format 2 または 3 を推奨（RGB+Classificationを確実に保持するため）
header = laspy.LasHeader(point_format=3, version="1.2")
header.offsets = np.min(down_xyz, axis=0)
header.scales = [0.001, 0.001, 0.001]

las = laspy.LasData(header)
las.x = down_xyz[:, 0]
las.y = down_xyz[:, 1]
las.z = down_xyz[:, 2]

# 4. Classificationの修正 (ASPRS標準にマッピングするか、そのまま入れるか)
# Potreeで"Road"などを表示させるには、SemanticKITTIのIDをASPRS準拠に書き換えるのが無難です。
# 例: SemanticKITTI 40(Road) -> ASPRS 11(Road/Circuit) または 13等
# ここでは一旦そのまま代入しますが、Potreeのサイドメニューで全クラスにチェックを入れてください。
las.classification = down_labels

# 5. 色の割り当て
def get_color_map():
    # SemanticKITTIのラベル: [R, G, B]
    return {
        0: [100, 100, 100], 10: [0, 0, 255], 11: [77, 179, 255],
        13: [0, 77, 128], 15: [255, 128, 0], 18: [128, 51, 26],
        20: [77, 77, 153], 30: [255, 51, 51], 40: [255, 0, 255], # Road
        50: [255, 204, 0], 70: [0, 255, 0], 80: [0, 255, 255], 81: [255, 255, 0]
    }

cmap = get_color_map()
# 効率的なマッピング
colors = np.array([cmap.get(l, [255, 0, 0]) for l in down_labels])

# laspy 2.x系でのRGB代入 (0-65535にスケーリング)
las.red = (colors[:, 0] * 256).astype(np.uint16)
las.green = (colors[:, 1] * 256).astype(np.uint16)
las.blue = (colors[:, 2] * 256).astype(np.uint16)

# 統計情報を更新して保存
las.update_header()
las.write(output_laz)

print(f"Done: {output_laz}")



修正のポイントとアドバイス
1. Potreeでの表示が崩れる理由
画像3枚目（ブラウザ）を見ると、点が非常にまばらで、一部のオブジェクト（車や壁らしきもの）の破片しか見えていません。これは以下の理由が考えられます。
• Classificationフィルタ: Potreeの左メニューで classification のチェックボックスが「Unclassified」や「Ground」だけに限定されていませんか？「show/hide all」にチェックを入れてみてください。
• Point Budget: ブラウザのスペックにより、表示点数制限（Point Budget）が低いと、変換後のデータの一部しか描画されないことがあります。設定で 2,000,000 以上に上げてください。
2. Classification（クラス分け）について
SemanticKITTIのラベル番号（例: 40は道路）は、LAS標準（ASPRS）の番号とは異なります。
• 対策: las.classification に入れる値を、以下の表のように変換すると、Web上のPotreeのサイドメニューで正しく名前（Ground, Building等）が表示されます。


