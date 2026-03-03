---
title: "Honey, I Shrunk the Model (Maybe): blk-archive vs AI Data"
date: 2026-03-03
draft: false
description: "Exploring storage savings of blk-archive across AI-style datasets, compared against tar + gzip."
tags: [Fedora, AI datasets, archival, storage]
---

## Because “It Should Work” Isn’t Data

After reading about the billions spent on AI infrastructure, I kept wondering: how
much of that storage is just... the same bytes over and over? So I decided to find out. As I'm
involved with https://github.com/device-mapper-utils/blk-archive I thought it would be good to understand
how much storage `blk-archive` can realistically save when pointed at AI-style datasets.

Let me be clear upfront: this isn't a speed test or hardware benchmark.
I'm only interested in one question: how many bytes go in, and how
many come out?

I performed all operations at the filesystem level with normal files in directories, **not** at the raw block device layer, though `blk-archive` can work at the block level, especially with thinly provisioned block devices.

`blk-archive` uses content-defined chunking (CDC) with variable-sized chunks. Unlike traditional fixed-size blocks, CDC determines chunk boundaries by analyzing the data content itself, typically by finding patterns in a rolling hash. This means if you insert a single byte at the beginning of a file, CDC will still identify and reuse most of the same chunks, whereas fixed-block approaches would misalign everything. For each chunk found, it is compressed before being written to the archive.

---

### Important Disclaimers

**The Data Is Not Truly Representative.** AI companies keep their training data composition private. I cannot reproduce exact corpus mixtures, internal filtering pipelines, prior deduplication passes, data refresh strategies, or proprietary preprocessing. I used datasets that are structural approximations, they measure storage behavior on **AI-shaped data**, not actual commercial training datasets. The data may not be large or diverse enough to reliably represent specific data types.

**The Tooling Is Early Stage.** `blk-archive` is early-stage software that may change formats, break compatibility, alter chunking behavior, or require re-archiving between versions. There are plans to rename it to `blk-stash` in the future. Nothing about the on-disk format should be considered stable yet. This post establishes a baseline for current behavior.

---

### What Counts as AI-Style Data?

Since proprietary corpora are opaque, I’m approximating structural categories commonly found in AI workflows.

The test data includes:

- Source code
- Completed AI models (small sample)
- Web text
- Wiki text
- Synthetic datasets
- Checkpoint data
- Tokenized data
- Image data

These represent a mix of:

- Large text corpora
- Highly structured markup
- Large binary tensors
- Versioned artifacts
- Already-compressed media
- Deterministically transformed datasets

Each category has very different redundancy characteristics.

---

### Comparison Baseline: tar + gzip

To make this meaningful, I compare `blk-archive` against a simple baseline:

    $ tar -cvzf dataset.tar.gz <dataset>

This produces a conventional compressed archive without cross-file deduplication.

Conceptually:

- `gzip` removes local redundancy and repeated byte patterns.
- `blk-archive` removes duplicate content across files and versions using CDC-based chunking, and then compresses the de-duplicated blocks.

The practical question is: how much additional storage does deduplication actually save beyond compression alone for these AI type files?

---

## Methodology

For each dataset:

### 1. Measure Raw Size

    du -sb <dataset_root>

Record:

- Raw byte size

---

### 2. Create tar + gzip Archive

    tar -czvf dataset.tar.gz <dataset_root>

Record:

- Total compressed archive size

---

### 3. Create blk-archive and pack the files

    blk-archive create -a test_archive && blk-archive pack -a test_archive <file or files of interest>

Record:

- Total archive byte storage on the file system eg. `du -sb <archive dir>`

I did not capture performance measurements. Blk-archive hasn't had many performance-focused enhancements yet.

---

## Metrics

**Deduplication Ratio:**
DedupRatio = RawSize / BlkArchiveSize

**blk-archive Savings (%):**
BlkSavings% = (1 - (BlkArchiveSize / RawSize)) × 100

**gzip Savings (%):**
GzipSavings% = (1 - (TarGzipSize / RawSize)) × 100

**Additional Savings Over gzip:**
AdditionalSavingsBytes = TarGzipSize - BlkArchiveSize

This last metric isolates the value added by deduplication beyond traditional compression.

**Example Calculation (Code Dataset):**
- Raw Size: 10.00 GiB
- tar+gzip: 2.31 GiB
- blk-archive: 1.72 GiB

Applying the formulas:
- Dedup Ratio = 10.00 / 1.72 = 5.8x
- blk Savings% = (1 - 1.72/10.00) × 100 = 82.77%
- gzip Savings% = (1 - 2.31/10.00) × 100 = 76.88%
- Additional Savings Beyond gzip = 82.77% - 76.88% = 5.89%

