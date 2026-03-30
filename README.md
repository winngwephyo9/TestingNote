--- 修正版解析開始: cloud_bin_0.ply ---
Room Height: 3.41m

========================================
Ceiling   :     9728 pts
Floor     :      104 pts
Wall      :    67855 pts
Door      :        0 pts
Table     :        0 pts
Chair     :        0 pts
Clutter   :   118446 pts
========================================
結果を保存しました:

import os
import numpy as np
import open3d as o3d
import laspy

# S3DISカラーマップの定義
S3DIS_COLORS = np.array([
    [255, 0, 0],   # 0: Ceiling (赤)
    [0, 255, 0],   # 1: Floor (緑)
    [0, 0, 255],   # 2: Wall (青)
    [255, 255, 0], # 3: Beam (黄)
    [255, 0, 255], # 4: Column (マゼンタ)
    [0, 255, 255], # 5: Window (シアン)
    [128, 255, 255], # 6: Door (薄水色)
    [255, 128, 0],   # 7: Table (オレンジ)
    [0, 128, 255],   # 8: Chair (濃い水色)
    [128, 0, 255],   # 9: Sofa (紫)
    [255, 0, 128],   # 10: Bookcase (ピンク)
    [128, 128, 128], # 11: Board (グレー)
    [0, 0, 0]        # 12: Clutter (黒)
], dtype=np.uint8)

# クラスIDと名前の対応
LABEL_NAMES = {
    0: "Ceiling", 1: "Floor", 2: "Wall", 6: "Door", 
    7: "Table", 8: "Chair", 12: "Clutter"
}

