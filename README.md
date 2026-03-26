Loading checkpoint: /mnt/d/PointCloudLibrary/Open3D-ML/randlanet_s3dis_202201071330utc.pth
Model loaded successfully.
Pre-processing hierarchy...
Running pass on 196133 points...
Traceback (most recent call last):
  File "/mnt/d/PointCloudLibrary/Open3D-ML/1_room_data_convert_ply.py", line 133, in <module>
    main()
  File "/mnt/d/PointCloudLibrary/Open3D-ML/1_room_data_convert_ply.py", line 106, in main
    final_labels = predictor.predict(xyz)
  File "/mnt/d/PointCloudLibrary/Open3D-ML/1_room_data_convert_ply.py", line 82, in predict
    results = self.model(inputs)
  File "/home/winngwephyo/miniconda3/envs/randla/lib/python3.9/site-packages/torch/nn/modules/module.py", line 1511, in _wrapped_call_impl
    return self._call_impl(*args, **kwargs)
  File "/home/winngwephyo/miniconda3/envs/randla/lib/python3.9/site-packages/torch/nn/modules/module.py", line 1520, in _call_impl
    return forward_call(*args, **kwargs)
  File "/home/winngwephyo/miniconda3/envs/randla/lib/python3.9/site-packages/open3d/_ml3d/torch/models/randlanet.py", line 254, in forward
    coords_list = [arr.to(self.device) for arr in inputs['coords']]
KeyError: 'coords'

import os
import open3d as o3d
import torch
import numpy as np
import laspy
import open3d.ml.torch as ml3d

# --- 1. S3DIS 公式カラーマップ (0-255スケール) ---
S3DIS_COLORS = {
    0:  [0, 255, 0],       # ceiling: 緑
    1:  [0, 0, 255],       # floor: 青
    2:  [255, 255, 0],     # wall: 黄 (ここを修正しました)
    3:  [255, 0, 255],     # beam: 紫
    4:  [0, 255, 255],     # column: 水色
    5:  [255, 165, 0],     # window: オレンジ
    6:  [255, 0, 0],       # door: 赤
    7:  [139, 69, 19],     # table: 茶色
    8:  [222, 184, 135],   # chair: ベージュ
    9:  [255, 192, 203],   # sofa: ピンク
    10: [128, 128, 128],   # bookcase: 灰色
    11: [0, 0, 128],       # board: 濃紺
    12: [255, 255, 255]    # clutter: 白
}

class S3DISPredictor:
    def __init__(self, checkpoint_path):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        
        # モデル構成
        model_cfg = {
            "name": "RandLANet",
            "num_neighbors": 16,
            "num_layers": 5,
            "num_classes": 13,
            "dim_input": 6, 
            "dim_feature": 0,
            "dim_output": [16, 64, 128, 256, 512],
            "sub_sampling_ratio": [4, 4, 4, 4, 4],
            "ign_label": -1
        }
        
        self.model = ml3d.models.RandLANet(**model_cfg)
        self.model.device = self.device 
        self.model.fc0 = torch.nn.Linear(6, 8, bias=True)
        
        self.pipeline = ml3d.pipelines.SemanticSegmentation(model=self.model, device=str(self.device))
        print(f"Loading checkpoint: {checkpoint_path}")
        self.pipeline.load_ckpt(checkpoint_path)
        self.pipeline.model.to(self.device)
        self.pipeline.model.eval()
        print("Model loaded successfully.")

    def predict(self, xyz):
        # 1. 特徴量作成
        z = xyz[:, 2]
        z_norm = (z - np.min(z)) / (np.max(z) - np.min(z) + 1e-6)
        fake_rgb = np.tile(z_norm[:, np.newaxis], (1, 3))
        feat_6d = np.concatenate([xyz, fake_rgb], axis=1).astype(np.float32)

        data = {
            'point': xyz.astype(np.float32),
            'feat': feat_6d,
            'label': np.zeros(len(xyz), dtype=np.int32)
        }
        
        print("Pre-processing hierarchy...")
        raw_inputs = self.model.preprocess(data, {'split': 'test'})
        
        # 2. KeyError: 'coords' 回避のための辞書再構築
        inputs = {}
        inputs['features'] = torch.from_numpy(feat_6d).unsqueeze(0).to(self.device)
        
        if 'points' in raw_inputs:
            inputs['coords'] = [torch.from_numpy(p).unsqueeze(0).to(self.device) for p in raw_inputs['points']]
        
        for key in ['neighbor_indices', 'sub_indices', 'interp_indices']:
            if key in raw_inputs:
                inputs[key] = [torch.from_numpy(item).unsqueeze(0).to(self.device) for item in raw_inputs[key]]

        print(f"Running pass on {len(xyz)} points...")
        with torch.no_grad():
            results = self.model(inputs)

        if isinstance(results, torch.Tensor):
            logits = results
        else:
            logits = results.get('predict', results.get('logits', next(iter(results.values()))))
            
        pred = logits.argmax(dim=-1).squeeze().cpu().numpy()
        return pred.astype(np.uint8)

def main():
    base_path = "/mnt/d/PointCloudLibrary/Open3D-ML"
    input_path = os.path.join(base_path, "3d_point_data/cloud_bin_0.ply")
    output_path = os.path.join(base_path, "3d_point_data/cloud_bin_0_fixed.las")
    checkpoint_path = os.path.join(base_path, "randlanet_s3dis_202201071330utc.pth")

    if not os.path.exists(input_path):
        print(f"Error: {input_path} not found.")
        return

    pcd = o3d.io.read_point_cloud(input_path)
    xyz = np.asarray(pcd.points).astype(np.float32)

    predictor = S3DISPredictor(checkpoint_path)
    final_labels = predictor.predict(xyz)

    # 統計
    label_names = {0:"Ceiling", 1:"Floor", 2:"Wall", 3:"Beam", 4:"Column", 5:"Window", 
                   6:"Door", 7:"Table", 8:"Chair", 9:"Sofa", 10:"Bookcase", 11:"Board", 12:"Clutter"}
    unique, counts = np.unique(final_labels, return_counts=True)
    print("\n" + "="*15 + " S3DIS ANALYSIS REPORT " + "="*15)
    for u, c in zip(unique, counts):
        print(f"{label_names.get(u, f'ID {u}'):10} : {c:8} pts ({(c/len(xyz)*100):.2f}%)")

    # LAS保存
    print(f"\nSaving result to {output_path}...")
    header = laspy.LasHeader(point_format=3, version="1.2")
    las = laspy.LasData(header)
    las.x, las.y, las.z = xyz[:, 0], xyz[:, 1], xyz[:, 2]
    las.classification = final_labels

    out_colors = np.zeros((len(xyz), 3), dtype=np.uint16)
    for lid, color in S3DIS_COLORS.items():
        mask = (final_labels == lid)
        out_colors[mask] = [color[0]*256, color[1]*256, color[2]*256]
    
    las.red, las.green, las.blue = out_colors[:, 0], out_colors[:, 1], out_colors[:, 2]
    las.write(output_path)
    print("SUCCESS: Process completed.")

if __name__ == "__main__":
    main()