This means `blk-archive` saves an additional 5.89% of the original data size compared to gzip alone.

---

## Results


| Dataset      | Raw (GiB) | tar+gzip (GiB) | blk-archive (GiB) | gzip Savings (%) | blk Savings (%) | Additional Savings Beyond gzip (%) |
|--------------|----------:|---------------:|------------------:|-----------------:|----------------:|------------------------------------|
| Synthetic    |    154.56 |          49.96 |              9.73 |          67.68%  |         93.71%  |              +26.03%  |
| Scrubbed web |      0.96 |           0.41 |              0.21 |          57.46%  |         77.66%  |              +20.20%  |
| AI models    |      6.86 |           5.43 |              4.42 |          20.90%  |         35.49%  |              +14.59%  |
| code         |     10.00 |           2.31 |              1.72 |          76.88%  |         82.77%  |               +5.89%  |
| web txt      |     27.96 |           9.01 |              7.92 |          67.76%  |         71.69%  |               +3.93%  |
| wiki txt     |     21.96 |           7.73 |              7.72 |          64.81%  |         64.86%  |               +0.05%  |
| checkpoint   |      7.49 |           6.85 |              6.93 |           8.48%  |          7.39%  |               -1.09%  |
| Train data   |     16.83 |          11.27 |             11.80 |          33.03%  |         29.88%  |               -3.15%  |
| Image        |      0.17 |           0.16 |              0.17 |           9.29%  |          4.87%  |               -4.42%  |

**Key Patterns:**
- Deduplication works best on synthetic/controlled data (+26%), scrubbed web text (+20%), and AI models (+14.6%)
- Source code shows moderate but meaningful gains (+5.9%)
- High-entropy data (training binaries, images) actually performs worse with deduplication than gzip alone
- Text corpora vary widely: preprocessed web content dedupes well, raw wiki text does not

---

## Expected Behavior by Dataset Type and what was actually encountered

#### Source Code

For the source code portion of the experiment, I used data from Hugging Face, specifically `bigcode/the-stack`, and capped the sample at roughly 10 GiB. Given how often developers rely on copy-and-paste, along with common programming idioms, repeated license headers, and shared dependencies, I expected both gzip and deduplication to deliver meaningful space savings. That expectation largely held true. What surprised me, however, was just how compressible source code turned out to be. Considering the sheer volume of source code included in modern training datasets, even a 5%+ improvement in storage efficiency from deduplication translates into a substantial reduction at scale.

---

#### Web Text

For this experiment, I pulled a sample of web data from `https://data.commoncrawl.org/crawl-data/` and capped it at roughly 28 GiB. Given how repetitive the web tends to be, consistent HTML markup, boilerplate headers and footers, templated layouts, mirrored sites, versioned pages, and plenty of scraped or syndicated content. I expected this dataset to compress extremely well and show clear gains from deduplication. The results told a different story: improvements beyond standard compression were fairly modest. While the data certainly compressed, deduplication didn’t deliver the dramatic savings I had anticipated.

---

#### Wiki Text

When I think of wiki data, I immediately think of Wikipedia, so for this portion of the experiment I pulled data from `https://dumps.wikimedia.org/enwiki/latest` and capped it at roughly 22 GiB. I wasn’t entirely sure what to expect from this dataset. Text generally compresses well, so it was reasonable to assume gzip would perform strongly. What surprised me, however, was that applying deduplication to the raw input provided virtually no additional benefit.

A possible explanation is that while Wikipedia articles share structure and formatting conventions, the actual article content is highly unique. Unlike web crawls or source repositories, there aren’t large volumes of duplicated pages, mirrored content, or copy-and-paste artifacts across documents. Most redundancy likely exists within individual files rather than between them, which standard compression algorithms already handle effectively. As a result, there simply wasn’t much cross-document duplication left for deduplication to exploit.

---

#### Checkpoint Data

Building models from scratch, rather than simply using them for inference, is new territory for me. When it comes specifically to training checkpoints, I initially assumed that consecutive saves would be fairly similar, especially when taken close together during training. However, in local testing with a small model, that assumption didn’t hold up. I constructed this data with a Python script that pulled the model from Hugging Face `distilbert-base-uncased`. See `generate_checkpoints.py` in the referenced GitHub repo for the specific code.

One detail that stood out: using PyTorch’s default zip-based serialization, saving the same checkpoint twice produces different files on disk. Even when the underlying tensor data is identical, the zip container introduces differences through embedded metadata—timestamps, file ordering, compression details. A simple test confirms this:

```python
model = AutoModelForMaskedLM.from_pretrained(MODEL_NAME)
torch.save(model.state_dict(), "test1.pt")
torch.save(model.state_dict(), "test2.pt")
# SHA256 hashes differ despite identical model weights
```

