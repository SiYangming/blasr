# blasr 使用示例

本文演示如何用 blasr 把 PacBio 长读（subreads）比对到参考基因组，包括建索引、
m5 输出、比对结果统计以及 BAM/SAM 输出。安装方式见 [`INSTALL.md`](INSTALL.md)。

> ⚠️ **淘汰提示**：blasr 已被 **minimap2 / pbmm2** 取代（pbmm2 README 明示
> *"pbmm2 … is the official replacement for BLASR"*），新项目请优先使用它们。
> 本文命令用于复现既有 blasr 流程。

---

## 前置准备

准备两个输入文件：

- 参考基因组 FASTA：`genome.fasta`
- PacBio reads（FASTA/FASTQ）：`subreads.fasta`

```bash
mkdir -p path/to/blasr
cd path/to/blasr

# 参考基因组与 subreads（路径请按实际替换）
cp /path/to/data/genome.fasta ./
cp /path/to/data/subreads.fasta ./
```

---

## 1. 构建后缀数组索引（sawriter）

先用 `sawriter` 把参考 FASTA 构建为后缀数组 `.sa` 索引，比对时用 `--sa` 指定，
可显著加快启动速度。

```bash
# sawriter <输出.sa> <参考.fasta>
sawriter genome.fasta.sa genome.fasta
```

---

## 2. 执行比对（m5 输出）

使用 blasr 把 subreads 比对到参考基因组，输出 blasr 原生 m5 格式：

```bash
blasr subreads.fasta genome.fasta --sa genome.fasta.sa --header -m 5 --out blasr.out5 --minPctAccuracy 70 --nproc 8 --stride 10
```

关键参数说明：

| 参数 | 含义 |
| --- | --- |
| `subreads.fasta` | 输入 PacBio reads（FASTA/FASTQ） |
| `genome.fasta` | 参考基因组 FASTA |
| `--sa genome.fasta.sa` | 使用 `sawriter` 生成的后缀数组索引 |
| `--header` | 输出结果带 header 行 |
| `-m 5` | 输出 m5 格式（blasr 原生成对比对格式） |
| `--out blasr.out5` | 输出文件名 |
| `--minPctAccuracy 70` | 最低比对准确率阈值（70%） |
| `--nproc 8` | 使用 8 个线程 |
| `--stride 10` | 索引步长 |

---

## 3. 比对结果统计（基于 m5 输出）

### 3.1 统计比对率

统计“比对长度 ≥ 70%”的序列数占总序列数的比例（每条约 70% 以上比对上的 reads 计数 / 全部 reads 数）：

```bash
perl -e '<>; while (<>) { @_ = split /\s+/; next if exists $qname{$_[0]}; $num ++ if $_[11] / $_[1] >= 0.7; $qname{$_[0]} = 1; } $qname = keys %qname; print "$num\t$qname\n"' blasr.out5
```

输出为两列：`有比对结果的序列数` 与 `总序列数`。

### 3.2 统计比对错误率中位数

仅统计读长 ≥ 1000bp 的 reads，输出其比对错误率的中位数：

```bash
perl -e '<>; while (<>) { @_ = split /\s+/; next if exists $qname{$_[0]}; $mm = ($_[12] + $_[13] + $_[14]) / $_[11]; push @mm, $mm if $_[1] >= 1000; } @mm = sort {$a <=> $b} @mm; print "$mm[@mm/2]\n";' blasr.out5
```

---

## 4. 输出 BAM / SAM

除 m5 外，blasr 也可直接输出 BAM 或 SAM（`--out` 后缀需与格式一致）：

```bash
# 输出 BAM（PacBio 推荐格式）
blasr subreads.fasta genome.fasta --sa genome.fasta.sa --bam --out aln.bam --nproc 8 --bestn 10

# 输出 SAM
blasr subreads.fasta genome.fasta --sa genome.fasta.sa --sam --out aln.sam --nproc 4
```

BAM 结果可继续用 samtools 排序、建索引：

```bash
samtools sort -@ 8 -o aln.sorted.bam -O BAM aln.bam
samtools index aln.sorted.bam
```

其他常用参数：

| 参数 | 含义 |
| --- | --- |
| `--bam` | 输出 BAM（需 `--out` 以 `.bam` 结尾） |
| `--sam` | 输出 SAM（默认输出为 m5 原生格式） |
| `--nproc N` | 线程数（默认 4） |
| `--bestn N` | 每条 read 最多报告 hits 数（默认 10） |

---

## 5. 替代方案（推荐）

blasr 已停止维护，推荐迁移到 **minimap2 / pbmm2**。以 minimap2 处理 PacBio
CLR 数据为例：

```bash
# PacBio 长读比对（map-pb 模式）
minimap2 -ax map-pb -t 8 genome.fasta subreads.fasta > pacbio.sam

# SAM 转 BAM 并排序
samtools sort -@ 8 -O BAM -o pacbio.bam pacbio.sam
samtools index pacbio.bam
```

- **minimap2**：通用长读比对器，适用 PacBio CLR / HiFi / Nanopore
- **pbmm2**：PacBio 官方继任者（minimap2 的 SMRT 包装），原生支持 PacBio BAM 输入输出
