# blasr 安装指南

BLASR（Basic Local Alignment with Successive Refinement，Chaisson & Tesler,
*BMC Bioinformatics* 2012;13:238）是 PacBio 面向单分子长读（SMRT CLR）的比对器，
采用 banded 比对 + 逐级精化（successive refinement）把高错误率长读比对到参考
基因组/参考序列集合。程序族包含两个命令：

- **sawriter**：参考 FASTA → `.sa` 后缀数组索引
- **blasr**：reads + 参考 + `.sa` → 比对结果（原生 m5 / SAM / BAM）

本文提供三种获取方式：

1. **本仓库 Release 预置 conda 环境包**（解压即用）
2. **bioconda 安装**
3. **本仓库源码编译**

> ⚠️ **淘汰提示**：blasr 上游已停止维护，功能上已被 **minimap2 / pbmm2** 取代
> （pbmm2 README 明示 *"pbmm2 … is the official replacement for BLASR"*）。
> 新项目请直接使用 minimap2 / pbmm2，本文仅供复现既有 blasr 流程使用。

---

## 方式一：Release 预置 conda 环境包（解压即用）

本仓库 Release 提供名为 `miniconda3_for_blasr.tar.gz` 的预置 conda 环境包，已内置
**blasr / bax2bam / bam2fastx** 三个命令。下载解压后即可直接作为 conda 环境使用，
无需联网、无需重新解析依赖。

```bash
# 1. 从本仓库 Release 页面下载附件 miniconda3_for_blasr.tar.gz 到当前目录
#    例如使用 curl（请以 Release 附件实际 URL 为准）
curl -L -O https://github.com/SiYangming/blasr/releases/download/conda-env/miniconda3_for_blasr.tar.gz

# 2. 解压到自定义安装前缀
mkdir -p /path/to/install
tar zxf miniconda3_for_blasr.tar.gz -C /path/to/install/

# 3. 将环境 bin 目录加入 PATH
echo 'PATH=$PATH:/path/to/install/miniconda3_for_blasr/bin/' >> ~/.bashrc
source ~/.bashrc

# 4. 验证
blasr --help
```

解压后目录结构为 `/path/to/install/miniconda3_for_blasr/`，其中 `bin/` 下即为
`blasr`、`bax2bam`、`bam2fastx` 等可执行文件。

---

## 方式二：bioconda 安装

使用 Miniconda3 + bioconda 渠道安装 blasr 全家（blasr / bax2bam / bam2fastx）。
建议把 Miniconda3 安装到独立前缀，避免与系统自带软件冲突。

```bash
# 1. 准备 Miniconda3（安装到独立前缀 /path/to/install/miniconda3_for_blasr）
#    安装过程中选择将 PATH 写入与否均可；此处以手动 export 为例
export PATH=/path/to/install/miniconda3_for_blasr/bin:$PATH

# 2. 添加 conda 渠道
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge

# 3. 安装 blasr 及其配套工具
conda install blasr
conda install bax2bam
conda install bam2fastx

# 4. 验证
blasr --help
```

也可以一次性创建独立环境（bioconda 停驻版本为 **blasr=5.3.5**）：

```bash
mamba create -n blasr-native -c conda-forge -c bioconda blasr=5.3.5
conda activate blasr-native
blasr --version 2>&1 | head -1
```

> 说明：blasr 的 license 为 **BSD-3-Clause-Clear**（bioconda 包元数据）。

---

## 方式三：本仓库源码编译

本仓库树内即为 blasr 的 C++ 源码。编译前需要满足 HDF5 依赖：**HDF5 1.8.0 或以上**，
且以 C++ 支持编译（需要存在 `libhdf5_cpp.a`）。若未把 HDF5 安装到系统默认位置，
需要通过两个环境变量 `HDF5INCLUDEDIR`、`HDF5LIBDIR` 指向 HDF5 的头文件与库目录。

```bash
# 1. 获取源码
git clone https://github.com/SiYangming/blasr.git
cd blasr

# 2. 若 HDF5 不在系统默认路径，指定其位置（示例路径请按实际替换）
export HDF5INCLUDEDIR=/path/to/hdf5/include
export HDF5LIBDIR=/path/to/hdf5/lib

# 3. 编译
make

# 4. 编译产物位于：
#    alignment/bin/blasr     （blasr 主程序）
#    alignment/bin/sawriter  （后缀数组索引程序）
```

`make` 默认构建 `alignment` 与 `samutils` 两组可执行程序。也可以安装到指定前缀：

```bash
# 安装到 /path/to/install/bin
make install PREFIX=/path/to/install
```

其他可用目标：

```bash
make build   # 仅编译
make clean   # 清理
```

> 说明：上游官方仓库 PacificBiosciences/blasr 已不可达，本仓库为其源码归档，
> 源码编译路线适合离线或需要自定义构建的场景；一般使用推荐优先走方式一或方式二。

---

## 环境变量配置

无论采用哪种方式，只要希望全局调用 blasr 相关命令，都建议把可执行目录加入 PATH。

```bash
# 方式一：预置 conda 环境包 / 方式二：conda 环境
echo 'PATH=$PATH:/path/to/install/miniconda3_for_blasr/bin/' >> ~/.bashrc
source ~/.bashrc
```

源码编译路线（方式三）额外需要 HDF5 相关变量：

```bash
# 仅源码编译时需要；路径请按实际 HDF5 安装位置替换
export HDF5INCLUDEDIR=/path/to/hdf5/include
export HDF5LIBDIR=/path/to/hdf5/lib
```

---

## 使用示例

安装完成后，如何建索引并跑通一次长读比对，以及如何统计比对结果，详见
[`Example.md`](Example.md)。
