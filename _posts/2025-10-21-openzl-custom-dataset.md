---
title: "OpenZL on a Custom Data Set"
date: 2025-10-21
tags: [Compression, OpenZL]
toc: false
---

The OpenZL [Quick Start][1] guide uses the sra0 file from the Silesia corpus and shows good results. I wanted to experiment with other file types. I picked a CSV file and I used ChatGPT to generate an example file, `openzl_benchmark_10k16.csv` with the following prompt.

```text
Generate an example csv file containing 10,000 rows and 16 columns.
The columns should contain a mix of numeric and string data.
```

I compressed this file with standard compressors and OpenZL (`zli`):

```bash
gzip -9 openzl_benchmark_10k16.csv
xz -9 openzl_benchmark_10k16.csv
zstd -19 openzl_benchmark_10k16.csv
zli compress --profile csv openzl_benchmark_10k16.csv --output openzl_benchmark_10k16.csv.zl
```

| Application | File Size (bytes) | Ratio to Uncompressed |
|-------------|-------------------|-----------------------|
| Uncompressed | 2,268,739 | 1.00 |
| gzip -9 | 996,444 | 0.44 |
| zstd -19 | 868,411 | 0.38 |
| xz -9 | 835,456 | 0.37 |
| OpenZL --profile csv | 802,280 | 0.35 |

Straight away OpenZL provides the best compression ratio. Notice that we had to tell OpenZL something about the data using `--profile csv`.

Now, let's try training mode to create a custom compressor.

```bash
zli train --profile csv openzl_benchmark_10k16.csv --output obm10k.zlc
```

This takes some time (2m 14s on my M1 MacBook Pro) and eventually produces:

```
1 files: 2268739 -> 649425 (3.49),  82.81 MB/s  547.67 MB/s
Training improved compression ratio by 23.54%
```

We can now use the newly created custom compressor to compress our CSV file.

```bash
zli compress openzl_benchmark_10k16.csv --compressor obm10k.zlc --output openzl_benchmark_10k16.csv.custom.zl
```

| Application | File Size (bytes) | Ratio to Uncompressed |
|-------------|-------------------|-----------------------|
| Uncompressed | 2,268,739 | 1.00 |
| gzip -9 | 996,444 | 0.44 |
| zstd -19 | 868,411 | 0.38 |
| xz -9 | 835,456 | 0.37 |
| OpenZL --profile csv | 802,280 | 0.35 |
| OpenZL (custom) | 649,425 | 0.29 |

This is a significant improvement (23.54%) over OpenZL --profile csv.

The custom-trained compressor demonstrates OpenZL's real advantage: it learns your specific data patterns. While generic compressors apply fixed algorithms, OpenZL adapts — delivering 23.54% better compression than its own CSV profile and 29% better than the best generic option (xz). For high-volume, structured data, that training cost pays off quickly.

[1]: https://openzl.org/getting-started/quick-start/
