---
source_url: https://developer.mozilla.org/en-US/blog/image-formats-pixels-graphics/
source_hash: c9f9f3d3
retrieved: 2026-05-16
title: 影像格式 (2):從 encoder 到 decoder 的 pixel 流程 (Image formats: Pixel data from encoders to decoders)
---

# 影像格式 (2):從 encoder 到 decoder 的 pixel 流程

> 原文發佈日期：2025-08-04,作者 Polina Gurtovaia。系列第二篇,接續 `color-models-humans-devices`,下一篇是 `image-formats-codecs-compression-tools`。

## TL;DR

影像 codec 的整體流程:**spatial locality 切塊 → prediction (預測) → block transform (DCT) → quantization → entropy coding → post-processing**。每一步都針對「人眼對什麼敏感、對什麼遲鈍」做取捨。理解這套流程才能解釋為什麼純色 100×100 是 500 byte、隨機雜訊 100×100 卻是 35 KB。

## Pixel vs Device Pixel

- **Image pixel**:格子裡的數值,沒有實體大小,由格式決定怎麼存。
- **Device pixel**:實體 (OLED 的 LED、LCD 的液晶、e-ink 的帶電粒子)。

bit depth 差異很大:多數格式 8-bit/channel (256 階,共 ~16M 色);AVIF 支援 8/10/12 bit;JPEG XL 最高 32 bit/channel。

## 從 JS 看 pixel 資料

```js
const imageToSymbols = (src, showPixelsSomehow) => {
  const img = new Image();
  img.onload = () => {
    const width = 128, height = 128;
    const canvas = new OffscreenCanvas(width, height);
    const ctx = canvas.getContext("2d");
    ctx?.drawImage(img, 0, 0);
    const resp = ctx.getImageData(0, 0, width, height);

    let str = "";
    for (let i = 0; i < resp.data.length; i += 4) {
      // 取 alpha channel (RGBA 的第 4 個 byte)
      str += resp.data[i + 3] > 255 / 2 ? "*" : ".";
      if (i % (4 * width) === 0) str += "\n";
    }
    showPixelsSomehow(str);
  };
  img.src = src;
};
```

`ImageData.data` 是連續的 Uint8 typed array,每 4 byte 表示一個 pixel (RGBA)。

## 為什麼壓縮必要

100×133 raw 8-bit RGB 無 alpha → ~40 KB;對應的 JPEG < 3 KB。換成 60 秒 24 fps 影片就是 1,440 frame × 38 KB ≈ 55 MB,顯然必須壓縮。

影片有兩種 frame:
- **Intraframes**:完整影像
- **Interframes**:儲存 motion vectors,描述 pixel 區塊如何在連續 frame 之間移動

**很多現代影像格式其實是影片 codec 的 intraframe**:AVIF、HEIF、WebP 都是。

## Container 與 bytestream

encoder 輸出的 bit sequence 叫 **bitstream**,切成 chunk 後封進 container:

| 格式 | bitstream | container |
|---|---|---|
| AVIF | AV1 | HEIF |
| WebP | VP8 (或 lossless WebP) | RIFF |
| PNG / JPEG / JPEG XL | 自帶 container | — |

AV1/AVIF 的 chunk 叫 **OBU (Open Bitstream Units)**。

## Decoder 流程

讀檔開頭的 **magic number** 辨識格式(AVIF 是 `ftypavif`),依 container 規格找 metadata 與 bitstream。例如某 JPEG:

```
width: 1536
height: 1536
bands: 3
format: uchar
interpretation: srgb
jpeg-multiscan: 1
interlaced: 1
jpeg-chroma-subsample: 4:2:0
```

> **實務建議**:在 `<img>` 上指定 `width` 與 `height` 屬性,別讓瀏覽器自己算 —— 否則會 layout shift。

## Encoder 核心技術

### 1. Lossy vs Lossless

| 格式 | 種類 | 備註 |
|---|---|---|
| PNG | Lossless | 永遠 lossless |
| JPEG | Lossy | 永遠 lossy |
| WebP | Hybrid | VP8 lossy 或 lossless 演算法 |
| AVIF | 兩者皆可 | |
| JPEG XL | 兩者皆可 | |

可以混用:alpha channel 用 lossless,顏色 channel 用 lossy。

### 2. Chroma subsampling

人眼對 luminance 敏感、對 color 較不敏感 → 在 YCbCr 空間「砍掉部分色彩資料」,decoder 從鄰近 pixel 推回。

三個數字描述 4×2 pixel 區域:

- 第一個固定為 4 (區域寬)
- 第二個:第一列存幾個 chroma sample (Cb/Cr)
- 第三個:第二列存幾個 chroma sample

AVIF 支援:`4:4:4` (無 subsampling) / `4:2:2` (砍一半) / `4:2:0` (留 1/4)。

### 3. 利用 spatial locality

純色 100×100 ≈ 500 byte;隨機雜訊 100×100 ≈ 35 KB(差 ~70×)。
原因:真實影像鄰近 pixel 通常相關(同色、漸層、紋理)。

codec 把影像切成 region 各自壓縮:

- JPEG 永遠用 8×8 block。
- AVIF 用 4×4 ~ 128×128 的彈性大小(永遠是 2 的次方,計算高效)。
- AVIF 用 **quadtree (四分樹) 遞迴切割**:從 superblock 開始,計算 region 的色彩平均與偏差;偏差太大就再切。

```js
function buildQuadtree({ x, y, w, h, forceSplitSize, minSize, threshold, data, totalWidth }) {
  const forced = w > forceSplitSize && h > forceSplitSize;
  const std = regionStd(x, y, w, h, data, totalWidth);
  if (w <= minSize || h <= minSize || (!forced && std < threshold)) return;
  ctx.strokeRect(x, y, w, h);
  const midW = (w / 2) | 0;
  const midH = (h / 2) | 0;
  // 遞迴切四份...
}
```

