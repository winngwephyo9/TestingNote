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
    print("\n" + "="*20 + " ROBUST FURNITURE RECOVERY " + "="*20)
    
    x, y, z = xyz_m[:, 0], xyz_m[:, 1], xyz_m[:, 2]
    h = y 
    h_min, h_max = np.min(h), np.max(h)
    
    pcd = o3d.geometry.PointCloud()
    pcd.points = o3d.utility.Vector3dVector(xyz_m)
    pcd.estimate_normals(search_param=o3d.geometry.KDTreeSearchParamHybrid(radius=0.15, max_nn=30))
    normals = np.asarray(pcd.normals)
    nh = np.abs(normals[:, 1]) # Y軸方向の水平度

    final_labels = np.full(len(xyz_m), 12, dtype=np.uint8)

    # 1. 床と天井（厚みを少し薄くして家具を巻き込まないようにする）
    final_labels[h > h_max - 0.15] = 1 # Floor (緑)
    final_labels[h < h_min + 0.15] = 0 # Ceiling (赤)

    # 2. 壁 (垂直面)
    is_wall = (final_labels == 12) & (nh < 0.5)
    final_labels[is_wall] = 2 # Wall (青)

    # 3. 家具の判定（「床から浮いている水平面」をすべて救済）
    # 高さの制限(0.6mなど)を撤去し、純粋に「床でも天井でも壁でもない水平面」を探す
    furn_mask = (final_labels == 12) & (nh > 0.55)
    
    if np.any(furn_mask):
        f_pcd = o3d.geometry.PointCloud()
        f_pcd.points = o3d.utility.Vector3dVector(xyz_m[furn_mask])
        # 少し広めの eps=0.25 で塊を作る
        clusters = np.array(f_pcd.cluster_dbscan(eps=0.25, min_points=10))
        
        for c_id in set(clusters):
            if c_id == -1: continue
            idx = np.where(furn_mask)[0][clusters == c_id]
            c_points = xyz_m[idx]
            
            # 塊の水平サイズ(diag)を測る
            bbox = np.max(c_points, axis=0) - np.min(c_points, axis=0)
            diag = np.sqrt(bbox[0]**2 + bbox[2]**2) 
            
            # --- シンプルなサイズ判定 ---
            if diag > 0.7:
                final_labels[idx] = 7 # Table (オレンジ)
            else:
                final_labels[idx] = 8 # Chair (水色)
                
                # 椅子救済：座面の周囲 0.2m にある Wall(2) を Chair(8) に変更
                center_x, center_z = np.mean(c_points[:, 0]), np.mean(c_points[:, 2])
                around_chair = (np.abs(x - center_x) < 0.2) & \
                               (np.abs(z - center_z) < 0.2) & \
                               (final_labels == 2)
                final_labels[around_chair] = 8

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
==================== ROBUST FURNITURE RECOVERY ====================
---------- CLASSIFICATION REPORT ----------

ID  0 | Ceiling    :     2509 points ( 1.28%)

ID  1 | Floor      :    58277 points (29.71%)

ID  2 | Wall       :    64453 points (32.86%)

ID  7 | Table      :    19360 points ( 9.87%)

ID  8 | Chair      :    50511 points (25.75%)

ID 12 | Clutter    :     1023 points ( 0.52%)

potreeConverter確認してみると

1.Ceiling とfloorが大体大丈夫そうです。

2.wallも大丈夫そうですが、椅子の色がwallのsome areaにつけています。

3.tableは表面（水平）の所と椅子の座る所にtableの色が付いています。

4.tableの壁という所に壁の色が付いています。

5.椅子の色が背中の所に椅子の色が付いていますが、座る所がtableの色と脚は床の色、後他の壁の所にも椅子の色が付いています。
<img width="1702" height="976" alt="image" src="https://github.com/user-attachments/assets/0d0c0786-b12b-456b-994a-91d742ee660b" />