For deduplication, this container-level variability obscures underlying tensor similarity and reduces CDC chunking effectiveness.

---

#### Completed AI Models

For completed AI models, I grabbed a couple of smaller ones (see `build_ai_dataset.sh`) to keep things manageable. My initial assumption was that models derived from other models would share a significant amount of underlying data. Intuitively, that feels like it should be true.

My experiments didn’t support that assumption. The overlap wasn’t nearly as substantial as I expected. Because model files are large and my testing environment has limits, I wasn’t able to explore this as deeply as I would have liked, and I ultimately left those results out.

The dataset included here contains only two relatively small models, so the sample size is admittedly limited. Larger models, especially closely related ones, would likely make for a much more interesting analysis. That’s an area worth revisiting with more compute and storage headroom.

---

#### Synthetic Dataset

I created the synthetic dataset in `build_ai_dataset.sh`. It is intentionally simple and serves as a calibration control for the experiments.

The dataset includes:

- A file captured at multiple points as it is incrementally appended to. This is roughly what I initially expected training checkpoints to resemble, though real checkpoints behave quite differently.
- Near-duplicate files.
- Exact duplicate files.

The duplication here is deliberate. This dataset is designed to be an easy win. If `blk-archive` does not perform exceptionally well on this data, that would indicate a fundamental issue in the deduplication approach rather than a limitation of the input data.

---

#### Image Data

For image data, I used the CIFAR-100 dataset (`https://www.cs.toronto.edu/~kriz/cifar-100-python.tar.gz`).

My expectation going in was that, since image data is typically already compressed, neither gzip nor deduplication would provide significant additional savings. The test results largely confirmed this assumption.

What was somewhat surprising, however, was how poorly deduplication performed relative to simple gzip. One possible explanation is that, in the absence of meaningful redundancy, the metadata overhead introduced by deduplication outweighs any tangible storage savings.

This is an area worth examining more closely within `blk-archive`.

---

#### Scrubbed Web

For this category, I used the LLaMA training data from the RedPajama dataset on Hugging Face, limited to ≤ 1 GiB.

Datasets of this type are often preprocessed with some form of deduplication. My understanding is that this is typically done using MinHash + LSH, which would, in theory, reduce the potential gains from a tool like `blk-archive`.

However, the results show a fairly strong reduction in size. That leaves a couple of possibilities: either this particular subset did not undergo as much MinHash + LSH deduplication as expected, or `blk-archive` is capturing a different class of redundancy that remains even after that preprocessing step.

---

#### Training Data

I came across this while experimenting with nanoGPT. If you follow the standard workflow, there is a “prepare” phase where you run `prepare.py`. When applied to the OpenWebText dataset, this process produces a `train.bin` file that is roughly 17 GiB in size. That file, along with a few others, serves as the input to the training process.

After training completes, the resulting model is approximately 128 MiB, with functionality roughly comparable to GPT-2 from OpenAI.

This turned out to be another case where `blk-archive` performed poorly relative to simple gzip compression. That result suggests there are certain data layouts, particularly large, tokenized binary blobs like `train.bin`, where traditional compression is simply a better fit than deduplication.

A likely factor here is entropy. Files like `train.bin` behave more like already-compressed data than like structured text. When the byte patterns are highly uniform and lack obvious repetition, there is not much for a deduplication engine to grab onto.

**Potential Improvement:** This suggests adding a lightweight entropy check during processing. If the data appears highly uniform, `blk-archive` could:
- Flag it as a poor candidate for deduplication
- Adjust its chunking strategy accordingly
- Skip compression for chunks before archiving

This would help identify cases like `train.bin` early, rather than discovering after the fact that gzip would have been the better choice. Ideally, `blk-archive` should recognize these patterns and handle them more intelligently.

---

## What I Learned

Across these dataset types, a few patterns became clear:

- Where deduplication meaningfully outperforms gzip, and where it does not.
- Where gzip alone is largely sufficient.
- Where content-defined chunking (CDC) effectively captures cross-file similarity.
- Where data layouts defeat both approaches.

Perhaps more importantly, this exploration provided a better understanding of the kinds of artifacts modern AI workflows produce. Not all AI-related files are good candidates for deduplication. Some datasets, particularly large tokenized binaries or already-compressed assets, behave almost like random data. In those cases, there is little structural redundancy to exploit, and tools like gzip may be the more appropriate choice.

There is certainly more to learn. The sample sizes were limited, and it is entirely possible that some assumptions, tooling choices, or experimental methods could be improved. That is part of the process.

What this work does provide is a baseline. I now have a clearer picture of where `blk-archive` performs well, where it struggles, and where future improvements should be focused. From here, iteration becomes intentional rather than exploratory.

---

## Future Work

This experiment revealed several concrete improvement opportunities for `blk-archive`:

