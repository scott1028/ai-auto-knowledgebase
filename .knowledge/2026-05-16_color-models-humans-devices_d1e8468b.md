---
source_url: https://developer.mozilla.org/en-US/blog/color-models-humans-devices/
source_hash: d1e8468b
retrieved: 2026-05-16
title: 影像格式 (1):人眼與裝置的色彩模型 (Image formats: Color models for humans and devices)
---

# 影像格式 (1):人眼與裝置的色彩模型

> 原文發佈日期：2025-05-06,作者 Polina Gurtovaia。本系列三部曲的第一篇,後續是 `image-formats-pixels-graphics` 與 `image-formats-codecs-compression-tools`。

## TL;DR

影像格式設計的核心,不是「最有效率地存 pixel」,而是「在考慮人眼感知與螢幕特性的前提下壓縮資訊」。理解色彩模型 (RGB → XYZ → Lab/LCH/Oklab → sRGB → YUV)、gamma correction、CICP/ICC profile 等概念,才能解釋為何「同一張圖在不同裝置看起來不同」、為何 codec 內部要轉換到 YUV、為何 HDR 是個整套系統問題。

## 螢幕、眼睛、大腦

- **螢幕**:每個 pixel 發出特定波長與強度的光;大多裝置發出的不是 monochromatic「純色」,而是多波長混合。
- **眼睛**:視網膜上有 rod (低光、無色覺) 與 cone。Cone 分三種:
  - S-cone 對藍敏感
  - M-cone 對綠敏感
  - L-cone 對紅敏感
- **大腦**:不直接處理三組訊號,而是映射到三個軸:**luminance、紅-綠、藍-黃**。色彩是知覺,不是物理量。
- 人眼對 **patterns / edges 敏感**,對高空間頻率細節較不敏感;這直接影響 codec 怎麼選擇要壓掉什麼。

> **Luminance ≠ brightness**:luminance 是物理量 (cd/m²);brightness 是知覺量。

## CIE 與色彩模型的演進

CIE (International Commission on Illumination) 多年來建立各種色彩模型來代表「平均觀察者」的感知:

### RGB → XYZ
RGB 的 color-matching function 中紅色靈敏度會出現「負值」,所以引入 XYZ。從 RGB 到 XYZ 的線性轉換:

```
Y = 0.2126·R + 0.7152·G + 0.0722·B
```

> 注意綠色係數最大 —— 人眼對綠光最敏感。Y 是 perceptual brightness;X / Z 負責色彩。**亮度與色彩分離**是後續一切處理的關鍵。

### chromaticity diagram & color space

把 X、Z 用 X+Y+Z 正規化丟掉 Y,就得到馬蹄形 chromaticity diagram —— 所有人眼可見的色彩集合。XYZ 缺點:色彩距離跟人眼感知不對應、許多 XYZ 值對應不到可見色,編碼浪費 bit。

但 XYZ 對「定義裝置色域」非常有用:量測裝置顯示 `(1,0,0)`、`(0,1,0)`、`(0,0,1)` 與白點的 XYZ → 得到 RGB primaries。**sRGB** 就是針對「標準裝置」定義的一組係數。

sRGB ↔ XYZ 的線性轉換係數:

```
[Rlinear]   [ 3.2406  -1.5372  -0.4986 ]   [X]
[Glinear] = [-0.9689   1.8758   0.0415 ] · [Y]
[Blinear]   [ 0.0557  -0.2040   1.0570 ]   [Z]

[X]   [0.4124  0.3576  0.1805]   [Rlinear]
[Y] = [0.2126  0.7152  0.0722] · [Glinear]
[Z]   [0.0193  0.1192  0.9505]   [Blinear]
```

> sRGB 涵蓋不到人眼可見的所有色 —— 色彩三角形外的綠色區是 sRGB 無法表達的。

## 知覺均勻的色彩空間:Lab / LCH / Oklab

### CIE L\*a\*b\*

從 XYZ 經非線性轉換而來:
- `L*`:亮度 (來自 Y)
- `a*`:紅-綠軸(可正可負;同時強紅+強綠的訊號不存在)
- `b*`:藍-黃軸

**Opponent-based color space** —— 把大腦處理機制編進色彩空間。Lab 中兩色的距離≈知覺距離。範例:

| 顏色 | sRGB | Lab (L*, a*, b*) |
|---|---|---|
| Red | 1, 0, 0 | 53.23, 80.11, 67.22 |
| Green | 0, 1, 0 | 87.74, -86.18, 83.19 |
| Blue | 0, 0, 1 | 32.30, 79.20, -107.85 |
| Yellow | 1, 1, 0 | 97.14, -21.55, 94.49 |

Lab 與裝置無關,但 Lab 需要 white point (常用 D65) 來模擬「色覺適應 (chromatic adaptation)」。

### LCH:更直覺的 Lab

把 `a*, b*` 改成極座標:`L`、`C` (chroma = √(a*² + b*²))、`H` (hue 角度)。例:

```
Lab: 63.76,  29.08, 66.2
Lab: 63.76, -66.2,  29.08
→
LCH: 63.76, 72.31,  66.29
LCH: 63.76, 72.31, 156.29
```

