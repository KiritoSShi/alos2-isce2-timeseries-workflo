# ALOS-2 ISCE2 + MintPy 可复用处理流程

本文档是把本次已经跑通的 ALOS-2 时序 InSAR 流程整理成可复用模板。以后换区域时，按本文顺序做，不要直接改旧成功工程。

本次成功案例路径：

```bash
/work/home/panada/insar/proc/alos2_stack
```

## 目录

- [0. 整体步骤介绍](#sec-0)
- [1. 工程和数据准备](#sec-1)
  - [1.1 新区域需要改什么](#sec-1-1)
  - [1.2 新建工程，只复制流程文件](#sec-1-2)
  - [1.3 准备新区域数据、DEM、water mask](#sec-1-3)
  - [1.4 一键替换旧工程参数](#sec-1-4)
  - [1.5 检查替换结果](#sec-1-5)
  - [1.6 检查 alosStack.xml 并重新生成 cmd](#sec-1-6)
- [2. 工程步骤：哪些 Slurm 自动连跑，哪里手动检查](#sec-2)
  - [2.1 先确认所有 Slurm 都在](#sec-2-1)
  - [2.2 阶段 0：读取数据，生成 dates/baseline](#sec-2-2)
  - [2.3 阶段 1：几何准备](#sec-2-3)
  - [2.4 阶段 2：干涉图、多视、几何多视](#sec-2-4)
  - [2.5 阶段 3：电离层到检查图](#sec-2-5)
  - [2.6 阶段 4：应用电离层校正](#sec-2-6)
  - [2.7 阶段 5：滤波、解缠、地理编码、MintPy](#sec-2-7)
- [3. 其它说明](#sec-3)
  - [3.1 water mask 到底怎么用](#sec-3-1)
  - [3.2 Slurm 核数和并发怎么理解](#sec-3-2)
  - [3.3 MintPy / QGIS 结果怎么看](#sec-3-3)
  - [3.4 哪些脚本保留但不作为主线](#sec-3-4)
  - [3.5 归档建议](#sec-3-5)

<a id="sec-0"></a>

## 0. 整体步骤介绍

完整流程是：

```text
手动：新建处理工程，只复制流程文件；data/DEM/water mask 目录可继续沿用
手动：一键替换旧路径、旧日期、旧 pair，并检查
手动：检查 alosStack.xml，重新 create_cmds.py

自动：run_cmd1_pre_offset.slurm
手动：检查 dates/ 和 baseline/ 是否生成

自动：submit_01_geometry.sh
手动：检查几何结果

自动：submit_02_pairs.sh
手动：检查干涉图、相干性、affine_transform

自动：submit_03_ion_until_check.sh
手动：看 fig_ion，判断是否排除坏 pair

自动：submit_04_ion_apply.sh
手动：检查 ion 校正是否应用

自动：submit_05_final_mintpy.sh
手动：检查 unwrap、geocode、MintPy 输出
```

这里的“自动”不是把所有 Slurm 合并成一个大 Slurm，而是用一个 `submit_*.sh` 脚本自动提交多个原来的 `run_*.slurm`：

```bash
sbatch --dependency=afterok:上一步JOBID 下一步.slurm
```

所以你需要保留：

```text
1. 原来的 run_*.slurm / mintpy/run_*.slurm
2. 新增的 submit_01 到 submit_05
```

你手动运行的是 `run_cmd1_pre_offset.slurm` 和 `submit_01` 到 `submit_05`。其中 `run_cmd1_pre_offset.slurm` 负责生成 `dates/`、`baseline/` 等基础目录；`submit_01` 到 `submit_05` 内部会自动 `sbatch` 原来的计算脚本。

<a id="sec-1"></a>

## 1. 工程和数据准备

<a id="sec-1-1"></a>

### 1.1 新区域需要改什么

每次换区域，主要改这些：

```text
NEW_PROC       新处理目录
NEW_DATA       ALOS-2 数据目录；如果你继续用原目录，就保持旧路径不变
NEW_DEM        DEM 文件；如果 DEM 仍覆盖新区域，就保持旧路径不变
NEW_WBD        water mask 文件；如果仍覆盖新区域，就保持旧路径不变
NEW_REF        新参考日期，YYMMDD 格式
DATES          全部日期
DATES2         非参考日期
PAIRS          全部干涉 pair
Slurm array    数组范围，0 到 数量-1
MintPy 配置    metaFile / baselineDir / unwFile / corFile / geomFile / reference.date
```

本次成功案例参数：

```text
DATES = 220316 220413 221221 230315 230412 231206
REFERENCE_DATE = 221221
DATES2 = 220316 220413 230315 230412 231206
PAIRS = 14 个 pair
RLOOKS1/ALOOKS1 = 2/3
RLOOKS2/ALOOKS2 = 4/4
FINAL_LOOKS = 8rlks_12alks
ION_LOOKS = 64rlks_96alks
```

<a id="sec-1-2"></a>

### 1.2 新建工程，只复制流程文件

不要直接改旧目录，也不建议 `cp -a` 复制整个旧 `proc`，因为会把 `dates_resampled/`、`pairs/`、`pairs_ion/`、MintPy `.h5` 等大结果一起复制过去。

这里的 `alos2_stack_NEW` 就是本次要跑的新处理工程目录，不是文档目录，也不是系统自动生成的名字。你可以继续用这个名字，不需要再改名；以后换区域时，也可以把它换成更具体的名字。

推荐只复制脚本、配置和文档：

```bash
cd /work/home/panada/insar/proc
mkdir -p alos2_stack_NEW
cd alos2_stack_NEW

cp ../alos2_stack/*.slurm ./
cp ../alos2_stack/alosStack.xml ./
cp ../alos2_stack/*.md ./ 2>/dev/null

mkdir -p mintpy
cp ../alos2_stack/mintpy/*.slurm mintpy/
cp ../alos2_stack/mintpy/*.txt mintpy/
cp ../alos2_stack/mintpy/*.cfg mintpy/ 2>/dev/null
```

如果你已经建好了 `/work/home/panada/insar/proc/alos2_stack_NEW`，这一节不用重复做，直接进入 1.4。

确认 Slurm 文件还在：

```bash
find . -maxdepth 2 -name "*.slurm" | sort
```

<a id="sec-1-3"></a>

### 1.3 准备或确认数据、DEM、water mask

如果你不想改数据目录，可以继续使用本次目录：

```bash
/work/home/panada/insar/data/alos2
/work/home/panada/insar/data/dem/dem_aw3d30.dem.wgs84
/work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd
/work/home/panada/insar/proc/alos2_stack_NEW
```

这时只需要确认这些数据覆盖新区域：

```text
ALOS-2 数据：新日期数据在 /work/home/panada/insar/data/alos2 里，并且解压/命名正确
DEM：原 DEM 覆盖新区域，并且 .xml/.vrt 存在
water mask：原 water mask 覆盖新区域，并且 .xml/.vrt 存在
```

检查 DEM：

```bash
ls -lh /work/home/panada/insar/data/dem/dem_aw3d30.dem.wgs84*
grep -A2 -n "FILE_NAME" /work/home/panada/insar/data/dem/dem_aw3d30.dem.wgs84.xml
```

检查 water mask：

```bash
ls -lh /work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd*
grep -A2 -n "FILE_NAME" /work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd.xml
```

<a id="sec-1-4"></a>

### 1.4 一键替换旧工程参数

在新工程目录创建：

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW
nano retarget_project.sh
```

写入：

```bash
#!/bin/bash
set -euo pipefail

# ===== CHANGE THESE =====
OLD_PROC="/work/home/panada/insar/proc/alos2_stack"
NEW_PROC="/work/home/panada/insar/proc/alos2_stack_NEW"

OLD_DATA="/work/home/panada/insar/data/alos2"
NEW_DATA="/work/home/panada/insar/data/alos2"

OLD_DEM="/work/home/panada/insar/data/dem/dem_aw3d30.dem.wgs84"
NEW_DEM="/work/home/panada/insar/data/dem/dem_aw3d30.dem.wgs84"

OLD_WBD="/work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd"
NEW_WBD="/work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd"

OLD_REF="221221"
NEW_REF="CHANGE_REF_YYMMDD"

DATES=(CHANGE_YYMMDD_1 CHANGE_YYMMDD_2 CHANGE_REF_YYMMDD CHANGE_YYMMDD_4)

PAIR_CONCURRENCY=8
ION_CONCURRENCY=4
DATE2_CONCURRENCY=3
RESAMPLE_CONCURRENCY=2

# ===== DO NOT EDIT BELOW unless needed =====
files=$(find . -maxdepth 2 \( -name "*.slurm" -o -name "alosStack.xml" -o -name "alos2_noto.txt" -o -name "smallbaselineApp.cfg" \) | sort)

DATES2=()
for d in "${DATES[@]}"; do
  [ "$d" = "$NEW_REF" ] || DATES2+=("$d")
done

PAIRS=()
for ((i=0; i<${#DATES[@]}; i++)); do
  for ((j=i+1; j<${#DATES[@]}; j++)); do
    PAIRS+=("${DATES[$i]}-${DATES[$j]}")
  done
done

date_block="${DATES[*]}"
date2_block="${DATES2[*]}"
pair_block="${PAIRS[*]}"

pair_last=$((${#PAIRS[@]} - 1))
date_last=$((${#DATES[@]} - 1))
date2_last=$((${#DATES2[@]} - 1))

for f in $files; do
  perl -0pi -e "s#\Q$OLD_PROC\E(?![A-Za-z0-9_])#$NEW_PROC#g" "$f"
  perl -0pi -e "s#\Q$OLD_DATA\E#$NEW_DATA#g" "$f"
  perl -0pi -e "s#\Q$OLD_DEM\E#$NEW_DEM#g" "$f"
  perl -0pi -e "s#\Q$OLD_WBD\E#$NEW_WBD#g" "$f"
  perl -0pi -e "s#\Q$OLD_REF\E#$NEW_REF#g" "$f"
  perl -0pi -e "s#\Q${NEW_PROC}_NEW\E#$NEW_PROC#g" "$f"
done

[ -f run_resample_array.slurm ] && perl -0pi -e "s/dates=\([^)]*\)/dates=($date_block)/s" run_resample_array.slurm

for f in run_offset_array.slurm run_geo2rdr_array.slurm run_rect_range_offset.slurm; do
  [ -f "$f" ] && perl -0pi -e "s/dates2=\([^)]*\)/dates2=($date2_block)/s" "$f"
done

for f in run_form_array.slurm run_mosaic_array.slurm run_diff_array.slurm run_look_array.slurm run_filt_array.slurm run_unwrap_array.slurm run_geocode_array.slurm run_ion_correct_array.slurm; do
  [ -f "$f" ] && perl -0pi -e "s/insarpair=\([^)]*\)/insarpair=($pair_block)/s" "$f"
done

for f in run_ion_array.slurm run_ion_unwrap_array.slurm run_ion_filt_array.slurm run_ion_prep_array.slurm run_ion_cal_array.slurm; do
  [ -f "$f" ] && perl -0pi -e "s/ionpair=\([^)]*\)/ionpair=($pair_block)/s" "$f"
done

[ -f run_quicklook_gdal.slurm ] && perl -0pi -e "s/pairs=\([^)]*\)/pairs=($pair_block)/s" run_quicklook_gdal.slurm

for f in run_form_array.slurm run_mosaic_array.slurm run_diff_array.slurm run_look_array.slurm run_filt_array.slurm run_unwrap_array.slurm run_geocode_array.slurm run_ion_correct_array.slurm; do
  [ -f "$f" ] && perl -0pi -e "s/#SBATCH --array=0-\d+%\d+/#SBATCH --array=0-$pair_last%$PAIR_CONCURRENCY/" "$f"
done

for f in run_ion_array.slurm run_ion_unwrap_array.slurm run_ion_filt_array.slurm run_ion_prep_array.slurm run_ion_cal_array.slurm; do
  [ -f "$f" ] && perl -0pi -e "s/#SBATCH --array=0-\d+%\d+/#SBATCH --array=0-$pair_last%$ION_CONCURRENCY/" "$f"
done

for f in run_offset_array.slurm run_geo2rdr_array.slurm run_rect_range_offset.slurm; do
  [ -f "$f" ] && perl -0pi -e "s/#SBATCH --array=0-\d+%\d+/#SBATCH --array=0-$date2_last%$DATE2_CONCURRENCY/" "$f"
done

[ -f run_resample_array.slurm ] && perl -0pi -e "s/#SBATCH --array=0-\d+%\d+/#SBATCH --array=0-$date_last%$RESAMPLE_CONCURRENCY/" run_resample_array.slurm

echo "=== old strings check ==="
grep -R "$OLD_PROC\|$OLD_DATA\|$OLD_DEM\|$OLD_WBD\|$OLD_REF" -n -- *.slurm mintpy/*.slurm alosStack.xml mintpy/*.txt 2>/dev/null || true

echo "=== duplicated NEW path check ==="
grep -R "${NEW_PROC}_NEW" -n -- *.slurm mintpy/*.slurm alosStack.xml mintpy/*.txt 2>/dev/null || true

echo "=== generated dates ==="
echo "DATES: ${DATES[*]}"
echo "DATES2: ${DATES2[*]}"
echo "PAIRS (${#PAIRS[@]}): ${PAIRS[*]}"

echo "=== bash syntax check ==="
bash -n *.slurm
bash -n mintpy/*.slurm
```

运行：

```bash
chmod +x retarget_project.sh
./retarget_project.sh
```

<a id="sec-1-5"></a>

### 1.5 检查替换结果

检查旧值是否还残留：

```bash
grep -R "/work/home/panada/insar/proc/alos2_stack\|221221\|220316\|220413\|230315\|230412\|231206\|dem_aw3d30\|asf_watermask_clip" -n -- *.slurm mintpy/*.slurm alosStack.xml mintpy/*.txt 2>/dev/null
```

看法：

```text
旧处理目录还出现：必须改
旧 DEM、旧 water mask 还出现：如果你本来就继续使用原 DEM/WBD，可以保留
旧日期还出现：如果新区域不是同一天，必须改
新区域刚好同日期：可以保留
```

检查 Bash 语法：

```bash
bash -n *.slurm
bash -n mintpy/*.slurm
```

<a id="sec-1-6"></a>

### 1.6 检查 alosStack.xml 并重新生成 cmd

检查：

```bash
grep -n "data directory\|dem for coregistration\|dem for geocoding\|water body\|reference date\|use water body" alosStack.xml
```

确认：

```text
data directory      ALOS-2 数据目录；如果不改数据目录，就仍是 /work/home/panada/insar/data/alos2
dem for coregistration / geocoding   DEM；如果不改 DEM，就仍是 dem_aw3d30.dem.wgs84
water body          water mask；如果不改，就仍是 asf_watermask_clip.wbd
reference date      新参考日期
use water body to dertermine number of matching offsets   None
```

重新生成命令：

```bash
source /work/home/panada/env/scons_env/bin/activate
source /work/home/panada/isce2_env.sh
export ISCE_STACK=/work/home/panada/isce/isce2-main/contrib/stack

cd /work/home/panada/insar/proc/alos2_stack_NEW
${ISCE_STACK}/alosStack/create_cmds.py -stack_par alosStack.xml
```

注意：`create_cmds.py` 只生成 `cmd_1.sh`、`cmd_2.sh`、`cmd_3.sh` 等处理命令；它不会生成 `dates/` 和 `baseline/`。下一步必须先跑阶段 0，把原始 ALOS-2 数据读入 ISCE 工程。

检查 `cmd_1.sh` 不要有 `-use_wbd_offset`：

```bash
grep -n "estimate_slc_offset\|use_wbd_offset" cmd_1.sh
```

如果还有：

```bash
sed -i 's/ -use_wbd_offset//g' cmd_1.sh
```

<a id="sec-2"></a>

## 2. 工程步骤：哪些 Slurm 自动连跑，哪里手动检查

这一章只讲工程怎么跑。判断规则是：

```text
自动连跑的 Slurm：可以放进 submit_*.sh，用 afterok 自动接下一步。
必须手动检查：不是一个 Slurm，而是某一组 Slurm 跑完后要停下来检查结果。
```

总表：

| 阶段 | 自动连跑的 Slurm | 跑完后必须手动检查什么 | 检查通过后 |
|---|---|---|---|
| 阶段 0 读取数据 | `run_cmd1_pre_offset.slurm` | `dates/`、`baseline/` 是否生成；参考日期是否在 `dates/` 里 | 跑阶段 1 |
| 阶段 1 几何准备 | `run_offset_array.slurm` -> `run_resample_array.slurm` -> `run_cmd1_tail.slurm` -> `run_geo2rdr_array.slurm` | `cull.off`、`dates_resampled`、`lat/lon/hgt/los`、`rg/az.off` 是否齐全 | 跑阶段 2 |
| 阶段 2 干涉图/多视 | `run_form_array.slurm` -> `run_mosaic_array.slurm` -> `run_radar_dem_offset.slurm` -> `run_rect_range_offset.slurm` -> `run_diff_array.slurm` -> `run_look_array.slurm` -> `run_look_geom.slurm` | `.cor`、`diff_*.int` 数量是否等于 pair 数；`affine_transform.txt` 是否正常；相干性是否明显异常 | 跑阶段 3 |
| 阶段 3 电离层检查图 | `run_ion_pairup.slurm` -> `run_ion_unwrap_array.slurm` -> `run_ion_filt_array.slurm` -> `run_ion_prep_array.slurm` -> `run_ion_cal_array.slurm` -> `run_ion_check.slurm` | 必须人工看 `fig_ion/*.tif`，判断是否有坏 pair | 没坏 pair 或修改 pair 列表后，跑阶段 4 |
| 阶段 4 应用 ion 校正 | `run_ion_ls.slurm` -> `run_ion_correct_array.slurm` | `dates_ion` 是否生成；`diff_*_ori.int` 是否备份；ion 校正是否应用回 `pairs` | 跑阶段 5 |
| 阶段 5 最终结果 | `run_filt_array.slurm` -> `run_unwrap_array.slurm` -> `run_geocode_array.slurm` -> `run_geocode_los.slurm` -> `mintpy/run_mintpy_load.slurm` -> `mintpy/run_mintpy_all.slurm` | `filt_*.int`、`.unw`、`.geo`、MintPy `.h5` 是否生成；unwrap/MintPy 结果是否合理 | 归档和可视化 |

换句话说：**先单独跑一次阶段 0；之后手动运行 5 个 `submit_*.sh`，每个 `submit_*.sh` 里面自动提交一串 `run_*.slurm`；每串跑完后你人工检查一次。**

<a id="sec-2-1"></a>

### 2.1 先确认所有 Slurm 都在

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW

ls run_offset_array.slurm run_resample_array.slurm run_cmd1_tail.slurm run_geo2rdr_array.slurm
ls run_form_array.slurm run_mosaic_array.slurm run_radar_dem_offset.slurm run_rect_range_offset.slurm run_diff_array.slurm run_look_array.slurm run_look_geom.slurm
ls run_ion_pairup.slurm run_ion_unwrap_array.slurm run_ion_filt_array.slurm run_ion_prep_array.slurm run_ion_cal_array.slurm run_ion_check.slurm run_ion_ls.slurm run_ion_correct_array.slurm
ls run_filt_array.slurm run_unwrap_array.slurm run_geocode_array.slurm run_geocode_los.slurm
ls mintpy/run_mintpy_load.slurm mintpy/run_mintpy_all.slurm
```

如果有文件不存在，先检查旧工程是否复制完整。

<a id="sec-2-2"></a>

### 2.2 阶段 0：读取数据，生成 dates/baseline

意义：

```text
把 data directory 里的 ALOS-2 原始数据读进 ISCE 工程，生成 dates/、baseline/ 等基础目录。
```

这一步必须在 offset/resample 之前跑。你刚才遇到的：

```text
Exception: cannot get reference date 230127 from the data list
find: dates: No such file or directory
```

就是因为跳过了这一段。不是少建一个空目录，而是少跑了 `cmd_1.sh` 前半段的 `read_data.py` 等命令。

从 `cmd_1.sh` 里提取 offset 之前的命令：

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW

awk '
/^##################################################$/ && seen_read && NR > 1 {exit}
/read data/ {seen_read=1}
{print}
' cmd_1.sh > run_cmd1_pre_offset.sh

chmod +x run_cmd1_pre_offset.sh
bash -n run_cmd1_pre_offset.sh
```

创建 Slurm：

```bash
nano run_cmd1_pre_offset.slurm
```

写入：

```bash
#!/bin/bash
#SBATCH --job-name=alos_pre_offset
#SBATCH --output=%x_%j.log
#SBATCH --error=%x_%j.err
#SBATCH -c 8
#SBATCH --export=NONE
#SBATCH -p wzhctest

set -e

source /etc/profile
module purge
source /work/home/panada/env/scons_env/bin/activate
source /work/home/panada/isce2_env.sh

export ISCE_STACK=/work/home/panada/isce/isce2-main/contrib/stack
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

cd /work/home/panada/insar/proc/alos2_stack_NEW

echo "=== Environment check ==="
which python3
python3 --version
echo "PWD=$PWD"

echo "=== Start pre-offset/read_data step ==="
date

bash -n run_cmd1_pre_offset.sh
bash run_cmd1_pre_offset.sh

echo "=== Done pre-offset/read_data step ==="
date

echo "=== Output check ==="
find dates -maxdepth 1 -type d | sort
find baseline -maxdepth 2 -type f | sort | head -50
```

提交：

```bash
jid_pre=$(sbatch --parsable run_cmd1_pre_offset.slurm)
echo "pre_offset/read_data job: $jid_pre"
squeue -j "$jid_pre"
```

检查：

```bash
sacct -j "$jid_pre" --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS
cat alos_pre_offset_${jid_pre}.err
tail -n 100 alos_pre_offset_${jid_pre}.log

find dates -maxdepth 1 -type d | sort
find baseline -maxdepth 2 -type f | sort | head -50
ls dates/CHANGE_REF_YYMMDD(改成具体的reference日期）
```

预期：

```text
Slurm State = COMPLETED
ExitCode = 0:0
dates/ 目录存在
baseline/ 目录存在
dates/CHANGE_REF_YYMMDD 存在
```

如果 `dates/CHANGE_REF_YYMMDD` 不存在，先不要继续阶段 1，回头检查：

```bash
grep -n "reference date" alosStack.xml
find /work/home/panada/insar/data/alos2 -maxdepth 1 -type d | sort
```

<a id="sec-2-3"></a>

### 2.3 阶段 1：几何准备

意义：

```text
生成 offset、重采样 SLC、参考日期几何 lat/lon/hgt/los、非参考日期几何 offset。
```

自动连跑的 Slurm：

```text
run_offset_array.slurm
run_resample_array.slurm
run_cmd1_tail.slurm
run_geo2rdr_array.slurm
```

手动创建提交脚本：

```bash
nano submit_01_geometry.sh
```

写入：

```bash
#!/bin/bash
set -euo pipefail

cd /work/home/panada/insar/proc/alos2_stack_NEW

jid_offset=$(sbatch --parsable run_offset_array.slurm)
jid_resample=$(sbatch --parsable --dependency=afterok:$jid_offset run_resample_array.slurm)
jid_tail=$(sbatch --parsable --dependency=afterok:$jid_resample run_cmd1_tail.slurm)
jid_geo2rdr=$(sbatch --parsable --dependency=afterok:$jid_tail run_geo2rdr_array.slurm)

echo "offset    $jid_offset"
echo "resample  $jid_resample"
echo "rdr2geo   $jid_tail"
echo "geo2rdr   $jid_geo2rdr"
```

自动运行：

```bash
bash submit_01_geometry.sh
squeue -u panada
```

手动检查：

```bash
find dates -name cull.off | wc -l
find dates_resampled -path "*/s1/*.slc" -type f | wc -l
find dates_resampled -path "*/insar/*_rg.off" -type f | wc -l
find dates_resampled -path "*/insar/*_az.off" -type f | wc -l
find dates_resampled/(改为reference的日期）/insar -name "*lat*" -o -name "*lon*" -o -name "*hgt*" -o -name "*los*"
```

预期：

```text
cull.off 数量 = 非参考日期数量
*_rg.off 数量 = 非参考日期数量
*_az.off 数量 = 非参考日期数量
参考日期 insar 目录下有 lat/lon/hgt/los/wbd
```

<a id="sec-2-4"></a>






### 2.4 阶段 2：干涉图、多视、几何多视

意义：

```text
把 dates/ 和 dates_resampled/ 组织成 pairs/，形成干涉图，做 mosaic，估计 radar/DEM affine，修正 range offset，生成差分干涉图和相干性。
```

进入 2.4 前必须先通过 2.3 硬检查：

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW

find dates_resampled -path "*/s1/*.slc" -type f | wc -l
find dates_resampled -path "*/insar/*_rg.off" -type f | wc -l
find dates_resampled -path "*/insar/*_az.off" -type f | wc -l
find dates_resampled/230127/insar -name "*lat*" -o -name "*lon*" -o -name "*hgt*" -o -name "*los*"
```

只有 `dates_resampled` 存在，并且 `rg.off / az.off / lat / lon / hgt / los` 都存在，才能进入 2.4。

2.4 的正确顺序：

| 顺序 | 步骤 | 脚本 / 命令 | 自动/手动 | 作用 | 检查 |
|---|---|---|---|---|---|
| 2.4.1 | pair up | `pair_up.py` | 自动前置 | 把 `dates/` 和 `dates_resampled/` 按 pair 组织到 `pairs/日期-日期/`，生成 `f1_0740/s1/*.slc` | `find pairs -path "*/f1_0740/*" -type f \| wc -l` 应大于 0 |
| 2.4.2 | form interferogram | `run_form_array.slurm` | 自动，按 pair | 形成每个 pair 的初始干涉图和振幅图 | `find pairs -path "*/f1_0740/*" -type f \| grep -E "\.int$\|\.amp$" \| wc -l` = pair 数 × 2 |
| 2.4.3 | mosaic interferogram | `run_mosaic_array.slurm` | 自动，按 pair | 把 frame/swath 干涉图整理到 pair 的 `insar/` 目录 | `find pairs -path "*/insar/*_2rlks_3alks.int" -type f \| wc -l` = pair 数 |
| 2.4.4 | radar DEM offset | `run_radar_dem_offset.slurm` | 自动，单任务 | 估计雷达坐标和 DEM 几何之间的 affine 关系 | `cat dates_resampled/230127/insar/affine_transform.txt` |
| 2.4.5 | rectify range offset | `run_rect_range_offset.slurm` | 自动，按 dates2，不是按 pair | 用 affine 修正每个非参考日期的 range offset | `find dates_resampled -path "*/insar/*_rg_rect.off" -type f \| wc -l` = dates2 数 |
| 2.4.6 | diff interferogram | `run_diff_array.slurm` | 自动，按 pair | 生成差分干涉图 | `find pairs -path "*/insar/diff_*_8rlks_12alks.int" -type f \| wc -l` = pair 数 |
| 2.4.7 | look/coherence | `run_look_array.slurm` | 自动，按 pair | 多视并生成相干性 | `find pairs -path "*/insar/*_8rlks_12alks.cor" -type f \| wc -l` = pair 数 |
| 2.4.8 | look geometry | `run_look_geom.slurm` | 自动，单任务 | 把 lat/lon/hgt/los/wbd 多视到 8rlks/12alks | `ls dates_resampled/230127/insar/230127_8rlks_12alks.{lat,lon,hgt,los,wbd}` |

Slurm 限制规则：

```text
wzhctest 当前出现过 QOSMaxSubmitJobPerUserLimit，约等于同一时刻最多 20 个提交任务。
#SBATCH --array=0-20%8 虽然最多同时跑 8 个，但仍然一次提交 21 个 array task，会被拒绝。
凡是 array 任务数大于 19，都要分块提交：0-18，然后 19-最后。
```

手动先跑 `pair_up.py`：

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW

/work/home/panada/isce/isce2-main/contrib/stack/alosStack/pair_up.py \
  -idir1 dates \
  -idir2 dates_resampled \
  -odir pairs \
  -ref_date 230127 \
  -pairs 220114-221118 220114-221230 220114-230127 220114-230310 220114-230421 220114-230505 221118-221230 221118-230127 221118-230310 221118-230421 221118-230505 221230-230127 221230-230310 221230-230421 221230-230505 230127-230310 230127-230421 230127-230505 230310-230421 230310-230505 230421-230505

find pairs/220114-221118 -maxdepth 5 -type f -o -type l | sort | head -120
find pairs -path "*/f1_0740/*" -type f | wc -l
```

确认 `pairs/*/f1_0740/s1/*.slc` 出现后，再创建自动脚本，从 form 跑到 look_geom：

```bash
nano submit_02_pairs_chunked.sh
```

写入：

```bash
#!/bin/bash
set -euo pipefail

cd /work/home/panada/insar/proc/alos2_stack_NEW

PAIR_CHUNK_SIZE=19
PAIR_MAX_RUNNING=8
DATE_MAX_RUNNING=3
QOS_WAIT=90

get_count_from_array () {
  local file=$1
  local name=$2
  grep "^${name}=" "$file" | sed 's/.*=(//; s/).*//' | wc -w
}

wait_job () {
  local jid=$1
  local label=$2

  echo "waiting $label $jid"

  while squeue -j "$jid" -h | grep -q .; do
    sleep 30
  done

  sacct -j "$jid" --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS

  if sacct -j "$jid" --format=State -n | grep -E "FAILED|CANCELLED|TIMEOUT|OUT_OF_MEMORY|NODE_FAIL" >/dev/null; then
    echo "ERROR: $label failed: $jid"
    exit 1
  fi

  echo "wait ${QOS_WAIT}s for QOS submit quota release"
  sleep "$QOS_WAIT"
}

submit_pair_chunks () {
  local script=$1
  local label=$2

  local pair_count
  pair_count=$(get_count_from_array "$script" insarpair)
  local last_index=$((pair_count - 1))

  echo "=== $label ==="
  echo "PAIR_COUNT=$pair_count LAST_INDEX=$last_index"

  local start=0
  while [ "$start" -le "$last_index" ]; do
    local end=$((start + PAIR_CHUNK_SIZE - 1))
    [ "$end" -gt "$last_index" ] && end="$last_index"

    local n=$((end - start + 1))
    local conc="$PAIR_MAX_RUNNING"
    [ "$n" -lt "$conc" ] && conc="$n"

    local jid
    jid=$(sbatch --parsable --array=${start}-${end}%${conc} "$script")
    echo "$label ${start}-${end}%${conc} $jid"
    wait_job "$jid" "$label ${start}-${end}%${conc}"

    start=$((end + 1))
  done
}

submit_dates2_array () {
  local script=$1
  local label=$2

  local date_count
  date_count=$(get_count_from_array "$script" dates2)
  local last_index=$((date_count - 1))

  echo "=== $label ==="
  echo "DATES2_COUNT=$date_count LAST_INDEX=$last_index"

  local conc="$DATE_MAX_RUNNING"
  [ "$date_count" -lt "$conc" ] && conc="$date_count"

  local jid
  jid=$(sbatch --parsable --array=0-${last_index}%${conc} "$script")
  echo "$label 0-${last_index}%${conc} $jid"
  wait_job "$jid" "$label"
}

submit_pair_chunks run_form_array.slurm form
submit_pair_chunks run_mosaic_array.slurm mosaic

echo "=== radar dem offset ==="
jid_rdrdem=$(sbatch --parsable run_radar_dem_offset.slurm)
echo "rdrdem $jid_rdrdem"
wait_job "$jid_rdrdem" rdrdem

submit_dates2_array run_rect_range_offset.slurm rect
submit_pair_chunks run_diff_array.slurm diff
submit_pair_chunks run_look_array.slurm look

echo "=== look geom ==="
jid_geom=$(sbatch --parsable run_look_geom.slurm)
echo "lookgeom $jid_geom"
wait_job "$jid_geom" lookgeom

echo "=== submit_02_pairs_chunked done ==="
date
```

运行：

```bash
chmod +x submit_02_pairs_chunked.sh
bash -n submit_02_pairs_chunked.sh
nohup bash submit_02_pairs_chunked.sh > submit_02_pairs_chunked.log 2>&1 &
tail -f submit_02_pairs_chunked.log
```

2.4 完成后检查：

```bash
find pairs -path "*/insar/*_2rlks_3alks.int" -type f | wc -l
find pairs -path "*/insar/*_2rlks_3alks.amp" -type f | wc -l
find dates_resampled -path "*/insar/*_rg_rect.off" -type f | wc -l
find pairs -path "*/insar/diff_*_8rlks_12alks.int" -type f | wc -l
find pairs -path "*/insar/*_8rlks_12alks.cor" -type f | wc -l
cat dates_resampled/230127/insar/affine_transform.txt
ls -lh dates_resampled/230127/insar/230127_8rlks_12alks.{lat,lon,hgt,los,wbd}
```

预期：

```text
2rlks int = pair 数
2rlks amp = pair 数
rg_rect.off = dates2 数
8rlks diff int = pair 数
8rlks cor = pair 数
affine_transform.txt 存在
230127_8rlks_12alks lat/lon/hgt/los/wbd 都存在
```

<a id="sec-2-5"></a>

### 2.5 阶段 3：电离层到检查图

意义：

```text
准备 pairs_ion，计算每个 pair 的电离层相位，并生成 fig_ion 检查图。
```

正确顺序：

```text
run_ion_pairup.slurm
run_ion_unwrap_array.slurm
run_ion_filt_array.slurm
run_ion_prep_array.slurm
run_ion_cal_array.slurm
run_ion_check.slurm
```

注意：`run_ion_pairup.slurm` 是单任务；后面 `run_ion_*_array.slurm` 如果是 21 个 pair，必须分块提交，不能一次 `0-20%4`。

创建自动分块脚本：

```bash
nano submit_03_ion_until_check_chunked.sh
```

写入：

```bash
#!/bin/bash
set -euo pipefail

cd /work/home/panada/insar/proc/alos2_stack_NEW

PAIR_CHUNK_SIZE=19
ION_MAX_RUNNING=4
QOS_WAIT=90

get_ion_count () {
  grep "^ionpair=" "$1" | sed 's/.*=(//; s/).*//' | wc -w
}

wait_job () {
  local jid=$1
  local label=$2

  echo "waiting $label $jid"

  while squeue -j "$jid" -h | grep -q .; do
    sleep 30
  done

  sacct -j "$jid" --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS

  if sacct -j "$jid" --format=State -n | grep -E "FAILED|CANCELLED|TIMEOUT|OUT_OF_MEMORY|NODE_FAIL" >/dev/null; then
    echo "ERROR: $label failed: $jid"
    exit 1
  fi

  echo "wait ${QOS_WAIT}s for QOS submit quota release"
  sleep "$QOS_WAIT"
}

submit_ion_chunks () {
  local script=$1
  local label=$2

  local pair_count
  pair_count=$(get_ion_count "$script")
  local last_index=$((pair_count - 1))

  echo "=== $label ==="
  echo "ION_PAIR_COUNT=$pair_count LAST_INDEX=$last_index"

  local start=0
  while [ "$start" -le "$last_index" ]; do
    local end=$((start + PAIR_CHUNK_SIZE - 1))
    [ "$end" -gt "$last_index" ] && end="$last_index"

    local n=$((end - start + 1))
    local conc="$ION_MAX_RUNNING"
    [ "$n" -lt "$conc" ] && conc="$n"

    local jid
    jid=$(sbatch --parsable --array=${start}-${end}%${conc} "$script")
    echo "$label ${start}-${end}%${conc} $jid"
    wait_job "$jid" "$label ${start}-${end}%${conc}"

    start=$((end + 1))
  done
}

echo "=== ion pairup ==="
jid_pairup=$(sbatch --parsable run_ion_pairup.slurm)
echo "ion_pairup $jid_pairup"
wait_job "$jid_pairup" ion_pairup

submit_ion_chunks run_ion_unwrap_array.slurm ion_unwrap
submit_ion_chunks run_ion_filt_array.slurm ion_filt
submit_ion_chunks run_ion_prep_array.slurm ion_prep
submit_ion_chunks run_ion_cal_array.slurm ion_cal

echo "=== ion check ==="
jid_check=$(sbatch --parsable run_ion_check.slurm)
echo "ion_check $jid_check"
wait_job "$jid_check" ion_check

echo "=== STOP: inspect fig_ion manually ==="
date
```

运行：

```bash
chmod +x submit_03_ion_until_check_chunked.sh
bash -n submit_03_ion_until_check_chunked.sh
nohup bash submit_03_ion_until_check_chunked.sh > submit_03_ion_until_check_chunked.log 2>&1 &
tail -f submit_03_ion_until_check_chunked.log
```

人工检查：

```bash
find fig_ion -type f | sort
```

这里必须人工看图，判断每个 pair 是否有严重长波条纹或明显异常。

如果有坏 pair：

```text
不要直接删结果。
先修改 run_ion_ls.slurm 中用于 least squares 的 pair 列表，排除坏 pair。
```

如果没有坏 pair，继续阶段 4。

<a id="sec-2-6"></a>

### 2.6 阶段 4：应用电离层校正

意义：

```text
用筛选后的 ion pair 反演各日期 ion，并应用回 pairs 下的差分干涉图。
```

正确顺序：

```text
run_ion_ls.slurm
run_ion_correct_array.slurm
```

`run_ion_ls.slurm` 是单任务；`run_ion_correct_array.slurm` 按 pair 跑，如果 pair 数超过 19，也要分块提交。

创建自动脚本：

```bash
nano submit_04_ion_apply_chunked.sh
```

写入：

```bash
#!/bin/bash
set -euo pipefail

cd /work/home/panada/insar/proc/alos2_stack_NEW

PAIR_CHUNK_SIZE=19
ION_MAX_RUNNING=4
QOS_WAIT=90

get_ion_count () {
  grep "^ionpair=" "$1" | sed 's/.*=(//; s/).*//' | wc -w
}

wait_job () {
  local jid=$1
  local label=$2

  echo "waiting $label $jid"

  while squeue -j "$jid" -h | grep -q .; do
    sleep 30
  done

  sacct -j "$jid" --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS

  if sacct -j "$jid" --format=State -n | grep -E "FAILED|CANCELLED|TIMEOUT|OUT_OF_MEMORY|NODE_FAIL" >/dev/null; then
    echo "ERROR: $label failed: $jid"
    exit 1
  fi

  echo "wait ${QOS_WAIT}s for QOS submit quota release"
  sleep "$QOS_WAIT"
}

submit_correct_chunks () {
  local script=run_ion_correct_array.slurm
  local label=ion_correct
  local pair_count
  pair_count=$(get_ion_count "$script")
  local last_index=$((pair_count - 1))

  echo "=== $label ==="
  echo "ION_PAIR_COUNT=$pair_count LAST_INDEX=$last_index"

  local start=0
  while [ "$start" -le "$last_index" ]; do
    local end=$((start + PAIR_CHUNK_SIZE - 1))
    [ "$end" -gt "$last_index" ] && end="$last_index"

    local n=$((end - start + 1))
    local conc="$ION_MAX_RUNNING"
    [ "$n" -lt "$conc" ] && conc="$n"

    local jid
    jid=$(sbatch --parsable --array=${start}-${end}%${conc} "$script")
    echo "$label ${start}-${end}%${conc} $jid"
    wait_job "$jid" "$label ${start}-${end}%${conc}"

    start=$((end + 1))
  done
}

echo "=== ion least squares ==="
jid_ls=$(sbatch --parsable run_ion_ls.slurm)
echo "ion_ls $jid_ls"
wait_job "$jid_ls" ion_ls

submit_correct_chunks

echo "=== submit_04_ion_apply done ==="
date
```

运行：

```bash
chmod +x submit_04_ion_apply_chunked.sh
bash -n submit_04_ion_apply_chunked.sh
nohup bash submit_04_ion_apply_chunked.sh > submit_04_ion_apply_chunked.log 2>&1 &
tail -f submit_04_ion_apply_chunked.log
```

检查：

```bash
find dates_ion -type f | sort | head
find pairs -path "*/insar/diff_*_8rlks_12alks_ori.int" -type f | wc -l
```

预期：

```text
dates_ion 里有每个日期的 ion 文件
每个 pair 的原始差分干涉图被备份为 *_ori.int
```

<a id="sec-2-7"></a>

### 2.7 阶段 5：滤波、解缠、地理编码、MintPy

意义：

```text
滤波、SNAPHU 解缠、地理编码，然后进入 MintPy 时序反演和速度场输出。
```

正确顺序：

```text
run_filt_array.slurm
run_unwrap_array.slurm
run_geocode_array.slurm
run_geocode_los.slurm
mintpy/run_mintpy_load.slurm
mintpy/run_mintpy_all.slurm
```

`run_filt_array.slurm / run_unwrap_array.slurm / run_geocode_array.slurm` 都按 pair 跑。如果 pair 数超过 19，必须分块提交。`run_geocode_los.slurm` 和 MintPy 两个脚本是单任务。

创建自动脚本：

```bash
nano submit_05_final_mintpy_chunked.sh
```

写入：

```bash
#!/bin/bash
set -euo pipefail

cd /work/home/panada/insar/proc/alos2_stack_NEW

PAIR_CHUNK_SIZE=19
PAIR_MAX_RUNNING=8
QOS_WAIT=90

get_pair_count () {
  grep "^insarpair=" "$1" | sed 's/.*=(//; s/).*//' | wc -w
}

wait_job () {
  local jid=$1
  local label=$2

  echo "waiting $label $jid"

  while squeue -j "$jid" -h | grep -q .; do
    sleep 30
  done

  sacct -j "$jid" --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS

  if sacct -j "$jid" --format=State -n | grep -E "FAILED|CANCELLED|TIMEOUT|OUT_OF_MEMORY|NODE_FAIL" >/dev/null; then
    echo "ERROR: $label failed: $jid"
    exit 1
  fi

  echo "wait ${QOS_WAIT}s for QOS submit quota release"
  sleep "$QOS_WAIT"
}

submit_pair_chunks () {
  local script=$1
  local label=$2

  local pair_count
  pair_count=$(get_pair_count "$script")
  local last_index=$((pair_count - 1))

  echo "=== $label ==="
  echo "PAIR_COUNT=$pair_count LAST_INDEX=$last_index"

  local start=0
  while [ "$start" -le "$last_index" ]; do
    local end=$((start + PAIR_CHUNK_SIZE - 1))
    [ "$end" -gt "$last_index" ] && end="$last_index"

    local n=$((end - start + 1))
    local conc="$PAIR_MAX_RUNNING"
    [ "$n" -lt "$conc" ] && conc="$n"

    local jid
    jid=$(sbatch --parsable --array=${start}-${end}%${conc} "$script")
    echo "$label ${start}-${end}%${conc} $jid"
    wait_job "$jid" "$label ${start}-${end}%${conc}"

    start=$((end + 1))
  done
}

submit_pair_chunks run_filt_array.slurm filt
submit_pair_chunks run_unwrap_array.slurm unwrap
submit_pair_chunks run_geocode_array.slurm geocode

echo "=== geocode los ==="
jid_los=$(sbatch --parsable run_geocode_los.slurm)
echo "geocode_los $jid_los"
wait_job "$jid_los" geocode_los

echo "=== mintpy load ==="
jid_mint_load=$(sbatch --parsable mintpy/run_mintpy_load.slurm)
echo "mintpy_load $jid_mint_load"
wait_job "$jid_mint_load" mintpy_load

echo "=== mintpy all ==="
jid_mint_all=$(sbatch --parsable mintpy/run_mintpy_all.slurm)
echo "mintpy_all $jid_mint_all"
wait_job "$jid_mint_all" mintpy_all

echo "=== submit_05_final_mintpy done ==="
date
```

运行：

```bash
chmod +x submit_05_final_mintpy_chunked.sh
bash -n submit_05_final_mintpy_chunked.sh
nohup bash submit_05_final_mintpy_chunked.sh > submit_05_final_mintpy_chunked.log 2>&1 &
tail -f submit_05_final_mintpy_chunked.log
```

检查：

```bash
find pairs -path "*/insar/filt_*_8rlks_12alks.int" -type f | wc -l
find pairs -path "*/insar/filt_*_8rlks_12alks.unw" -type f | wc -l
find pairs -path "*/insar/filt_*_8rlks_12alks_msk.unw" -type f | wc -l
find pairs -path "*/insar/*_8rlks_12alks*.geo*" -type f | sort | head
ls -lh mintpy/*.h5 mintpy/geo/*.h5 2>/dev/null
```

预期：

```text
filt int 数量 = pair 数量
unw 数量 = pair 数量
msk unw 数量 = pair 数量
MintPy 生成 timeseries.h5 / velocity.h5 / geo/geo_timeseries.h5 / geo/geo_velocity.h5
```

注意：`mintpy/run_mintpy_all.slurm` 可能最后绘图报错导致 Slurm 显示 `FAILED`。如果主要 `.h5` 已经生成，科学结果可能已经成功。
<a id="sec-3"></a>

## 3. 其它说明

<a id="sec-3-1"></a>

### 3.1 water mask 到底怎么用

本次 `alosStack.xml` 里有：

```xml
<property name="water body">/work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd</property>
<property name="use water body to dertermine number of matching offsets">None</property>
```

意思是：

```text
water body 路径仍然提供给 ISCE2 后续步骤使用
use water body to determine offsets = None，只是关闭 offset 阶段的 water mask 匹配点筛选
```

所以你会看到水体被掩盖，因为后面的 `rdr2geo / look_geom / unwrap / MintPy` 仍然可能使用 water mask。

常见输出：

```text
dates_resampled/REF/insar/REF_8rlks_12alks.wbd
pairs/*/insar/filt_*_8rlks_12alks_msk.unw
mintpy/waterMask.h5
```

<a id="sec-3-2"></a>

### 3.2 Slurm 核数和并发怎么理解

建议：

```text
单个 pair/date 任务：通常 4-16 核够用
更快的方式：多个 pair/date 并行，而不是一个任务盲目开 64 核
```

示例：

```bash
#SBATCH -c 8
#SBATCH --array=0-13%4
```

含义：

```text
每个任务 8 核
最多同时跑 4 个 array task
最多占 32 核
```

你的云服务器允许调用很多 CPU 时，优先增加 `%并发数`，但要注意磁盘 I/O 和存储空间。

<a id="sec-3-3"></a>

### 3.3 MintPy / QGIS 结果怎么看

主要结果：

```text
mintpy/timeseries.h5
mintpy/velocity.h5
mintpy/temporalCoherence.h5
mintpy/geo/geo_timeseries.h5
mintpy/geo/geo_velocity.h5
mintpy/geo/geo_temporalCoherence.h5
mintpy/geo/geo_velocity.kmz
```

QGIS 中：

```text
geo_velocity.h5 选择 velocity，看平均形变速率
geo_timeseries.h5 的 band 是不同日期的累计形变
单位通常是 m 或 m/year，显示时可乘 100 转 cm 或 cm/year
```

本次日期 band 对应：

```text
Band 1 = 20220316
Band 2 = 20220413
Band 3 = 20221221
Band 4 = 20230315
Band 5 = 20230412
Band 6 = 20231206
```

<a id="sec-3-4"></a>

### 3.4 哪些脚本保留但不作为主线

```text
run_cmd1.slurm
  直接跑完整 cmd_1.sh，容易某一步失败后连锁失败。保留，不推荐主线使用。

run_ion_array.slurm
  需要传 subband/unwrap/filt/prep/cal 参数，容易忘。主线用拆开的 ion 脚本。

run_ion_subband_one.slurm
run_ion_subband_retry.slurm
run_ion_subband_230412_231206.slurm
  本次补救单个失败 pair 用，换区域不作为主线。

run_quicklook.slurm
  Python/PIL 版本，之前因 PIL 缺失失败。

run_quicklook_gdal.slurm
  可选 quicklook，不影响主流程。

run_smd1.slurm
  你贴出的内容为空，归档即可。
```

<a id="sec-3-5"></a>

### 3.5 归档建议

成功跑完后建议保存：

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW

mkdir -p /work/home/panada/insar/archive/alos2_stack_NEW/slurm
mkdir -p /work/home/panada/insar/archive/alos2_stack_NEW/config
mkdir -p /work/home/panada/insar/archive/alos2_stack_NEW/logs

cp *.slurm submit_*.sh /work/home/panada/insar/archive/alos2_stack_NEW/slurm/ 2>/dev/null
cp mintpy/*.slurm /work/home/panada/insar/archive/alos2_stack_NEW/slurm/ 2>/dev/null
cp alosStack.xml cmd_*.sh /work/home/panada/insar/archive/alos2_stack_NEW/config/ 2>/dev/null
cp mintpy/*.txt mintpy/*.cfg /work/home/panada/insar/archive/alos2_stack_NEW/config/ 2>/dev/null
cp *.log *.err /work/home/panada/insar/archive/alos2_stack_NEW/logs/ 2>/dev/null
cp mintpy/*.log mintpy/*.err /work/home/panada/insar/archive/alos2_stack_NEW/logs/ 2>/dev/null
```