def refined_furniture_analysis(input_ply, output_laz):
    print(f"--- 修正版解析開始: {os.path.basename(input_ply)} ---")
    
    # 1. データの読み込みと準備
    pcd = o3d.io.read_point_cloud(input_ply)
    xyz = np.asarray(pcd.points)
    num_points = len(xyz)

    # 2. 高さ軸の特定と範囲計算
    # S3DISデータは通常Z軸が高さ方向。ptp(range)が最も大きい軸を高さとする。
    up_axis = np.argmax(np.ptp(xyz, axis=0))
    h = xyz[:, up_axis]
    h_min, h_max = h.min(), h.max()
    room_height = h_max - h_min
    print(f"Room Height: {room_height:.2f}m")

    # 3. 法線ベクトルの計算
    pcd.estimate_normals(search_param=o3d.geometry.KDTreeSearchParamHybrid(radius=0.1, max_nn=30))
    normals = np.asarray(pcd.normals)

    # 4. 法線方向による分類の準備
    # Z成分が大きい点（水平面）
    is_horiz = np.abs(normals[:, up_axis]) > 0.85 
    # Z成分が小さい点（垂直面）
    is_vert = np.abs(normals[:, up_axis]) < 0.15

    # 全点をClutter(12)で初期化
    labels = np.full(num_points, 12, dtype=np.uint8)

    # 5. 天井と床の認識（高さと法線方向を組み合わせる）
    labels[(h < h_min + room_height * 0.1) & is_horiz] = 0  # Ceiling
    labels[(h > h_max - room_height * 0.1) & is_horiz] = 1  # Floor
    
    # 壁の認識（垂直面）
    labels[(labels == 12) & is_vert] = 2  # Wall

    # 6. 家具解析（椅子・テーブル・ドアの認識）
    # 床、天井、壁以外の点を対象にする
    mask_to_analyze = (labels == 12)
    if np.any(mask_to_analyze):
        rem_pcd = pcd.select_by_index(np.where(mask_to_analyze)[0])
        clusters = np.array(rem_pcd.cluster_dbscan(eps=0.06, min_points=15)) # パラメータ微調整

        for c_id in set(clusters):
            if c_id == -1: continue # ノイズを除外
            idx = np.where(mask_to_analyze)[0][clusters == c_id]
            c_pts = xyz[idx]
            c_normals = normals[idx]
            
            # バウンディングボックスの計算
            bbox = c_pts.max(axis=0) - c_pts.min(axis=0)
            horiz_diag = np.sqrt(np.sum(np.delete(bbox, up_axis)**2)) # 水平方向の対角線
            
            # 床(h_min)からの高さを計算（S3DISはZが上を向いている）
            dist_from_floor = c_pts[:, up_axis].mean() - h_min
            
            # クラスタ内の水平面の割合
            horiz_ratio = np.sum(np.abs(c_normals[:, up_axis]) > 0.8) / len(c_pts)

            # === 判定ロジックの修正 ===

            # 椅子: 小さな塊、床から少し浮いている。水平面の割合が高め。
            # テーブルより先に判定することで、椅子の誤認識を防ぐ。
            if 0.2 < dist_from_floor < 0.7 and horiz_diag < 0.8 and horiz_ratio > 0.3:
                labels[idx] = 8 # Chair
            
            # テーブル: 床から一定の高さに浮いている、水平方向の広がりが大きい、水平面の割合が高い。
            elif 0.5 < dist_from_floor < 1.1 and horiz_diag > 0.7 and horiz_ratio > 0.5:
                # 高さと水平面の条件を厳格に
                labels[idx] = 7 # Table
            
            # ドア: 壁に近い高さ、床に接している、垂直面の塊
            elif bbox[up_axis] > 1.8 and dist_from_floor < 0.4:
                # 床への接触を厳格に判定
                labels[idx] = 6 # Door

    # 7. 壁の再判定（テーブルやドアとして認識されなかった垂直面）
    # 壁の近くにある垂直面を再度壁に戻す。
    rem_mask = (labels == 12)
    if np.any(rem_mask):
        rem_pcd = pcd.select_by_index(np.where(rem_mask)[0])
        # 再度クラスタリングして大きな壁の塊を特定
        clusters = np.array(rem_pcd.cluster_dbscan(eps=0.1, min_points=50))
        for c_id in set(clusters):
            if c_id == -1: continue
            idx = np.where(rem_mask)[0][clusters == c_id]
            c_pts = xyz[idx]
            c_normals = normals[idx]
            
            bbox = c_pts.max(axis=0) - c_pts.min(axis=0)
            
            # 壁は垂直方向に大きく伸びる
            if bbox[up_axis] > 2.0 and np.mean(np.abs(c_normals[:, up_axis])) < 0.2:
                labels[idx] = 2 # Wall

    # 解析結果の出力
    print("\n" + "="*40)
    for i, name in LABEL_NAMES.items():
        count = np.sum(labels == i)
        print(f"{name:10}: {count:8} pts")
    print("="*40)

    # LAZファイルとして保存
    header = laspy.LasHeader(point_format=3, version="1.2")
    header.offsets, header.scales = np.min(xyz, axis=0), [0.001, 0.001, 0.001]
    las = laspy.LasData(header)
    las.x, las.y, las.z = xyz[:, 0], xyz[:, 1], xyz[:, 2]
    las.classification = labels
    
    # 色を付与（uint16型に変換、0-65535の範囲）
    c8 = S3DIS_COLORS[labels]
    las.red, las.green, las.blue = c8[:,0].astype(np.uint16)*256, c8[:,1].astype(np.uint16)*256, c8[:,2].astype(np.uint16)*256
    las.write(output_laz)
    print(f"結果を保存しました: {output_laz}")

if __name__ == "__main__":
    # ファイルパスを適切に設定してください
    # Windowsの場合は r"C:\path\to\your\file.ply" のように r を付けると安全です
    input_file = r"F:\PointCloudProcessing\3d_point_data\cloud_bin_0.ply" 
    output_file = r"F:\PointCloudProcessing\3d_point_data\cloud_bin_0_semantic_refined.laz"
    
    refined_furniture_analysis(input_file, output_file)



import os
import numpy as np
import open3d as o3d
import laspy

S3DIS_COLORS = np.array([
    [255, 0, 0], [0, 255, 0], [0, 0, 255], [255, 255, 0], [255, 0, 255],
    [0, 255, 255], [128, 255, 255], [255, 128, 0], [0, 128, 255],
    [128, 0, 255], [255, 0, 128], [128, 128, 128], [0, 0, 0]
], dtype=np.uint8)

LABEL_NAMES = {0:"Ceiling", 1:"Floor", 2:"Wall", 6:"Door", 7:"Table", 8:"Chair", 12:"Clutter"}