### 1. Intelligent Entropy Detection

High-entropy data (tokenized training binaries, already-compressed images) performs worse with deduplication than gzip alone. Adding a lightweight entropy check during processing could:
- Detect poor deduplication candidates early
- Adjust chunking strategies or skip deduplication for high-entropy chunks
- Automatically fall back to compression-only mode when appropriate
- Warn users when their data won't benefit from deduplication

### 2. Metadata Overhead Optimization

For datasets where deduplication provides minimal benefit (like images), the metadata overhead actually hurts compression ratios. Investigating ways to reduce this overhead would help:
- Minimize metadata per-chunk when storing small or already-compressed files
- Consider hybrid approaches that switch between deduplication and straight compression
- Benchmark different metadata formats to find the optimal balance

### 3. Expanded Dataset Testing

The current experiments used limited sample sizes and may not represent all use cases. Additional testing should include:
- Larger, more diverse model collections (especially model families and fine-tuned variants)
- Multi-version checkpoint sequences from actual training runs
- Different preprocessing pipelines and tokenization schemes
- Real production AI infrastructure workloads at scale

---

## Closing

The results make one thing clear: the value of deduplication depends heavily on the data type.

For synthetic datasets, preprocessed web content, and certain AI model artifacts, `blk-archive` delivered substantial savings beyond gzip, up to 26% additional reduction in some cases. In contrast, for high-entropy data such as tokenized training binaries and already-compressed image datasets, deduplication performed worse than gzip alone. These cases highlight clear areas where the tool can be improved.

The primary goal of `blk-archive` is reducing data at rest. In some environments, especially where data is read once and discarded (“read and burn”), long-term archival efficiency may not be a priority. It’s an open question how much demand there is for optimization at this phase of the data lifecycle, but as AI datasets continue to grow, storage efficiency remains a practical concern.

These findings establish a baseline and help define the next phase of `blk-archive` development. We now have a clearer understanding of where it performs well, where it struggles, and where engineering effort should be focused next.

For more information about the project, see the main repository:
https://github.com/device-mapper-utils/blk-archive


### How to Reproduce the Data

You can find all scripts and programs used to generate the test data for this post in the GitHub repository:

https://github.com/tasleson/dedupe_blog

The experiments use a custom branch of `blk-archive` with features that make the tool easier to use:

```
git clone -b dedupe_blog https://github.com/tasleson/blk-archive
```

I’ve made an effort to ensure everything is accurate and reproducible. That said, it’s possible I missed something, or that certain environmental assumptions (dependencies, system configuration, available storage, etc.) need to be satisfied for the results to match exactly.

If you run into issues, spot mistakes, or see opportunities to improve the methodology, please open an issue or submit a pull request. Feedback and corrections are welcome.

AI is an intensely competitive space, and much of the interesting work happens behind closed doors. That makes it even more valuable when practitioners share experiments, results, and lessons learned. Even small, practical findings can help move the broader community forward.

---

### Appendix: Detailed Results in Bytes

For those interested in the exact measurements, here are the complete results in bytes:

| Dataset      | Raw Size (bytes) | tar+gzip Size (bytes) | blk-archive Size (bytes) | gzip Savings (%) | blk Savings (%) | Extra Savings vs gzip |
|--------------|------------------:|----------------------:|-------------------------:|-----------------:|----------------:|----------------------:|
| Synthetic    |    165,958,158,420 |        53,640,196,898 |           10,444,186,805 |          67.68%  |         93.71%  |              +26.03%  |
| Scrubbed web |      1,029,537,053 |           437,929,200 |              229,990,214 |          57.46%  |         77.66%  |              +20.20%  |
| AI models    |      7,364,692,743 |         5,825,389,202 |            4,750,546,448 |          20.90%  |         35.49%  |              +14.59%  |
| code         |     10,737,450,312 |         2,482,452,936 |            1,850,310,398 |          76.88%  |         82.77%  |               +5.89%  |
| web txt      |     30,024,932,043 |         9,679,676,166 |            8,500,761,920 |          67.76%  |         71.69%  |               +3.93%  |
| wiki txt     |     23,585,076,069 |         8,299,099,651 |            8,287,008,094 |          64.81%  |         64.86%  |               +0.05%  |
| checkpoint   |      8,038,951,640 |         7,357,216,204 |            7,444,980,172 |           8.48%  |          7.39%  |               -1.09%  |
| Train data   |     18,071,164,978 |        12,102,155,168 |           12,670,437,336 |          33.03%  |         29.88%  |               -3.15%  |
| Image        |        186,301,098 |           169,001,437 |              177,223,220 |           9.29%  |          4.87%  |               -4.42%  |

---

#### Note: Written by a human, restructuring, grammar and misc. improvements by AI :-)