### 4. Prediction modes

不存「整個 block 的 pixel」,而是存「block 與某個 prediction 的差異」(稱為 **residual**)。

常見 prediction 來源:
- 用 block 上方一列、左方一行的鄰近 pixel 做參考
- 鄰近 pixel 的平均
- 沿某方向重複
- 結合多個 pixel
- 進階模式:遞迴從先前預測過的 pixel 再產 patch
- **Chroma-from-luma** (AVIF、JPEG XL):由 Y 推預測 U、V
- 直接「複製其他 block」(對螢幕截圖、重複紋理特別好用)

prediction modes 數量:AVIF 71 種,WebP 10 種,JPEG **不用 prediction**。

### 5. Block transform —— DCT (Discrete Cosine Transform)

把「residual block」轉到頻域:用一組固定的 cosine basis function 來合成 block,各 basis 用不同 coefficient 加權。人眼對高頻不敏感,在頻域比較容易決定「哪些細節可丟」。

> 注意:現代 codec 把 DCT 套在 **residual** 上,不是原始 pixel。

例:8×8 block 中間有 2 pixel 厚的橫線,DCT 後大多數係數會集中在低頻區。

最小可跑的 2D DCT (示意):

```js
const alpha = (u, size) => u === 0 ? 1 / Math.sqrt(size) : Math.sqrt(2 / size);

const dct2D = (block) => {
  const size = block.length;
  const result = Array.from({ length: size }, () => Array(size).fill(0));
  for (let u = 0; u < size; u++) {
    for (let v = 0; v < size; v++) {
      let sum = 0;
      for (let x = 0; x < size; x++) {
        for (let y = 0; y < size; y++) {
          sum +=
            block[x][y] *
            Math.cos(((2 * x + 1) * u * Math.PI) / (2 * size)) *
            Math.cos(((2 * y + 1) * v * Math.PI) / (2 * size));
        }
      }
      result[u][v] = alpha(u, size) * alpha(v, size) * sum;
    }
  }
  return result;
};
```

實際實作的最佳化:
- 拆成兩次 1D DCT (先列、後行)。
- 用 butterfly-based fast DCT。
- 用 SIMD 指令做 vector 化。
- AVIF 還可用 sine transform、GPU 計算 (AV1 支援)。
- AVIF 允許再切成更小的 sub-block 各自 transform。

### 6. Quantization (量化)

把 DCT 浮點係數除以量化參數再取整 —— **目標是製造大量 0**。量化矩陣對 Y 與 chroma channel 不同。例如 JPEG 標準量化矩陣 (Y):

```
[16 11 10 16 24 40 51 61]
[12 12 14 19 26 58 60 55]
[14 13 16 24 40 57 69 56]
[14 17 22 29 51 87 80 62]
[18 22 37 56 68 109 103 77]
[24 35 55 64 81 104 113 92]
[49 64 78 87 103 121 120 101]
[72 92 95 98 112 100 103 99]
```

JPEG 的 "quality" 設定就是這組矩陣的縮放係數;進階 codec 允許每塊用不同量化參數(讓影像不同區域有不同品質)。

### Bonus: Interlacing (progressive rendering)

把不同 frequency 的係數重新排序 —— 粗略細節先存先載。多層 scan 從模糊逐步精細。多數格式 **不會增加檔案大小**,所以基本上免費。

> **PNG 例外**:interlaced PNG 用不同的方法,**反而會增加檔案大小**。能避就避。

### 7. Entropy coding

對量化後的整數係數做無損編碼,核心想法:**常見值用較短 bit**。例:

```
124 124 124 124 10 123 123 1 1 1 2
```

頻次:124 → 4 次,1 → 3 次,123 → 2 次,2 → 1 次,10 → 1 次。對應編碼:

```
124 - 0
1   - 10
123 - 110
2   - 1110
10  - 1111
```

→ 88 bit 縮到 24 bit。這就是 JPEG 用的 **Huffman coding**。

進階 codec 用 **arithmetic coding** 或 **asymmetric numeral systems (ANS)**,接近 Shannon entropy 理論極限,壓縮率比 Huffman 高。

### 8. Post-processing filters (decoder 端)

lossy 壓縮會引入 artifact:
- **Blocking**:平坦區域可見的方塊
- **Blurry edges**
- **Color distortions**
- **Ringing**:邊緣周圍的暈圈
- **Mosquito noise**:邊緣周圍小點閃爍

decoder 端的 filter 用來平滑這些 artifact,參數寫在 bytestream 內。例:

- **Deblocking filters** (Gabor filter,用於 JPEG XL)
- **Wiener filter** (AV1,降噪)
- **CDEF (Constrained Directional Enhancement Filter)**:抗 ringing

若多個 filter 一起用,順序固定 —— 例如 CDEF 在 deblocking 之後。

## 小結

現代 codec 大致流程相同,但在 block size、prediction modes、transform、entropy coding、post-processing pipeline 各有變化。**「更新的 codec 更複雜、有時較慢,但通常壓縮率更好」** —— 但「更好」要在系列第三篇講的 metrics 與 trade-off 下才能定義。

## 個人筆記

- 每個步驟的 lossy/lossless 屬性:subsampling = lossy、prediction 本身 lossless (但 residual 後續會被量化變 lossy)、DCT 因浮點 rounding 有微小 loss、quantization = 最大 lossy 來源、entropy coding lossless。
- 想學 DCT,自己跑一次 8×8 block 的 demo 最快。
- 對「為何 AVIF 在低品質特別好」這個觀察,根本原因是更彈性的 block size + 大量 prediction modes,跟低頻區可以用很大的 block 直接收掉。