def final_refined_analysis(input_ply, output_laz):
    print(f"--- 精密解析開始: {os.path.basename(input_ply)} ---")
    pcd = o3d.io.read_point_cloud(input_ply)
    xyz = np.asarray(pcd.points)
    
    up_axis = np.argmin(np.ptp(xyz, axis=0))
    h = xyz[:, up_axis]
    h_min, h_max = h.min(), h.max()
    room_height = h_max - h_min

    pcd.estimate_normals(search_param=o3d.geometry.KDTreeSearchParamHybrid(radius=0.1, max_nn=30))
    normals = np.asarray(pcd.normals)
    is_horiz = np.abs(normals[:, up_axis]) > 0.80 
    is_vert = np.abs(normals[:, up_axis]) < 0.20

    labels = np.full(len(xyz), 12, dtype=np.uint8)

    # 1. 構造物（天井・床・壁）
    labels[(h < h_min + room_height*0.1) & is_horiz] = 0 # Ceiling
    labels[(h > h_max - room_height*0.1) & is_horiz] = 1 # Floor
    labels[(labels == 12) & is_vert] = 2 # Wall

    # 2. 家具解析（水平面を持つ物体を優先）
    rem_mask = (labels == 12)
    if np.any(rem_mask):
        rem_pcd = pcd.select_by_index(np.where(rem_mask)[0])
        clusters = np.array(rem_pcd.cluster_dbscan(eps=0.07, min_points=20))
        
        for c_id in set(clusters):
            if c_id == -1: continue
            idx = np.where(rem_mask)[0][clusters == c_id]
            c_pts = xyz[idx]
            c_normals = normals[idx]
            
            bbox = c_pts.max(axis=0) - c_pts.min(axis=0)
            horiz_diag = np.sqrt(np.sum(np.delete(bbox, up_axis)**2))
            # 床(h_max)からの高さを計算（反転考慮）
            dist_from_floor = np.abs(c_pts[:, up_axis].mean() - h_max)
            
            # 水平面の割合をチェック
            horiz_ratio = np.sum(np.abs(c_normals[:, up_axis]) > 0.8) / len(c_pts)

            # --- 判定ロジックの入れ替え ---
            # テーブル: 水平面が多く、床から浮いている一定の大きさ
            if horiz_ratio > 0.4 and 0.5 < dist_from_floor < 1.1 and horiz_diag > 0.6:
                labels[idx] = 7 # Table
            # 椅子: 小さな塊で、床から少し浮いている
            elif horiz_diag < 0.6 and dist_from_floor < 0.8:
                labels[idx] = 8 # Chair
            # ドア: 壁に近く、垂直方向が長く、かつ「床に接している」
            elif bbox[up_axis] > 1.5 and dist_from_floor < 0.5:
                # 完全に壁（2）の一部として判定されないように、少し浮いている垂直面をドアに
                labels[idx] = 6 # Door
            else:
                labels[idx] = 12

    # 結果表示
    print("\n" + "="*40)
    for i, name in LABEL_NAMES.items():
        count = np.sum(labels == i)
        print(f"{name:10}: {count:8} pts")
    print("="*40)

    # 保存処理
    header = laspy.LasHeader(point_format=3, version="1.2")
    header.offsets, header.scales = np.min(xyz, axis=0), [0.001, 0.001, 0.001]
    las = laspy.LasData(header)
    las.x, las.y, las.z = xyz[:, 0], xyz[:, 1], xyz[:, 2]
    las.classification = labels
    c8 = S3DIS_COLORS[labels]
    las.red, las.green, las.blue = c8[:,0].astype(np.uint16)*256, c8[:,1].astype(np.uint16)*256, c8[:,2].astype(np.uint16)*256
    las.write(output_laz)

if __name__ == "__main__":
    # パスは適宜書き換えてください
    input_file = "/mnt/d/PointCloudLibrary/Open3D-ML/3d_point_data/cloud_bin_0.ply"
    output_file = "/mnt/d/PointCloudLibrary/Open3D-ML/3d_point_data/cloud_bin_0_semantic.laz"
    # process_to_laz = "cloud_bin_0_complete.laz"
    final_refined_analysis(input_file, output_file)

--- 精密解析開始: cloud_bin_0.ply ---

========================================
Ceiling   :      160 pts
Floor     :    44860 pts
Wall      :   110378 pts
Door      :        0 pts
Table     :    10964 pts
Chair     :      733 pts
Clutter   :    29038 pts
========================================
色の分析について正しく分析できるように修正してください。
椅子の形なのに椅子の色が椅子に付けていません。
テーブルの色はテーブルの表面ぐらい色が付いています。壁のsomeareaにもテーブルの色が付いています。
修正してください。
<img width="1688" height="890" alt="image" src="https://github.com/user-attachments/assets/4c1d7725-fa6d-4e75-bafa-8b6369635179" />


