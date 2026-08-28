
```
原始图片
   │
   ▼
① Detection 检测
   │  找到文字/票据/身份证区域
   ▼
② Alignment 对齐
   │  纠正旋转、透视、版式偏移
   ▼
③ Text Recognition 识别
   │  图片 → 字符串
   ▼
④ Correction 矫正
   │  OCR结果 → 更可信的结果
   ▼
结构化字段
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
