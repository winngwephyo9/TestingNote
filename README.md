def refine_s3dis_full_logic(xyz_m, ai_labels, spacing_m):
    print("\n" + "="*20 + " GEOMETRY ANALYSIS " + "="*20)
    
    # 1. 点群のスケールに合わせた法線推定
    pcd = o3d.geometry.PointCloud()
    pcd.points = o3d.utility.Vector3dVector(xyz_m)
    # 半径を固定値（例: 10cm〜20cm）にする方が安定します
    pcd.estimate_normals(search_param=o3d.geometry.KDTreeSearchParamHybrid(radius=0.15, max_nn=30))
    normals = np.asarray(pcd.normals)
    
    # Z軸方向の法線強度（絶対値）
    # 1.0に近い = 真上/真下を向いている（水平面：床・天井・テーブル）
    # 0.0に近い = 横を向いている（垂直面：壁）
    nz = np.abs(normals[:, 2]) 
    
    z = xyz_m[:, 2]
    z_min, z_max = np.min(z), np.max(z)
    z_range = z_max - z_min
    
    final_labels = np.full(len(xyz_m), 12, dtype=np.uint8) # Default: Clutter

    # --- 修正ロジック ---

    # A. 床 (Floor) の判定
    # 最下部から20cm以内で、かつ法線がしっかり上を向いているもの
    is_floor = (z < z_min + 0.2) & (nz > 0.85)
    final_labels[is_floor] = 1

    # B. 壁 (Wall) の判定
    # 法線が横を向いている（nzが小さい）ものはほぼ壁
    # しきい値を 0.3 程度に下げて、少しの傾きも壁として許容する
    is_wall = (nz < 0.4) 
    final_labels[is_wall] = 2

    # C. 天井 (Ceiling) 
    # 最上部付近で、法線が下（または上）を向いている
    is_ceil = (z > z_max - 0.2) & (nz > 0.85)
    final_labels[is_ceil] = 0

    # D. 家具 (Table / Chair) の抽出
    # 床でも壁でも天井でもない、中間の高さにある水平面
    remain_mask = (final_labels == 12) & (nz > 0.7)
    
    if np.any(remain_mask):
        f_pcd = o3d.geometry.PointCloud()
        f_pcd.points = o3d.utility.Vector3dVector(xyz_m[remain_mask])
        
        # クラスタリング（epsを0.1〜0.2に絞り、椅子とテーブルを分離しやすくする）
        clusters = np.array(f_pcd.cluster_dbscan(eps=0.2, min_points=20))
        
        for c_id in set(clusters):
            if c_id == -1: continue
            idx = np.where(remain_mask)[0][clusters == c_id]
            
            c_points = xyz_m[idx]
            bbox = np.max(c_points, axis=0) - np.min(c_points, axis=0)
            diag = np.linalg.norm(bbox[:2]) # 水平方向の広がり
            
            # 高さ(Z)の平均でテーブルか椅子（座面）かを補助判定
            avg_z = np.mean(c_points[:, 2])
            rel_z = avg_z - z_min
            
            # 判定: 広がりが大きく、ある程度の高さ(0.6m以上)ならテーブル
            if diag > 0.6 and rel_z > 0.5:
                final_labels[idx] = 7 
            else:
                final_labels[idx] = 8 # Chair

    return final_labels



import os
import numpy as np
import open3d as o3d
import torch
import torch.nn as nn
import laspy
from scipy.spatial import cKDTree

# Try to import Open3D-ML, catch error if missing
try:
    import open3d.ml.torch as ml3d
except ImportError:
    print("ERROR: Open3D-ML not found. Ensure it is installed and in your PYTHONPATH.")
    exit()

# --- S3DIS Utility Functions ---
def estimate_normals(xyz, radius):
    pcd = o3d.geometry.PointCloud()
    pcd.points = o3d.utility.Vector3dVector(xyz)
    pcd.estimate_normals(search_param=o3d.geometry.KDTreeSearchParamHybrid(radius=float(radius), max_nn=50))
    return np.asarray(pcd.normals)

