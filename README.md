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
  - [1.2 复制旧成功工程](#sec-1-2)
  - [1.3 准备新区域数据、DEM、water mask](#sec-1-3)
  - [1.4 一键替换旧工程参数](#sec-1-4)
  - [1.5 检查替换结果](#sec-1-5)
  - [1.6 检查 alosStack.xml 并重新生成 cmd](#sec-1-6)
- [2. 工程步骤：手动 -> 自动 -> 手动](#sec-2)
  - [2.1 先确认所有 Slurm 都在](#sec-2-1)
  - [2.2 阶段 1：几何准备](#sec-2-2)
  - [2.3 阶段 2：干涉图、多视、几何多视](#sec-2-3)
  - [2.4 阶段 3：电离层到检查图](#sec-2-4)
  - [2.5 阶段 4：应用电离层校正](#sec-2-5)
  - [2.6 阶段 5：滤波、解缠、地理编码、MintPy](#sec-2-6)
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
手动：复制旧成功工程，准备新数据/DEM/water mask
手动：一键替换旧路径、旧日期、旧 pair，并检查
手动：检查 alosStack.xml，重新 create_cmds.py

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

你手动运行的是 `submit_01` 到 `submit_05`，它们内部会自动 `sbatch` 原来的计算脚本。

<a id="sec-1"></a>

## 1. 工程和数据准备

<a id="sec-1-1"></a>

### 1.1 新区域需要改什么

每次换区域，主要改这些：

```text
NEW_PROC       新处理目录
NEW_DATA       新 ALOS-2 数据目录
NEW_DEM        新 DEM 文件
NEW_WBD        新 water mask 文件，或设 None
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

### 1.2 复制旧成功工程

不要直接改旧目录。先复制：

```bash
cd /work/home/panada/insar/proc
cp -a alos2_stack alos2_stack_NEW
cd alos2_stack_NEW
```

确认 Slurm 文件还在：

```bash
find . -maxdepth 2 -name "*.slurm" | sort
```

<a id="sec-1-3"></a>

### 1.3 准备新区域数据、DEM、water mask

建议目录：

```bash
/work/home/panada/insar/data/alos2_NEW
/work/home/panada/insar/data/dem/NEW
/work/home/panada/insar/proc/alos2_stack_NEW
```

你需要准备：

```text
ALOS-2 数据：解压后每个日期目录里有 IMG / LED / VOL
DEM：ISCE2 可读的 .dem.wgs84 + .xml + .vrt
water mask：如果使用，需要 .wbd + .xml + .vrt
```

检查 DEM：

```bash
ls -lh /work/home/panada/insar/data/dem/NEW/NEW.dem.wgs84*
grep -A2 -n "FILE_NAME" /work/home/panada/insar/data/dem/NEW/NEW.dem.wgs84.xml
```

检查 water mask：

```bash
ls -lh /work/home/panada/insar/data/dem/NEW/NEW_watermask.wbd*
grep -A2 -n "FILE_NAME" /work/home/panada/insar/data/dem/NEW/NEW_watermask.wbd.xml
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
NEW_DATA="/work/home/panada/insar/data/alos2_NEW"

OLD_DEM="/work/home/panada/insar/data/dem/dem_aw3d30.dem.wgs84"
NEW_DEM="/work/home/panada/insar/data/dem/NEW/NEW.dem.wgs84"

OLD_WBD="/work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd"
NEW_WBD="/work/home/panada/insar/data/dem/NEW/NEW_watermask.wbd"

OLD_REF="221221"
NEW_REF="CHANGE_REF_YYMMDD"

DATES=(CHANGE_YYMMDD_1 CHANGE_YYMMDD_2 CHANGE_REF_YYMMDD CHANGE_YYMMDD_4)
DATES2=(CHANGE_YYMMDD_1 CHANGE_YYMMDD_2 CHANGE_YYMMDD_4)
PAIRS=(CHANGE_YYMMDD_1-CHANGE_YYMMDD_2 CHANGE_YYMMDD_1-CHANGE_REF_YYMMDD CHANGE_REF_YYMMDD-CHANGE_YYMMDD_4)

PAIR_CONCURRENCY=8
ION_CONCURRENCY=4
DATE2_CONCURRENCY=3
RESAMPLE_CONCURRENCY=2

# ===== DO NOT EDIT BELOW unless needed =====
files=$(find . -maxdepth 2 \( -name "*.slurm" -o -name "alosStack.xml" -o -name "alos2_noto.txt" -o -name "smallbaselineApp.cfg" \) | sort)

date_block="${DATES[*]}"
date2_block="${DATES2[*]}"
pair_block="${PAIRS[*]}"

pair_last=$((${#PAIRS[@]} - 1))
date_last=$((${#DATES[@]} - 1))
date2_last=$((${#DATES2[@]} - 1))

for f in $files; do
  perl -0pi -e "s#\Q$OLD_PROC\E#$NEW_PROC#g" "$f"
  perl -0pi -e "s#\Q$OLD_DATA\E#$NEW_DATA#g" "$f"
  perl -0pi -e "s#\Q$OLD_DEM\E#$NEW_DEM#g" "$f"
  perl -0pi -e "s#\Q$OLD_WBD\E#$NEW_WBD#g" "$f"
  perl -0pi -e "s#\Q$OLD_REF\E#$NEW_REF#g" "$f"
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
旧处理目录、旧 DEM、旧 water mask 还出现：必须改
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
data directory      新 ALOS-2 数据目录
dem for coregistration / geocoding   新 DEM
water body          新 water mask，或 None
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

检查 `cmd_1.sh` 不要有 `-use_wbd_offset`：

```bash
grep -n "estimate_slc_offset\|use_wbd_offset" cmd_1.sh
```

如果还有：

```bash
sed -i 's/ -use_wbd_offset//g' cmd_1.sh
```

<a id="sec-2"></a>

## 2. 工程步骤：手动 -> 自动 -> 手动

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

### 2.2 阶段 1：几何准备

意义：

```text
生成 offset、重采样 SLC、参考日期几何 lat/lon/hgt/los、非参考日期几何 offset。
```

包含 Slurm：

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
find dates -name cull.off | sort | wc -l
find dates_resampled -path "*/insar/*_rg.off" -type f | wc -l
find dates_resampled -path "*/insar/*_az.off" -type f | wc -l
find dates_resampled/CHANGE_REF_YYMMDD/insar -name "*lat*" -o -name "*lon*" -o -name "*hgt*" -o -name "*los*"
```

预期：

```text
cull.off 数量 = 非参考日期数量
*_rg.off 数量 = 非参考日期数量
*_az.off 数量 = 非参考日期数量
参考日期 insar 目录下有 lat/lon/hgt/los/wbd
```

<a id="sec-2-3"></a>

### 2.3 阶段 2：干涉图、多视、几何多视

意义：

```text
形成 pair 干涉图，做 mosaic，估计 radar/DEM affine，修正 range offset，生成差分干涉图和相干性。
```

包含 Slurm：

```text
run_form_array.slurm
run_mosaic_array.slurm
run_radar_dem_offset.slurm
run_rect_range_offset.slurm
run_diff_array.slurm
run_look_array.slurm
run_look_geom.slurm
```

手动创建提交脚本：

```bash
nano submit_02_pairs.sh
```

写入：

```bash
#!/bin/bash
set -euo pipefail

cd /work/home/panada/insar/proc/alos2_stack_NEW

jid_form=$(sbatch --parsable run_form_array.slurm)
jid_mosaic=$(sbatch --parsable --dependency=afterok:$jid_form run_mosaic_array.slurm)
jid_rdrdem=$(sbatch --parsable --dependency=afterok:$jid_mosaic run_radar_dem_offset.slurm)
jid_rect=$(sbatch --parsable --dependency=afterok:$jid_rdrdem run_rect_range_offset.slurm)
jid_diff=$(sbatch --parsable --dependency=afterok:$jid_rect run_diff_array.slurm)
jid_look=$(sbatch --parsable --dependency=afterok:$jid_diff run_look_array.slurm)
jid_geom=$(sbatch --parsable --dependency=afterok:$jid_look run_look_geom.slurm)

echo "form      $jid_form"
echo "mosaic    $jid_mosaic"
echo "rdrdem    $jid_rdrdem"
echo "rect      $jid_rect"
echo "diff      $jid_diff"
echo "look      $jid_look"
echo "lookgeom  $jid_geom"
```

自动运行：

```bash
bash submit_02_pairs.sh
squeue -u panada
```

手动检查：

```bash
find pairs -path "*/insar/*_8rlks_12alks.cor" -type f | wc -l
find pairs -path "*/insar/diff_*_8rlks_12alks.int" -type f | wc -l
cat dates_resampled/CHANGE_REF_YYMMDD/insar/affine_transform.txt
```

预期：

```text
cor 数量 = pair 数量
diff int 数量 = pair 数量
affine_transform.txt 存在，RMS 不离谱
```

<a id="sec-2-4"></a>

### 2.4 阶段 3：电离层到检查图

意义：

```text
准备 pairs_ion，计算每个 pair 的电离层相位，并生成 fig_ion 检查图。
```

包含 Slurm：

```text
run_ion_pairup.slurm
run_ion_unwrap_array.slurm
run_ion_filt_array.slurm
run_ion_prep_array.slurm
run_ion_cal_array.slurm
run_ion_check.slurm
```

手动创建提交脚本：

```bash
nano submit_03_ion_until_check.sh
```

写入：

```bash
#!/bin/bash
set -euo pipefail

cd /work/home/panada/insar/proc/alos2_stack_NEW

jid_pairup=$(sbatch --parsable run_ion_pairup.slurm)
jid_unwrap=$(sbatch --parsable --dependency=afterok:$jid_pairup run_ion_unwrap_array.slurm)
jid_filt=$(sbatch --parsable --dependency=afterok:$jid_unwrap run_ion_filt_array.slurm)
jid_prep=$(sbatch --parsable --dependency=afterok:$jid_filt run_ion_prep_array.slurm)
jid_cal=$(sbatch --parsable --dependency=afterok:$jid_prep run_ion_cal_array.slurm)
jid_check=$(sbatch --parsable --dependency=afterok:$jid_cal run_ion_check.slurm)

echo "ion_pairup $jid_pairup"
echo "ion_unwrap $jid_unwrap"
echo "ion_filt   $jid_filt"
echo "ion_prep   $jid_prep"
echo "ion_cal    $jid_cal"
echo "ion_check  $jid_check"
echo "STOP after ion_check: inspect fig_ion/*.tif manually."
```

自动运行：

```bash
bash submit_03_ion_until_check.sh
squeue -u panada
```

手动检查：

```bash
find fig_ion -type f | sort
```

这里必须人工看图。判断每个 pair 是否有严重长波条纹或明显异常。

如果有坏 pair：

```text
不要直接删结果。
先修改 run_ion_ls.slurm 中用于 least squares 的 pair 列表，排除坏 pair。
```

如果没有坏 pair，继续阶段 4。

<a id="sec-2-5"></a>

### 2.5 阶段 4：应用电离层校正

意义：

```text
用筛选后的 ion pair 反演各日期 ion，并应用回 pairs 下的差分干涉图。
```

包含 Slurm：

```text
run_ion_ls.slurm
run_ion_correct_array.slurm
```

手动创建提交脚本：

```bash
nano submit_04_ion_apply.sh
```

写入：

```bash
#!/bin/bash
set -euo pipefail

cd /work/home/panada/insar/proc/alos2_stack_NEW

jid_ls=$(sbatch --parsable run_ion_ls.slurm)
jid_correct=$(sbatch --parsable --dependency=afterok:$jid_ls run_ion_correct_array.slurm)

echo "ion_ls      $jid_ls"
echo "ion_correct $jid_correct"
```

自动运行：

```bash
bash submit_04_ion_apply.sh
squeue -u panada
```

手动检查：

```bash
find dates_ion -type f | sort | head
find pairs -path "*/insar/diff_*_8rlks_12alks_ori.int" -type f | wc -l
```

预期：

```text
dates_ion 里有每个日期的 ion 文件
每个 pair 的原始差分干涉图被备份为 *_ori.int
```

<a id="sec-2-6"></a>

### 2.6 阶段 5：滤波、解缠、地理编码、MintPy

意义：

```text
滤波、SNAPHU 解缠、地理编码，然后进入 MintPy 时序反演和速度场输出。
```

包含 Slurm：

```text
run_filt_array.slurm
run_unwrap_array.slurm
run_geocode_array.slurm
run_geocode_los.slurm
mintpy/run_mintpy_load.slurm
mintpy/run_mintpy_all.slurm
```

手动创建提交脚本：

```bash
nano submit_05_final_mintpy.sh
```

写入：

```bash
#!/bin/bash
set -euo pipefail

cd /work/home/panada/insar/proc/alos2_stack_NEW

jid_filt=$(sbatch --parsable run_filt_array.slurm)
jid_unwrap=$(sbatch --parsable --dependency=afterok:$jid_filt run_unwrap_array.slurm)
jid_geocode=$(sbatch --parsable --dependency=afterok:$jid_unwrap run_geocode_array.slurm)
jid_los=$(sbatch --parsable --dependency=afterok:$jid_geocode run_geocode_los.slurm)
jid_mint_load=$(sbatch --parsable --dependency=afterok:$jid_los mintpy/run_mintpy_load.slurm)
jid_mint_all=$(sbatch --parsable --dependency=afterok:$jid_mint_load mintpy/run_mintpy_all.slurm)

echo "filt       $jid_filt"
echo "unwrap     $jid_unwrap"
echo "geocode    $jid_geocode"
echo "geocodeLOS $jid_los"
echo "mintLoad   $jid_mint_load"
echo "mintAll    $jid_mint_all"
echo "Note: mintpy_all may show FAILED at final plotting even if main HDF5 products are created."
```

自动运行：

```bash
bash submit_05_final_mintpy.sh
squeue -u panada
```

手动检查：

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
