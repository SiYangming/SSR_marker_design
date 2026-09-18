# SSR_marker_design 归档说明与安装

> 仓库根目录的 `README.md` 为仓库自带说明（Galaxy wrapper 说明），本文件补充软件简介、版本/平台、仓库结构与安装配置。

## 1. 软件简介

本仓库是 **MISA（MIcroSAtellite identification tool）SSR 位点查找与引物设计工具集**，同时附带 Galaxy wrapper 与辅助脚本。核心链路为「MISA 检测 SSR → Primer3 批量设计侧翼引物」：

| 文件 | 作用 |
| ---- | ---- |
| `misa.pl` / `misa.ini` | MISA v2.1：从 FASTA 中识别完美 SSR（unit size 1–6）与复合 SSR，输出 `.misa` 位点表与 `.statistics` 统计 |
| `misa_primer3.pl` | 读取 `.misa`，逐个 SSR 截取两侧翼序列并调用 `primer3_core` 设计引物，输出 TSV 引物表与可选的 GFF3 |
| `p3_in.pl` / `p3_out.pl` | MISA→Primer3 输入构造与输出整理的 helper（供 `design_MISA.sh` 调用） |
| `p3_misa_parameter.pl` | MISA 参数版脚本 |
| `p3_settings.txt` | `misa_primer3.pl` 使用的 Primer3 设置模板 |
| `design_MISA.sh` / `clean_fasta_header.sh` | Galaxy wrapper 使用的辅助脚本 |
| `*.xml` | Galaxy wrapper 与工具注册片段 |

引用：Thiel et al. 2003（PMID 12589540）；Baldwin et al. 2012（doi:10.1007/s11032-012-9727-6）。

## 2. 版本与平台

| 项 | 说明 |
| ---- | ---- |
| MISA | `misa.pl` v2.1（release date 25/08/2020），纯 Perl 实现，无需编译；`misa.ini` 含可选的 `GFF` 输出开关 |
| Primer3 | `primer3_core`（本文档以 primer3-2.5.0 为例，需自行编译） |
| 并行调度 | `ParaFly`（`misa_primer3.pl --CPU N` 依赖） |
| 平台 | Linux x86_64，依赖 Perl 与 Primer3；Galaxy wrapper 需 Galaxy 平台 |

## 3. 仓库结构

```
SSR_marker_design/
├── README.md                 # 仓库自带说明（Galaxy wrappers and helpers for MISA）
├── misa.pl                   # MISA v2.1 主程序（SSR 检测）
├── misa.ini                  # MISA 参数文件
├── misa_primer3.pl           # MISA → Primer3 批量引物设计
├── p3_in.pl                  # 由 .misa + FASTA 生成 Primer3 输入
├── p3_out.pl                 # 由 Primer3 输出整理为引物表格
├── p3_misa_parameter.pl      # MISA 参数版脚本
├── p3_settings.txt           # Primer3 设置模板
├── design_MISA.sh            # p3_in.pl + primer3_core + p3_out.pl 串接脚本
├── clean_fasta_header.sh     # 去除 FASTA header 描述
├── design_MISA.xml           # Galaxy wrapper：MISA Primer Design
├── misa2gff.xml              # Galaxy wrapper：misa2gff
├── p3_misa_parameter.xml     # Galaxy wrapper：p3_misa_parameter
├── clean_fasta_header.xml    # Galaxy wrapper：clean_fasta_header
└── tool_conf_entry_MISA.xml  # Galaxy tool_conf.xml 注册片段
```

## 4. 依赖

| 依赖 | 必需性 | 用途 |
| ---- | ---- | ---- |
| Perl | 必需 | 运行 `misa.pl`、`misa_primer3.pl`、`p3_in.pl`、`p3_out.pl` |
| Primer3（`primer3_core`） | 必需 | 引物设计引擎，由 `misa_primer3.pl` / `design_MISA.sh` 调用 |
| ParaFly | 可选 | 仅当 `misa_primer3.pl` 使用 `--CPU N`（N>1）并行时 |
| e-PCR（epcr，2.3.12） | 可选 | 引物电子 PCR / 特异性验证；本仓库脚本未直接调用 |
| Galaxy | 可选 | 仅 `*.xml` wrapper 需要 |