def refine_s3dis_full_logic(xyz_m, ai_labels, spacing_m):
    print("\n" + "="*20 + " GEOMETRY ANALYSIS " + "="*20)
    
    z = xyz_m[:, 2]
    # 法線計算の半径をさらに広げ、安定度を高める
    normals = estimate_normals(xyz_m, radius=spacing_m * 30.0)
    nz = np.abs(normals[:, 2]) # 1.0に近いほど水平面、0に近いほど垂直面
    
    final_labels = np.full(len(xyz_m), 12, dtype=np.uint8) # 初期値はClutter
    
    z_min, z_max = np.min(z), np.max(z)
    z_range = z_max - z_min
    
    # --- 1. 床 (Floor: 1) の強制判定 ---
    # 下位30%の範囲内にある「水平に近い面」はすべて床としてマーク
    # ※テーブルと混同しないよう、まずは低い位置を優先
    is_floor_candidate = (z < z_min + z_range * 0.25) & (nz > 0.6)
    final_labels[is_floor_candidate] = 1

    # --- 2. 天井 (Ceiling: 0) ---
    is_ceil = (z > z_max - z_range * 0.15) & (nz > 0.6)
    final_labels[is_ceil] = 0

    # --- 3. 壁 (Wall: 2) ---
    # 床・天井以外で、法線が横を向いている(nz < 0.6)
    is_wall = (final_labels == 12) & (nz < 0.6)
    final_labels[is_wall] = 2

    # --- 4. 家具 (Table/Chair) ---
    # まだ分類されていない(ID 12)のうち、水平な面(nz > 0.5)を抽出
    furn_mask = (final_labels == 12) & (nz > 0.5)
    
    if np.any(furn_mask):
        f_pcd = o3d.geometry.PointCloud()
        f_pcd.points = o3d.utility.Vector3dVector(xyz_m[furn_mask])
        # epsを 0.35 に広げ、バラバラの椅子を一つの塊にする
        clusters = np.array(f_pcd.cluster_dbscan(eps=0.35, min_points=10))
        
        for c_id in set(clusters):
            if c_id == -1: continue
            idx = np.where(furn_mask)[0][clusters == c_id]
            
            c_points = xyz_m[idx]
            bbox = np.max(c_points, axis=0) - np.min(c_points, axis=0)
            # 水平方向の対角線の長さ
            diag = np.linalg.norm(bbox[:2]) 
            
            # 判定: 0.7m以上の広がりがあればテーブル、それ未満なら椅子
            if diag > 0.7:
                final_labels[idx] = 7 # Table
            else:
                final_labels[idx] = 8 # Chair

    return final_labels

def main():
    # --- UPDATE THESE PATHS ---
    base_path = "/mnt/d/PointCloudLibrary/Open3D-ML"
    input_path = os.path.join(base_path, "3d_point_data/cloud_bin_0.ply")
    output_path = os.path.join(base_path, "3d_point_data/cloud_bin_0_fixed.las")
    checkpoint_path = os.path.join(base_path, "randlanet_s3dis_202201071330utc.pth")

    if not os.path.exists(input_path):
        print(f"ERROR: Input file not found at {input_path}")
        return

    # Load Data
    pcd = o3d.io.read_point_cloud(input_path)
    xyz = np.asarray(pcd.points).astype(np.float32)
    
    # Simple Inference Mock (If you want to test without the heavy model first)
    # In a real run, you would use your RandLANet model.forward() here.
    print(f"Processing {len(xyz)} points...")
    
    # Dummy AI labels for logic testing (replace with actual model output)
    dummy_ai_labels = np.full(len(xyz), 12, dtype=np.uint8) 

    # Run Refinement
    final_labels = refine_s3dis_full_logic(xyz, dummy_ai_labels, 0.01)

   # デバッグ用の統計表示
    unique, counts = np.unique(final_labels, return_counts=True)
    label_map = {
        0: "Ceiling", 1: "Floor", 2: "Wall", 3: "Beam", 4: "Column",
        5: "Window", 6: "Door", 7: "Table", 8: "Chair", 9: "Sofa",
        10: "Bookcase", 11: "Board", 12: "Clutter"
    }

    print("\n" + "-"*10 + " CLASSIFICATION REPORT " + "-"*10)
    total_pts = len(final_labels)
    for u, c in zip(unique, counts):
        name = label_map.get(u, f"Unknown({u})")
        percentage = (c / total_pts) * 100
        print(f"ID {u:2} | {name:10} : {c:8} points ({percentage:5.2f}%)")
    print("-" * 43)

    # Save to LAS
    header = laspy.LasHeader(point_format=3, version="1.2")
    las = laspy.LasData(header)
    las.x, las.y, las.z = xyz[:, 0], xyz[:, 1], xyz[:, 2]
    las.classification = final_labels
    las.write(output_path)
    print(f"SUCCESS: Saved to {output_path}")

if __name__ == "__main__":
    main()

	Processing 196133 points...

==================== GEOMETRY ANALYSIS ====================

---------- CLASSIFICATION REPORT ----------
ID  0 | Ceiling    :    19305 points ( 9.84%)
ID  1 | Floor      :     9227 points ( 4.70%)
ID  2 | Wall       :    82322 points (41.97%)
ID  7 | Table      :    53589 points (27.32%)
ID 12 | Clutter    :    31690 points (16.16%)
-------------------------------------------
1. LivingRoomCloudPointのcloud_bin_0のファイルには大体ceilingがないぐらい
現在ceilingとして表示しているのは壁の一部にceilingの色が付いています。
2. floorとして認知しているポイントは現在テーブルの一部と壁の一部にfloorの色が付いています。
3. wallとして記載しているポイントは床の全体ぐらいにwallの色が付いています。
4.tableとしてそんなに記載しているポイントは壁の全体ぐらいにtableの色が付いています。
5.椅子として形があるのに椅子の形を椅子として認識されない状態です。椅子の壁をテーブルの色が付いています。椅子の手がwallの色が付いています。

<img width="1492" height="900" alt="image" src="https://github.com/user-attachments/assets/147fb67c-8c2a-4d0e-8eb9-c8b0ff1f6e23" />
