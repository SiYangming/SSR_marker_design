# SSR_marker_design 使用示例

前置：按 `INSTALL.md` 安装 Perl、Primer3（`primer3_core`）与 ParaFly（并行时）；`PATH` 中可访问 `misa.pl`、`misa_primer3.pl`、`primer3_core`。以下命令中的 `path/to/` 为占位路径，按实际环境替换。

## 0. 准备输入

```bash
mkdir -p path/to/work && cd path/to/work

# misa.pl 只从当前工作目录读取 misa.ini
cp path/to/SSR_marker_design/misa.ini .

# 确保 misa.ini 的 GFF 为 false，以便产出下游引物设计所需的 .misa
perl -p -i -e 's/^GFF:.*/GFF:                        false/' misa.ini

ln -sf path/to/genome.fasta genome.fasta
```

## 1. SSR 检测（misa.pl）

```bash
perl path/to/SSR_marker_design/misa.pl genome.fasta

# 产物（写在输入 FASTA 路径旁）：
#   genome.fasta.misa         SSR 位点表
#   genome.fasta.statistics   SSR 统计报告
ls -l genome.fasta.misa genome.fasta.statistics
```

## 2. 查看 `.misa` 结果

```bash
head -n 5 genome.fasta.misa
# ID        SSR nr.  SSR type  SSR      size  start  end
# seq1      1        p2        (AG)12   24    122    145

# SSR 类型分布（p1–p6 完美 SSR；c、c* 复合 SSR）
tail -n +2 genome.fasta.misa | cut -f3 | sort | uniq -c | sort -rn
```

`.misa` 各列含义：`ID`（序列名）/ `SSR nr.`（该序列内 SSR 序号）/ `SSR type`（p1–p6 = 完美 SSR 且基序长度 1–6；c = 相邻复合；c* = 含间隔碱基的复合）/ `SSR`（基序）/ `size` / `start` / `end`（1-based 闭区间）。

## 3. 引物设计

### 方式一：misa_primer3.pl（推荐）

读取 `.misa` 位点表，逐个 SSR 截取两侧翼序列，经 `ParaFly` 并行调用 `primer3_core` 批量设计引物：

```bash
perl path/to/SSR_marker_design/misa_primer3.pl \
    --CPU 8 \
    --flanking_length 300 \
    --min_product_length 100 \
    --max_product_length 250 \
    --gff3_out misa_primer3.gff3 \
    --p3_setting_file p3_settings_file \
    genome.fasta.misa genome.fasta > misa_primer3.out
```

参数说明：

| 参数 | 默认 | 说明 |
| ---- | ---- | ---- |
| `--flanking_length` | 300 | 截取 SSR 两侧翼该长度序列作为 Primer3 输入模板 |
| `--min_product_length` | 100 | 引物允许的最小产物长度 |
| `--max_product_length` | 250 | 引物允许的最大产物长度 |
| `--CPU` | 1 | 通过 `ParaFly` 并行化的并行数 |
| `--p3_setting_file` | 无 | Primer3 设置文件；不指定时结果会明显变差 |
| `--gff3_out` | 无 | 可选，输出 GFF3 结果 |

> `--CPU` 依赖 `ParaFly`（脚本内部以 `ParaFly -c misa_primer3.commands -CPU N` 调度）。运行会在当前目录生成命令文件 `misa_primer3.commands` 与中间目录 `misa_primer3.tmp/`（每位点一个输入/输出文件，便于排查）。

### 方式二：design_MISA.sh（Galaxy helper 串接）

```bash
sh path/to/SSR_marker_design/design_MISA.sh genome.fasta.misa genome.fasta ssr_primers.tsv
```

该脚本等价展开为：

```bash
perl path/to/SSR_marker_design/p3_in.pl  genome.fasta.misa genome.fasta temp.p3in
cat temp.p3in | primer3_core --io_version=3 > temp.p3out
perl path/to/SSR_marker_design/p3_out.pl temp.p3out genome.fasta.misa ssr_primers.tsv
rm -f temp.*
```

用法：`sh design_MISA.sh <misa_file> <source_fasta_file> <output_file>`。

## 4. 结果解读

`misa_primer3.out` 为制表符分隔的引物表：前 7 列为 `ID / SSR nr. / SSR type / SSR / size / start / end`，之后每个 SSR 最多 5 组引物，每组为 `left PRIMER / Tm / size / Right Primer / Tm / size / Product size`。

```bash
# 统计设计出引物的记录数
awk -F'\t' 'NR>1{n++; if($8!="")p++} END{print p"/"n" 条记录设计出引物"}' misa_primer3.out

# 抽取全部左引物序列（FASTA）
awk -F'\t' 'NR>1 && $8!=""{print ">"$1"_L"$2"\n"$8}' misa_primer3.out
```

`misa_primer3.gff3` 每条记录描述一个 SSR 位点，属性列包含 `ID`、`Type`、`SSR`、`Size` 以及 `Primer_N_left_seq/left_tm/left_size/right_seq/right_tm/right_size/product_size`，可直接在 IGV / JBrowse 中查看标记位置。

## 5. 后续验证（可选）

- **引物特异性**：可用 e-PCR（epcr）做电子 PCR 检查，或用 NCBI Primer-BLAST 逐条复核；本仓库脚本未直接调用 epcr。
- **标记可用性**：多态性需在群体样本中经 PCR 验证。
- **规模化提示**：SSR 数可达数万条、`primer3_core` 每位点调用一次，整机耗时由并行数与位点数决定；全基因组可先按染色体拆分并行，再合并 `.misa` 与 `misa_primer3.out`。