## 5. 安装与配置

### 5.1 获取仓库

```bash
git clone https://github.com/SiYangming/SSR_marker_design.git
cd SSR_marker_design
```

`misa.pl` 为纯 Perl 脚本，无需编译；`misa.ini` 已随仓库提供。

### 5.2 安装 Primer3（primer3_core，源码编译）

```bash
mkdir -p /path/to/install
wget https://sourceforge.net/projects/primer3/files/primer3/2.5.0/primer3-2.5.0.tar.gz -P /path/to/install/
tar zxf /path/to/install/primer3-2.5.0.tar.gz -C /path/to/install/
cd /path/to/install/primer3-2.5.0/src/
make all
export PATH=$PATH:/path/to/install/primer3-2.5.0/src/

# 断言：命令可达
primer3_core --help < /dev/null | head -n 2
```

### 5.3 conda 替代路线（Primer3 + ParaFly）

```bash
mamba create -n ssr -c conda-forge -c bioconda perl primer3=2.6.1 parafly
conda activate ssr
primer3_core --help < /dev/null | head -n 2
```

> 注意：bioconda 的 primer3 包可能不带 `primer3_config` 目录，此时需按下文 5.5 处理 `PRIMER_THERMODYNAMIC_PARAMETERS_PATH`。

### 5.4 配置 `misa.ini`

`misa.pl` 只从当前工作目录读取 `misa.ini`（无命令行参数指定）。文件三个字段：

```
definition(unit_size,min_repeats):                   1-10 2-6 3-5 4-5 5-5 6-5
interruptions(max_difference_between_2_SSRs):        100
GFF:                                                 true
```

- `definition`：unit size 1–6 各自的最小重复次数；
- `interruptions`：复合 SSR 中两个 SSR 之间允许的最大碱基数；
- `GFF`：**⚠️ 与 `.misa` 输出互斥**——`GFF: true` 时 `misa.pl` 只输出逐序列的 `<序列 ID>.gff`，不产出 `.misa`；而下游 `misa_primer3.pl` / `p3_in.pl` 需要 `.misa`，因此走引物设计链路时必须设为 `false`：

```bash
perl -p -i -e 's/^GFF:.*/GFF:                        false/' misa.ini
```

### 5.5 生成 Primer3 设置文件（`p3_settings_file`）

`p3_settings.txt` 含注释行，且其中 `PRIMER_THERMODYNAMIC_PARAMETERS_PATH` 为示例路径，需要清理并改写为实际的 `primer3_config` 目录：

```bash
# 去掉注释与空行，并在 P3_FILE_ID 前补空行
perl -p -e 's/\s*#.*//; s/^\s*$//; s/P3_FILE_ID/\nP3_FILE_ID/' p3_settings.txt > p3_settings_file

# 将热力学参数目录指向实际 primer3_config
perl -p -i -e 's#^PRIMER_THERMODYNAMIC_PARAMETERS_PATH.*#PRIMER_THERMODYNAMIC_PARAMETERS_PATH=/path/to/install/primer3-2.5.0/src/primer3_config/#' p3_settings_file
```

> ⚠️ 若 `PRIMER_THERMODYNAMIC_PARAMETERS_PATH` 指向不存在的目录，`primer3_core` 会报 `PRIMER_ERROR=Unable to open file .../dangle.dh`（每个位点都失败）。此时应删除该行，让 `primer3_core` 回退使用编译内置的默认参数（SantaLucia 参数），结果与显式指定一致。

### 5.6 Galaxy（可选）

仓库提供 `design_MISA.xml`、`misa2gff.xml`、`p3_misa_parameter.xml`、`clean_fasta_header.xml` 等 wrapper，以及注册片段 `tool_conf_entry_MISA.xml`。将 XML 与对应脚本放入 Galaxy 工具目录，并按注册片段在 `tool_conf.xml` 中登记即可。
