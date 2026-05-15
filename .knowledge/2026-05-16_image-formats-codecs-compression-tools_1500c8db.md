---
source_url: https://developer.mozilla.org/en-US/blog/image-formats-codecs-compression-tools/
source_hash: 1500c8db
retrieved: 2026-05-16
title: 影像格式 (3):編解碼器與壓縮工具 (Image formats: Codecs and compression tools)
---

# 影像格式 (3):編解碼器與壓縮工具

> 原文發佈日期：2025-11-05,作者 Polina Gurtovaia。本系列三部曲的最後一篇,前兩篇分別講「色彩模型」與「像素資料從編碼到解碼」。

## TL;DR

「JPEG 是不是還是最佳影像格式」沒有單一答案 —— 取決於你的影像類型、尺寸、預算、需要的品質、可用工具。這篇用實驗演示如何「正確比較」codec,並提醒**不要只用一張圖、不要只用一種品質參數**就下結論。最後給出一份實用建議清單。

## 選 codec 前該先問自己的問題

- 影像類型?動物毛皮羽毛、地圖、HTML rasterized 元素、向量圖,各自表現差很多。
- 尺寸?大圖 (>1000px) 還是縮圖?
- 終端使用者裝置?是否以行動為主?
- 處理預算?圖片數量多嗎?
- 編碼速度需求?on-the-fly 壓縮通常願意犧牲壓縮率換速度。
- 需要怎樣的品質?專業攝影可能需要 lossless 或高 bit depth。
- 在你需要的品質下「視覺看起來」如何?例如低品質下 AVIF 結果驚人,但它的 filter 會把毛髮糊掉。
- 你被綁定的既有工具支援該格式/參數嗎?

> 編碼結果取決於三件事:**影像本身、codec 本身、傳給 codec 的參數**。例如黑色矩形對任何 codec 都好壓,白雜訊則對所有 codec 都很難。

## 案例 1:壓縮一隻松鼠

原始檔 2733 × 3727、16-bit、約 50 MB。只能在 JPEG / AVIF 二選一。

### 第一輪實驗 (錯誤示範)

對同一張圖,從 quality=1 跑到 100 各做 100 個 JPEG 與 100 個 AVIF。**結果 AVIF 看起來輸 JPEG**(同樣 quality 數值下 AVIF 檔案更大)。

但這結論有問題。檢視 AVIF encoder log:

```sh
AVIF to be written: (Lossy)
 * Resolution     : 2733x3727
 * Bit Depth      : 12 <-- aha!
 * Format         : YUV444 <-- aha!
 ...
```

—— 12-bit 深度 + YUV 4:4:4 (沒 subsampling)。對 JPEG 不公平。改成 8-bit + 4:2:0 後 AVIF 還是輸。

### 第二輪 (引入品質度量)

問題出在「JPEG 的 quality=N」和「AVIF 的 quality=N」**不是同一件事**,我們在比不可比的東西。需要客觀度量:

| 度量 | 行為 | 評價 |
|---|---|---|
| **MSE** (Mean Squared Error) | lower-is-better,0 = 相同 | 對少量極端 pixel 過度敏感,忽略結構/亮度,**不適合** |
| **PSNR** | 與 MSE 算法不同但問題類似 | 廣泛使用但**有同樣缺陷** |
| **SSIM** (Structural Similarity Index) | higher-is-better,1 = 相同 | 模擬人類感知:把影像切成重疊小區塊,各自比較亮度/對比/結構後平均 |

換成 SSIM 比較後:在相同檔案大小下 AVIF 的 SSIM 明顯高於 JPEG,**低品質時 AVIF 優勢更大**。

> 不過數學度量未必符合你的任務。

## 案例 2:有烏鴉 vs 沒烏鴉

對比兩張影像:一張保留烏鴉但有 artifact,一張烏鴉被擦掉只剩背景純色。猜誰的數學度量好?

—— 「沒有烏鴉」的版本反而在某些度量上比較好,因為差異「平均」起來小。**這就是為什麼度量不能無腦相信。**

### 換用 neural network 評估

用以人類標註訓練的 NN 模型來打分數時,**有烏鴉的版本** 0.52 > **沒烏鴉的版本** 0.47,符合直覺。

把同樣的 NN 模型套到松鼠實驗 (這次先縮到 1000 × 1000,接近真實 web 情境 —— HTTP Archive 2025 平均圖大小約 1 MB,等於中等品質 2000 × 2000 JPEG)。AVIF 仍然勝出。

## codec 與「參數」的影響

兩張「品質 75」的 JPEG 看起來幾乎一樣 —— 但檔案大小差 30%。差別是用 MozJPEG vs Chrome 內建 codec。

同 codec、不同參數:

```sh
vips copy raw.png 'subsampling_off.jpg[Q=75, no-subsample]'
vips copy raw.png 'subsampling_on.jpg[Q=75,strip,interlace]'
```

(同樣 Q=75,有無 subsampling/interlace 結果差異可觀。)

## 結論清單

- **多數時候你不需要自己壓縮**。考慮用雲端 on-the-fly 服務 (Cloudflare 等),可在 URL 帶 codec 參數,還順便提供 resize/crop;Cloudflare 把多格式輸出當作一個計費單位,蠻划算。
- **第一步可以無腦試**:`MozJPEG`,參數 `Q=75, strip, interlace`。多數靜態檔可以瘦下來很多。
- **下一步**:輸出多格式,用 `<picture>` 分裝置投遞;試 WebP、AVIF。
- 盡量用 **progressive** 影像。**不要用 interlaced PNG** —— 比非 interlaced 大;能不用 PNG 就不用,WebP / AVIF 的 lossless 更有效率。
- 壓縮時注意 codec 選擇與參數,用 `vips`、`sharp` 等工具,或 Squoosh、Figma 外掛。
- **看不到的圖就不要載**。從最簡單的 `loading="lazy"` 到 Intersection Observer API 都行。
- 蒐集影像使用統計,定期優化。

## 個人筆記

- 比較 codec 時,最容易踩的坑是「相信兩家 codec 的 quality 數值是同一個尺度」。第一步應該是固定 bit depth 與 subsampling 等底層參數。
- 度量(metrics)選擇要看任務:低品質壓縮看 SSIM 或 NN-based;若任務在意 artifact 而非平均誤差,沒度量就直接看人眼。
- 本系列其他兩篇都很值得讀(`color-models-humans-devices`、`image-formats-pixels-graphics`),這篇基本上是它們的應用篇。