兩色相同亮度、相同 chroma,只差 hue 90°。CSS:

```css
.first  { background-color: lch(63.76 72.31 66.29); }
.second { background-color: lch(63.76 72.31 calc(66.29 + 90)); }
```

### Oklab / Oklch (2020)

CSS Color Module Level 4 同時定義 CIE L\*a\*b\* 與 **Oklab**。Oklab 改進:

- 對藍色 hue 更均勻。
- 數值計算更穩定 (對機器友善)。
- 從 sRGB 推導,對開發者直覺。
- 對應的 polar form 是 **Oklch**。

## 影像/影片實務上的色彩空間:sRGB + YUV

XYZ / Lab 太「貴」,影像格式幾乎都用 sRGB,GPU 也針對 sRGB 最佳化。但 RGB 不利處理 —— **codec 內部會轉成 YUV**:

```
Y = 0.299·R + 0.587·G + 0.114·B
U = (B - Y) / 1.402
V = (R - Y) / 1.772
```

YUV 的核心好處:**亮度 (Y) 與色差 (U, V) 分離**。人眼對亮度遠比色彩敏感,因此 codec 通常對 Y 與 UV 採用不同的壓縮策略(例如對 UV 做 subsampling 4:2:0)。

> 命名混亂:YUV、YCbCr、Y'CbCr 常被混用;但 Y 與 Y' 不同(後者是 gamma corrected)。Lab 是知覺均勻的,YUV 不是。

## Gamma correction —— 為何 128 不是中灰

人類感知亮度是**對數的**(Weber-Fechner law),裝置卻是線性輸出。

範例:純線性的 0→255 灰階漸層,中間實際感知並非 (128,128,128) 而比較像 (180,180,180)。

CSS 中的 RGB 值是**已經做過 gamma correction 的值**。要從 linear 轉成 sRGB:

```
C_sRGB = 12.92 · C_linear                          if C_linear ≤ 0.0031308
       = 1.055 · C_linear^(1/2.4) - 0.055          otherwise
```

效果:暗部得到更多 bit (符合人眼對暗部更敏感)。Y' = gamma corrected Y。對應公式稱為 **transfer function**,色彩空間相關。

## 從相機到螢幕:CICP 與 ICC profile

「同一張 RGB 值在不同裝置看起來不同」 → 影像需要額外資訊讓 decoder 還原。裝置色彩空間由 **3 件事** 決定:

1. **Color primaries (CP)**:三原色點
2. **White point**
3. **Transfer function (TC)**:gamma correction 等非線性轉換

AVIF / 多數影像格式採用兩種機制:

- **ICC profile** —— 由製作影像的裝置寫入。內含精確的 primaries、transfer function、white point,可能含查表預先計算。
- **CICP (Coding-independent code points)** —— 影片較常用,比 ICC 簡單。

CICP 由 3 個值組成,索引 H.273 標準的對應表:

| 縮寫 | 意義 |
|---|---|
| **MC** | RGB ↔ YUV 的轉換係數 |
| **TC** | 非線性 transfer function |
| **CP** | RGB → XYZ 的色域 primaries |

例如 AVIF 內 CICP=2/2/6 表示 CP 未指定、TC 未指定、MC 用標準 RGB→YUV (與 JPEG 同)。如有 ICC profile,CP/TC 取自 profile,MC 仍取自 CICP。

decode 後可能經 XYZ 過中間色彩空間再轉成裝置色彩空間,結果送 GPU 顯示。

## HDR

人眼可感知的色彩與亮度範圍 **遠大** 於一般裝置可記錄/顯示的範圍 (SDR)。要拿到 HDR 結果需 4 件事齊備:

1. 能擷取廣域色彩/亮度的裝置 (專業相機、部分手機)
2. 能高效儲存此範圍的檔案格式
3. 能正確 decode 的軟體 (現代瀏覽器在部分 OS 支援)
4. 能顯示此範圍的顯示器

任一缺項,結果就是 SDR。Tone mapping 把 HDR 壓回 SDR,但「整張統一套用」會丟掉細節。

**Gain map** 是改進:嵌在影像中的灰階 map,每個 pixel 的亮度乘上 gain map 對應值。JPEG / AVIF 都支援但仍未標準化、實驗中。

實務上更直接的方案是用更高 bit depth (AVIF 支援 10 / 12 bit per channel)。

## 小結

- 色彩空間有 device-specific (sRGB) 與 device-independent (XYZ / Lab) 之別。
- 為一致顯示,需在 device-specific ↔ universal 間互轉,並做 gamma correction。
- 「亮度 / 色彩分離」(YUV、Lab) 對影像壓縮極為重要 —— 下一篇會深入這點。

## 個人筆記

- 看完這篇再去寫 CSS 顏色就更能判斷何時用 `lch()`、`oklch()` 而不只是 `rgb()`。
- 「為何 codec 都先轉 YUV」這個老問題,根本原因是「人眼對亮度更敏感,所以對色差可以 subsample」。
- 在 web 上實作 HDR 需確認:OS、瀏覽器、顯示器、影像格式四件齊備,缺一就回到 SDR。
