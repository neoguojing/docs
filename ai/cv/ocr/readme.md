
```
身份证图片
     ↓
目标检测
     │
     │ 找到身份证区域
     ↓
身份证 Crop
     ↓
Alignment
     │
     │ 把身份证拉正
     ↓
标准身份证
     ↓
OCR Detection（如果 Recognition 前需要）
     │
     │ DBNet 找文字区域
     ↓
Text Region
     ↓
Text Recognition
     │
     │ CRNN / SVTR 等
     ↓
文字
     ↓
Correction
     ↓
最终字段
```

```
                 OCR Pipeline
                      │
       ┌──────────────┼──────────────┐
       ↓              ↓              ↓
  Detection       Alignment      Recognition
       │              │              │
       │              │              ├─ CRNN
       │              │              ├─ SVTR
       │              │              └─ Transformer
       │              │
       │              ├─ Keypoint
       │              ├─ Homography
       │              └─ STN
       │
       ├─ DBNet
       ├─ EAST
       ├─ CRAFT
       └─ YOLO（目标检测）

                      ↓
                  Correction
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
        Rule       Dictionary   Model
          │           │           │
          └───────────┼───────────┘
                      ↓
                  Structure
                      ↓
                  Final Result
```
