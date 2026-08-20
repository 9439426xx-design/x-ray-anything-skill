---
name: x-ray-anything
description: "Transform any user-supplied image into a classic medical blue-white X-ray stylization: a bright near-white skeleton as the brightest element, dark background, blue-white palette, semi-transparent fabric keeping its original hue. Use for people, animals, plants, objects, or scenes when the user wants a radiographic look that is aesthetic rather than diagnostic."
---

# X-Ray-anything

把任意图片变成 X 光风格。不是专业医疗用途——纯风格化。

风格化是「转换」而非「创作」：保留原图的构图、姿态、主体身份，只替换视觉语言。最终效果像给这张照片拍了一张 X 光片。

## Asset（参考图锚定质感）

参考图是稳定性的核心——它们直接定义目标质感，消除文字描述的歧义。

- Use `assets/bone-reference.png` for the ideal **bone + hair + overall** rendering: PURE WHITE bones (peak near RGB 245-255), luminous separated hair locks, clean blue-white palette, and strong rim light. Match its bone whiteness, hair luminescence, and tonal purity — NOT its hairstyle, garment, or pose.
- Use `assets/cloth-reference.png` for the ideal **clothing** rendering: a semi-transparent color film with clear structure lines and visible fabric presence. Match its opacity, fabric occlusion, and structure-line clarity — NOT its exact garment.

When the generation model supports reference-image input, always pass these reference images alongside the source photo, with the instruction: "match the hair/clothing/bone quality of the references, applied to the source's actual hairstyle and garment."

## Decision Priority

Resolve conflicts in this order:

1. Skeletal structure accuracy — bones must be anatomically correct where visible.
2. Realistic occlusion — hair and clothing genuinely block the bones behind them.
3. Density simulation — dark regions read as bright/dense, bright regions as transparent.
4. Blue-white palette consistency — no warm tones, no neutral gray.
5. Background darkness — the subject must clearly pop from the background.

## 输出规范

| 维度 | 要求 |
|------|------|
| **背景** | 纯黑 #000000（纯色底原图）。环境照→分层暗背景（2.5-3 档暗于主体） |
| **色调** | 经典医用蓝白：高光→近白，暗部→深蓝黑，中调有明显蓝偏 |
| **对比度** | 高对比。高密度→亮蓝白，低密度→透明/暗 |
| **透视感** | 密度穿透。暗色区域变亮（密度高），亮色区域变透明（密度低） |
| **边缘光** | 主体外轮廓有明显蓝白 Rim Light |
| **边缘处理** | 外部轮廓锐利，内部密度过渡柔和 |
| **画面质感** | 完全干净。无扫描线/颗粒/暗角 |
| **辨识度** | 原图可辨，X 光风格主导 |

## 执行流程

### Step 1 — 接收图片

用户提供图片。识别额外指令（「去背景」等）。

### Step 2 — 分析图片

1. **背景类型**：有背景（环境/随手拍）还是纯色底？
2. **主体类型**：人物 / 宠物动物 / 植物 / 物品 / 混合？→ 决定动态注入内容
3. **显著手持物**：识别→描述内部结构→注入 prompt
4. **人数**：单人/多人？

### Step 3 — 生成 X 光图像

将原图 + 参考图 + 以下 prompt 一起发送给生图模型。

背景句按 Step 1 指令动态选择：用户说「去背景」→ 用 `The background is pure black.`；未说 → 用 `The background is dark, much darker than the subject.`

输出尺寸按原图：读取原图宽高作为 `size` 参数（格式 `宽*高`）；若超出模型支持范围（单边 512–2048 或比例超 1:8–8:1），按原比例缩放到范围内。

```
[原图] + [bone-reference.png] + [cloth-reference.png]

The skeleton is the brightest element, near-white with a blue tint.
The background is dark, much darker than the subject; sky keeps a faint low-contrast haze.
Color palette: blue and white only, except fabric keeps its original hue, desaturated.
Soft tissue and hair stay as blue-gray mid-tones between the dark background and the white bone.
Keep the original pose, framing, and subject silhouette.
Eye sockets are empty black hollows, no eyes, no eyelashes.
```

### Step 4 — 质量自查

1. 背景是否正确（纯黑 / 分层暗背景）？
2. 主体有边缘光吗？
3. 主体比背景亮吗？
4. 背景有比主体暗部还亮的区域吗？
5. 有五官残留吗？（眼眶内有眼睛/虹膜→失败）
6. 有骨骼吗？（人物图）
7. 头发真的遮挡了头骨吗？（头发盖住的地方，头骨看不到）
8. 衣服降低了骨骼可见度吗？（约 60%，不是完全透明也不是完全遮挡）
9. 看起来像 X 光吗？

不合格就重做。

## Hard Avoids

避免：眼睛、虹膜、瞳孔、睫毛、眉毛、嘴唇、鼻子软组织、面部皮肤纹理、妆容、眼线；灰色/灰阶/脏颗粒/阴森的骨骼（骨骼必须纯净亮白，绝不能灰暗、脏污、恐怖）；稀疏的淡发丝或片状灰糊（头发必须是密集发光、分股的发丝集合）；完全实体的衣服或完全透明的衣服（衣服必须是半透明色膜）；背景比主体还亮；扫描线、颗粒、暗角、水印。

## 输出格式

```markdown
**生成图**

![X-ray](image-path)

**说明**

[一句话说明用了什么主体类型、背景模式]
```

除非用户明确要求，否则不显示生成 prompt。
