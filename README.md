# ALOS-2 ISCE2 + MintPy 可复用处理流程

本文档用于把已经跑通的 ALOS-2 时序 InSAR 流程保存成模板。换一个区域时，优先复制本文件和对应 `*.slurm` / `*.xml` / `*.txt` 配置，然后只修改标记为 `CHANGE` 的位置。

当前成功案例路径示例：

```bash
/work/home/panada/insar/proc/alos2_stack
```

## 目录

**先按这个走**

- [0. 换区域时按这个顺序做](#step-0)
  - [0.1 总目录](#step-0-1)
  - [0.2 第一步：复制旧成功工程](#step-0-2)
  - [0.3 第二步：准备新区域数据](#step-0-3)
  - [0.4 第三步：一键替换和检查](#step-0-4)
  - [0.5 第四步：检查 alosStack.xml 并生成 cmd](#step-0-5)
  - [0.6 第五步：分段自动跑](#step-0-6)
  - [0.7 哪些地方必须手动停下来](#step-0-7)

**详细步骤和背景**

- [0A. 新区域必须修改的内容](#step-0a)
- [1. 推荐目录结构](#step-1)
- [2. 环境](#step-2)
- [3. 数据准备](#step-3)
- [4. DEM / water mask](#step-4)
- [5. alosStack.xml 配置](#step-5)
- [6. 生成 ISCE2 命令](#step-6)
- [7. Slurm 总体策略](#step-7)
- [8. 本次实际存在的 Slurm 文件清单](#step-8)
- [9. ISCE2 Slurm 模板](#step-9)
- [10. MintPy 流程](#step-10)
- [11. 最终结果说明](#step-11)
- [12. 归档建议](#step-12)
- [13. 换区域最快流程](#step-13)
- [14. DInSAR 与时序的关系](#step-14)
- [15. 自动化方式：哪些可以自动检查后继续](#step-15)
- [16. 换区域一键替换脚本模板](#step-16)
- [17. water mask 本次怎么来的、以后怎么做](#step-17)

**按真实 Slurm 审计后的准流程**

- [18. 按本次真实 Slurm 审计后的准流程](#step-18)
  - [18.1 真实 Slurm 分类](#step-18-1)
  - [18.2 换区域准确必改清单](#step-18-2)
  - [18.3 一键机械替换脚本](#step-18-3)
  - [18.4 自动提交链：到几何完成](#step-18-4)
  - [18.5 自动提交链：干涉图到多视](#step-18-5)
  - [18.6 自动提交链：ion 到人工筛查点](#step-18-6)
  - [18.7 ion 筛查后继续到应用校正](#step-18-7)
  - [18.8 自动提交链：滤波、解缠、地理编码、MintPy](#step-18-8)
  - [18.9 怎么检查这份文档和脚本是否一致](#step-18-9)
  - [18.10 water mask 本次到底怎么处理](#step-18-10)

<a id="step-0"></a>

## 0. 换区域时按这个顺序做

这一节是主线。以后换区域时，不要先跳到后面的 cmd1/cmd2/cmd3 解释里找东西，先按这里从上往下做。后面的章节是细节、解释和备查。

<a id="step-0-1"></a>

### 0.1 总目录

```text
第 0 节    实际操作顺序：从复制旧工程到 MintPy 输出
第 1-7 节  环境、数据、DEM、alosStack.xml、create_cmds 的背景说明
第 8 节    你本次真实存在的 Slurm 文件清单
第 9-17 节 分步骤模板和解释
第 18 节   按你真实 Slurm 审计后的准流程、自动提交链、一键替换脚本
```

真正操作时按下面 7 步走：

```text
1. 复制成功工程到新区域目录
2. 准备新区域 ALOS2 数据、DEM、water mask
3. 用 retarget_project.sh 一键替换路径/日期/pair/array，并检查旧值
4. 修改并检查 alosStack.xml，重新 create_cmds.py
5. 分段自动提交 ISCE2：几何 -> 干涉图 -> ion -> unwrap/geocode
6. 人工检查必须看的结果：geometry、coherence、ion_check、unwrap
7. 跑 MintPy，检查 timeseries/velocity/geo 结果
```

<a id="step-0-2"></a>

### 0.2 第一步：复制旧成功工程

不要直接改旧的成功目录。先复制：

```bash
cd /work/home/panada/insar/proc
cp -a alos2_stack alos2_stack_NEW
cd alos2_stack_NEW
```

这个新目录里会带着你之前写好的 Slurm：

```bash
find . -maxdepth 2 -name "*.slurm" | sort
```

<a id="step-0-3"></a>

### 0.3 第二步：准备新区域数据

你需要先准备好：

```text
新 ALOS2 数据目录
新 DEM 文件
新 DEM 的 .xml/.vrt
新 water mask 文件，如果继续使用 water mask
```

例如：

```bash
/work/home/panada/insar/data/alos2_NEW
/work/home/panada/insar/data/dem/NEW/NEW.dem.wgs84
/work/home/panada/insar/data/dem/NEW/NEW_watermask.wbd
```

water mask 如果暂时没有，也可以先设 `None`，但如果你想保持和本次一样的水体掩膜效果，就要准备新的 `.wbd/.xml/.vrt`。

<a id="step-0-4"></a>

### 0.4 第三步：一键替换和检查

这一步的目的：不用你一个个打开 Slurm 改，把旧路径、旧日期、旧 pair、旧 array 范围机械替换掉。

在新工程目录里：

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW
nano retarget_project.sh
```

脚本内容在第 18.3 节。你只需要改脚本顶部这些变量：

```text
NEW_PROC
NEW_DATA
NEW_DEM
NEW_WBD
NEW_REF
DATES
DATES2
PAIRS
PAIR_CONCURRENCY
ION_CONCURRENCY
DATE2_CONCURRENCY
RESAMPLE_CONCURRENCY
```

然后运行：

```bash
chmod +x retarget_project.sh
./retarget_project.sh
```

它会做三件事：

```text
1. 替换所有 Slurm / xml / MintPy txt 里的旧路径和旧参考日期
2. 替换 dates / dates2 / pair 数组
3. 自动调整 #SBATCH --array=0-N%M 的范围
```

运行完后，继续手动检查一遍旧值：

```bash
grep -R "/work/home/panada/insar/proc/alos2_stack\|221221\|220316\|220413\|230315\|230412\|231206\|dem_aw3d30\|asf_watermask_clip" -n -- *.slurm mintpy/*.slurm alosStack.xml mintpy/*.txt 2>/dev/null
```

搜出来的内容逐个看：如果是旧处理目录、旧 DEM、旧 water mask，必须改；如果只是新区域刚好也有相同日期，可以保留。

再做语法检查：

```bash
bash -n *.slurm
bash -n mintpy/*.slurm
```

<a id="step-0-5"></a>

### 0.5 第四步：检查 alosStack.xml 并生成 cmd

先检查 `alosStack.xml`：

```bash
grep -n "data directory\|dem for coregistration\|dem for geocoding\|water body\|reference date\|use water body" alosStack.xml
```

你要确认：

```text
data directory      指向新 ALOS2 数据目录
dem for coregistration / geocoding 指向新 DEM
water body          指向新 water mask，或设为 None
reference date      是新参考日期
use water body to dertermine number of matching offsets 保持 None
```

然后重新生成 ISCE2 命令：

```bash
export ISCE_STACK=/work/home/panada/isce/isce2-main/contrib/stack
${ISCE_STACK}/alosStack/create_cmds.py -stack_par alosStack.xml
```

生成后一定检查 `cmd_1.sh` 里不要再出现 `-use_wbd_offset`：

```bash
grep -n "estimate_slc_offset\|use_wbd_offset" cmd_1.sh
```

如果还出现 `-use_wbd_offset`，删掉：

```bash
sed -i 's/ -use_wbd_offset//g' cmd_1.sh
```

<a id="step-0-6"></a>

### 0.6 第五步：分段自动跑

这里的“自动”不是把所有 Slurm 合并成一个大 Slurm，而是用 `sbatch --dependency=afterok:JOBID` 自动接力。原来的 `run_*.slurm` 必须保留在目录里。

这一小节按真正操作顺序写：**手动准备 -> 自动跑一段 -> 手动检查 -> 再自动跑下一段**。

#### 0.6.1 手动：先确认这些文件都在

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW

ls run_offset_array.slurm run_resample_array.slurm run_cmd1_tail.slurm run_geo2rdr_array.slurm
ls run_form_array.slurm run_mosaic_array.slurm run_radar_dem_offset.slurm run_rect_range_offset.slurm run_diff_array.slurm run_look_array.slurm run_look_geom.slurm
ls run_ion_pairup.slurm run_ion_unwrap_array.slurm run_ion_filt_array.slurm run_ion_prep_array.slurm run_ion_cal_array.slurm run_ion_check.slurm run_ion_ls.slurm run_ion_correct_array.slurm
ls run_filt_array.slurm run_unwrap_array.slurm run_geocode_array.slurm run_geocode_los.slurm
ls mintpy/run_mintpy_load.slurm mintpy/run_mintpy_all.slurm
```

如果这里有文件不存在，先不要跑自动链，回去检查复制旧工程是否完整。

#### 0.6.2 手动：创建 5 个提交脚本

这 5 个是“提交脚本”，用 `bash` 运行；它们内部会自动 `sbatch` 你的 `run_*.slurm`。

**submit_01_geometry.sh**

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
echo "Next manual check after geo2rdr finishes."
```

**submit_02_pairs.sh**

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
echo "Next manual check after lookgeom finishes."
```

**submit_03_ion_until_check.sh**

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

**submit_04_ion_apply.sh**

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
echo "Next manual check after ion_correct finishes."
```

**submit_05_final_mintpy.sh**

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

给它们执行权限：

```bash
chmod +x submit_01_geometry.sh submit_02_pairs.sh submit_03_ion_until_check.sh submit_04_ion_apply.sh submit_05_final_mintpy.sh
```

#### 0.6.3 自动 1：跑几何准备

手动提交：

```bash
bash submit_01_geometry.sh
```

查看队列：

```bash
squeue -u panada
```

等全部结束后，手动检查：

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

#### 0.6.4 自动 2：跑干涉图、多视、几何多视

几何检查没问题后，手动提交：

```bash
bash submit_02_pairs.sh
```

等全部结束后，手动检查：

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

#### 0.6.5 自动 3：跑电离层到检查图

干涉图检查没问题后，手动提交：

```bash
bash submit_03_ion_until_check.sh
```

等全部结束后，必须人工看 `fig_ion`：

```bash
find fig_ion -type f | sort
```

这里是人工节点。你要看每个 ion 图有没有严重长波条纹、明显异常 pair。

如果有坏 pair：不要直接删结果，先修改 `run_ion_ls.slurm` 中用于 least squares 的 pair 列表，把坏 pair 排除。

如果 ion 图可以接受，继续下一步。

#### 0.6.6 自动 4：应用电离层校正

手动提交：

```bash
bash submit_04_ion_apply.sh
```

等结束后，手动检查：

```bash
find dates_ion -type f | sort | head
find pairs -path "*/insar/diff_*_8rlks_12alks_ori.int" -type f | wc -l
```

预期：

```text
dates_ion 里有每个日期的 ion 文件
每个 pair 的原始差分干涉图被备份为 *_ori.int
```

#### 0.6.7 自动 5：滤波、解缠、地理编码、MintPy

手动提交：

```bash
bash submit_05_final_mintpy.sh
```

等结束后，手动检查：

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

注意：`mintpy/run_mintpy_all.slurm` 可能最后绘图报错导致 Slurm 显示 `FAILED`，但如果主要 `.h5` 都已经生成，科学结果可能已经成功。以 `ls -lh mintpy/*.h5 mintpy/geo/*.h5` 为准。

<a id="step-0-7"></a>

### 0.7 哪些地方必须手动停下来

能自动的：文件是否生成、任务是否成功、pair 数量是否够，这些可以靠脚本检查。

必须手动的：

```text
1. 新 ALOS2 数据日期和命名是否正确
2. DEM / water mask 是否覆盖研究区
3. submit_01 后：lat/lon/hgt/los/offset 是否齐全
4. submit_02 后：coherence 和干涉图是否明显异常
5. submit_03 后：fig_ion/*.tif 是否有严重长波条纹坏 pair
6. unwrap 后：解缠有没有大片跳变、断裂、异常孤岛
7. MintPy 后：temporalCoherence、velocity、timeseries 是否合理
```

所以这不是“一键从原始数据跑到最后完全不看”。更稳的自动方式是：**自动跑一段，人工看关键质量，再继续下一段。**

<a id="step-0a"></a>

## 0A. 新区域必须修改的内容

每次换区域，至少修改这些：

```text
CHANGE: 项目名，例如 noto、area2、eq2024 等
CHANGE: 原始 ALOS-2 数据目录
CHANGE: 处理目录
CHANGE: DEM 范围和 DEM 文件名
CHANGE: alosStack.xml 里的 data directory / DEM / reference date / pair list
CHANGE: 参考日期 reference date
CHANGE: 干涉 pair 列表
CHANGE: MintPy 配置文件名和路径
CHANGE: 如果图像范围变化很大，Slurm 时间和内存
```

本次成功案例的关键参数：

```text
DATES = 220316 220413 221221 230315 230412 231206
REFERENCE_DATE = 221221
FRAME = 0720
SWATH = 1
RLOOKS1/ALOOKS1 = 2/3
RLOOKS2/ALOOKS2 = 4/4
FINAL_LOOKS = 8rlks_12alks
```

<a id="step-1"></a>

## 1. 推荐目录结构

新区域建议用独立目录，避免污染成功案例。

```bash
mkdir -p /work/home/panada/insar/data/alos2_NEW
mkdir -p /work/home/panada/insar/data/dem/NEW
mkdir -p /work/home/panada/insar/proc/alos2_stack_NEW
```

推荐结构：

```text
/work/home/panada/insar/data/alos2_NEW/        原始或解压后的 ALOS-2 数据
/work/home/panada/insar/data/dem/NEW/          DEM / water mask
/work/home/panada/insar/proc/alos2_stack_NEW/  ISCE2 + MintPy 处理目录
```

<a id="step-2"></a>

## 2. 环境

### ISCE2 环境

用于 alosStack / ISCE2：

```bash
source /work/home/panada/env/scons_env/bin/activate
source /work/home/panada/isce2_env.sh
export ISCE_STACK=/work/home/panada/isce/isce2-main/contrib/stack
```

如果 Slurm 脚本里需要完整环境：

```bash
source /etc/profile
module purge
source /work/home/panada/env/scons_env/bin/activate
source /work/home/panada/isce2_env.sh
export ISCE_STACK=/work/home/panada/isce/isce2-main/contrib/stack
```

### MintPy 环境

用于时序反演：

```bash
source /work/home/panada/MintPY/activate_mintpy.sh
```

MintPy 读取 ISCE/alosStack 元数据时，还需要 ISCE2 Python 路径和动态库：

```bash
export PYTHONPATH=/work/home/panada/isce_pkg:/work/home/panada/isce_pkg/isce/components:$PYTHONPATH
export LD_LIBRARY_PATH=/public/software/mathlib/hdf5/1.12.2/gnu/lib:/work/home/panada/local/lib64:/work/home/panada/local/lib:/work/home/panada/local/gcc-10/lib64:/opt/rh/devtoolset-7/root/usr/lib64:/opt/rh/devtoolset-7/root/usr/lib/gcc/x86_64-redhat-linux/7:/work/home/panada/local/python3.6/lib:$LD_LIBRARY_PATH
```

检查：

```bash
python -c "import mintpy, numpy; print('mintpy', mintpy.__version__); print('numpy', numpy.__version__)"
python -c "import isce, isceobj; from iscesys.ImageApi import DataAccessor; print('isce ok')"
```

注意：本次加权反演需要把 NumPy 从 `2.x` 降到 `1.26.4` 后才成功。

<a id="step-3"></a>

## 3. 数据准备

### 3.1 解压和命名

ALOS-2 每期解压后目录内通常有：

```text
IMG-HH-ALOS2...
LED-ALOS2...
VOL-ALOS2...
```

alosStack 可以读取按日期整理的目录，建议命名为六位日期：

```text
220316
220413
221221
230315
230412
231206
```

如果原始 zip 类似：

```text
127-720-RU2_9-20220316.zip
```

可以解压后改成：

```text
220316
```

这是人工检查点：确认每个日期目录里都有 `IMG/LED/VOL` 文件。

<a id="step-4"></a>

## 4. DEM / water mask

### 4.1 DEM

本次使用过 AW3D30 或 SRTMGL1_E 下载。换区域时必须重新设置范围：

```text
CHANGE: north/south/east/west
CHANGE: demtype
CHANGE: output file name
```

下载后 ISCE2 最终需要 `.dem.wgs84` 及其 `.xml/.vrt`，并且 `.xml` 里的 `FILE_NAME` 必须是绝对路径。

检查：

```bash
ls -lh /work/home/panada/insar/data/dem/NEW/*.dem.wgs84*
grep -A2 -n "FILE_NAME" /work/home/panada/insar/data/dem/NEW/*.dem.wgs84.xml
```

### 4.2 water mask

本次最终使用：

```text
/work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd
```

注意：water body 可以用于 geocoding/water mask，但如果 `estimate_slc_offset.py` 加了 `-use_wbd_offset` 且水体文件格式不合适，可能导致 offset 失败。

本次解决方式：

```text
alosStack.xml:
<property name="use water body to dertermine number of matching offsets">None</property>

cmd_1.sh:
删除 estimate_slc_offset.py 命令里的 -use_wbd_offset
```

这是重要人工检查点：生成 `cmd_1.sh` 后必须检查：

```bash
grep -n "estimate_slc_offset\|use_wbd_offset" cmd_1.sh
```

如果还有 `-use_wbd_offset`，按需删除。

<a id="step-5"></a>

## 5. alosStack.xml 配置

复制模板：

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW
cp /work/home/panada/isce/isce2-main/contrib/stack/alosStack/alosStack.xml ./
```

必须改：

```xml
<property name="data directory">CHANGE_RAW_ALOS2_DIR</property>

<property name="dem for coregistration">CHANGE_DEM_ABSOLUTE_PATH</property>
<property name="dem for geocoding">CHANGE_DEM_ABSOLUTE_PATH</property>
<property name="water body">CHANGE_WBD_ABSOLUTE_PATH_OR_None</property>

<property name="reference date of the stack">CHANGE_REFERENCE_DATE</property>
```

如果先整幅影像跑，可以不设置 geocode bounding box。后续如需裁剪再改。

关键检查：

```bash
grep -n "data directory\|dem for coregistration\|dem for geocoding\|water body\|reference date" alosStack.xml
grep -n "use water body to dertermine" alosStack.xml
```

<a id="step-6"></a>

## 6. 生成 ISCE2 命令

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW
export ISCE_STACK=/work/home/panada/isce/isce2-main/contrib/stack
${ISCE_STACK}/alosStack/create_cmds.py -stack_par alosStack.xml
```

成功后会生成：

```text
cmd_1.sh
cmd_2.sh
cmd_3.sh
cmd_4.sh
```

分别大致是：

```text
cmd_1.sh  读取数据、baseline、offset、resample、rdr2geo、geo2rdr
cmd_2.sh  干涉图、mosaic、diff interferogram、look/coherence、几何多视
cmd_3.sh  电离层校正
cmd_4.sh  滤波、解缠、地理编码
```

生成后不要直接全跑，先检查：

```bash
grep -n "use_wbd_offset\|estimate_slc_offset" cmd_1.sh
grep -n "^#\|form_interferogram\|mosaic_interferogram\|diff_interferogram\|look_coherence\|look_geom" cmd_2.sh | head -80
grep -n "^#\|ion\|subband\|unwrap\|filt\|cal\|correct" cmd_3.sh | head -120
grep -n "^#\|filt\|unwrap\|geocode" cmd_4.sh | head -120
```

<a id="step-7"></a>

## 7. Slurm 总体策略

不要一个任务盲目申请 64 核。很多 ISCE2/MintPy 步骤不是 64 核线性加速。

推荐：

```text
单个 pair/date 任务：4-8 核
MintPy 时序反演：16 核 / 32G
多个 pair 用 job array 并行
多个区域可以同时跑多个 16 核任务
```

Slurm 分区：

```text
-p wzhctest
```

如果内存/CPU 比例报错，增加 `-c` 而不是盲目降低内存。

<a id="step-8"></a>

## 8. 本次实际存在的 Slurm 文件清单

这一节按服务器实际输出整理：

```bash
cd /work/home/panada/insar/proc/alos2_stack
find . -maxdepth 2 -name "*.slurm" | sort
```

实际存在的 Slurm 文件：

```text
./mintpy/run_mintpy_all.slurm
./mintpy/run_mintpy_load.slurm
./run_cmd1.slurm
./run_cmd1_tail.slurm
./run_diff_array.slurm
./run_filt_array.slurm
./run_form_array.slurm
./run_geo2rdr_array.slurm
./run_geocode_array.slurm
./run_geocode_los.slurm
./run_ion_array.slurm
./run_ion_cal_array.slurm
./run_ion_check.slurm
./run_ion_correct_array.slurm
./run_ion_filt_array.slurm
./run_ion_ls.slurm
./run_ion_pairup.slurm
./run_ion_prep_array.slurm
./run_ion_subband_230412_231206.slurm
./run_ion_subband_one.slurm
./run_ion_subband_retry.slurm
./run_ion_unwrap_array.slurm
./run_look_array.slurm
./run_look_geom.slurm
./run_mosaic_array.slurm
./run_offset_array.slurm
./run_quicklook_gdal.slurm
./run_quicklook.slurm
./run_radar_dem_offset.slurm
./run_rect_range_offset.slurm
./run_resample_array.slurm
./run_smd1.slurm
./run_unwrap_array.slurm
```

换区域时，建议先把成功目录复制成新目录，再统一替换路径、日期和 pair。

### 8.1 DEM / 环境 / 安装类

| Slurm 文件 | 作用 | 是否可自动 | 换区域要改 |
|---|---|---|---|
| `run_quicklook.slurm` | 早期尝试生成 quicklook 图 | 可选 | 输入文件路径、输出目录 |
| `run_quicklook_gdal.slurm` | GDAL 方式生成 quicklook | 可选 | 输入文件路径、输出目录 |

说明：

```text
DEM 下载脚本不在当前 alos2_stack 目录的 find 输出中，但新区域仍然需要单独准备 DEM。
MintPy 安装脚本不需要每个区域保存/重跑，除非重新装环境。
quicklook 脚本不是主流程必要步骤，可以保留作参考。
```

### 8.2 ISCE2 cmd_1 相关

| Slurm 文件 | 对应步骤 | 作用 | 是否可自动 | 人工检查 |
|---|---|---|---|---|
| `run_cmd1.slurm` | `cmd_1.sh` | 早期直接跑完整 cmd1 | 不推荐全自动 | 容易因某一步失败导致后面连锁失败 |
| `run_cmd1_tail.slurm` | cmd1 后半段 | `mosaic_parameter` + `rdr2geo` | 可自动 | 检查 `lat/lon/hgt/los` |
| `run_offset_array.slurm` | `estimate_slc_offset.py` | 并行估计 SLC offset / cull.off | 可自动 | 检查每个非参考日期 `cull.off` |
| `run_resample_array.slurm` | `resample_common_grid.py` | 并行重采样日期到共同网格 | 可自动 | 检查 `dates_resampled` |
| `run_geo2rdr_array.slurm` 或 `run_geo2rdr.slurm` | `geo2rdr.py` | 对非参考日期并行计算几何 offset | 可自动 | 检查各日期 `*_rg.off` / `*_az.off` |
| `run_smd1.slurm` | cmd1 相关补充/单步测试 | 视脚本内容 | 需要打开确认 | 保存作参考 |

本次对应检查命令：

```bash
find dates -name cull.off | sort
find dates_resampled -path "*/insar/*_rg.off" -type f | sort
find dates_resampled -path "*/insar/*_az.off" -type f | sort
find dates_resampled/CHANGE_REFERENCE_DATE/insar -name "*lat*" -o -name "*lon*" -o -name "*hgt*" -o -name "*los*"
```

换区域必须改：

```text
CHANGE: 处理目录
CHANGE: reference date
CHANGE: dates2 数组
CHANGE: nrlks1/nalks1，如果你改了 looks
```

### 8.3 ISCE2 cmd_2 相关

| Slurm 文件 | 对应步骤 | 作用 | 是否可自动 | 人工检查 |
|---|---|---|---|---|
| `run_form_array.slurm` | `form_interferogram.py` | 14 个 pair 并行生成干涉图 | 可自动 | 检查 `.int/.amp` |
| `run_mosaic_array.slurm` | `mosaic_interferogram.py` | mosaic 单 frame/swath 结果 | 可自动 | 检查 pair 的 `insar` 文件 |
| `run_radar_dem_offset.slurm` | `rdr_dem_offset.py` | 估计雷达/DEM offset，生成 affine transform | 建议单独跑 | 检查 `affine_transform.txt` |
| `run_rect_range_offset.slurm` | `rect_range_offset.py` | 矫正 range offset | 可自动 | 检查 `*_rg_rect.off` |
| `run_diff_array.slurm` | `diff_interferogram.py` | 生成差分干涉图 | 可自动 | 检查 `diff_*_8rlks_12alks.int` |
| `run_look_array.slurm` | `look_coherence.py` | 多视并生成相干性 | 可自动 | 检查 `.cor/.amp` |
| `run_look_geom.slurm` | `look_geom.py` | 多视几何文件 | 可自动，依赖 look array | 检查 `8rlks_12alks.{lat,lon,hgt,los,wbd}` |

本次检查命令：

```bash
find pairs -path "*/insar/*_8rlks_12alks.cor" -type f | wc -l
find pairs -path "*/insar/diff_*_8rlks_12alks.int" -type f | wc -l
find dates_resampled/CHANGE_REFERENCE_DATE/insar -name "*8rlks_12alks*" | sort
cat dates_resampled/CHANGE_REFERENCE_DATE/insar/affine_transform.txt
```

换区域必须改：

```text
CHANGE: insarpair 数组
CHANGE: reference date
CHANGE: 处理目录
CHANGE: array 范围，例如 0-13 对应 14 个 pair
```

### 8.4 cmd_3 电离层校正相关

| Slurm 文件 | 对应步骤 | 作用 | 是否可自动 | 人工检查 |
|---|---|---|---|---|
| `run_ion_array.slurm` 或 `ion_array` | `ion_subband.py` 等 | 早期把 ion 子步骤按 pair array 跑 | 部分可自动 | 检查失败 pair |
| `run_ion_pairup.slurm` | ion pair 准备 | 准备 `pairs_ion` 的 pair 目录/链接 | 可自动 | 检查 `pairs_ion` 数量 |
| `run_ion_subband_retry.slurm` | `ion_subband.py` | 针对失败 pair 重跑 subband | 手动触发 | 检查对应 `pairs_ion` |
| `run_ion_subband_one.slurm` | 单 pair subband | 单独重跑一个 pair | 手动触发 | 检查 log |
| `run_ion_subband_230412_231206.slurm` | 单 pair subband | 本次补跑失败 pair | 手动触发 | 检查 log |
| `run_ion_unwrap_array.slurm` | ion unwrap | 电离层子带解缠 | 可自动 | 检查 err 是否只有 processing line |
| `run_ion_filt_array.slurm` | ion filt | 电离层滤波 | 可自动 | 检查 ion 滤波输出 |
| `run_ion_prep_array.slurm` | ion prep | 准备 ion 估计输入 | 可自动 | 检查输出 |
| `run_ion_cal_array.slurm` | ion cal | 计算每个 pair 电离层相位 | 可自动 | 检查 `ion_cal` 输出 |
| `run_ion_check.slurm` | ion check | 生成 `fig_ion/*.tif` | 必须人工看图 | 筛选长波条纹坏 pair |
| `run_ion_ls.slurm` | ion least squares | 用 pair 反演各日期电离层 | 人工筛查后自动 | 检查 rank / dates_ion |
| `run_ion_correct_array.slurm` | ion correct | 把电离层校正应用回 `pairs` | 可自动 | 检查 `diff_*_ori.int` |

本次人工筛查点：

```text
fig_ion/*.tif 需要人工看。
长波条纹严重、明显异常的 pair 可以在 ion_ls 前排除。
本次最终没有排除 pair，14 个 pair 全部用于 ion_ls。
```

本次检查命令：

```bash
find fig_ion -type f | sort
find dates_ion -type f | sort | head
find pairs -path "*/insar/diff_*_8rlks_12alks_ori.int" -type f | wc -l
```

换区域必须改：

```text
CHANGE: pair 数组
CHANGE: reference date
CHANGE: 处理目录
CHANGE: 如果排除 pair，ion_ls 脚本中的 used pairs
```

### 8.5 cmd_4 滤波 / 解缠 / 地理编码相关

| Slurm 文件 | 对应步骤 | 作用 | 是否可自动 | 人工检查 |
|---|---|---|---|---|
| `run_filt_array.slurm` | `filt.py` | 对 14 个 pair 滤波 | 可自动 | 检查 `filt_*.int` |
| `run_unwrap_array.slurm` | `unwrap_snaphu.py` | 对 14 个 pair 解缠 | 可自动但需人工质检 | 检查 `.unw` / `_msk.unw` |
| `run_geocode_array.slurm` | `geocode.py` | 对 pair 结果地理编码 | 可自动 | 检查 `.geo` |
| `run_geocode_los.slurm` | `geocode.py` | 单独地理编码 LOS | 可自动 | 检查 `los.geo` |

本次检查命令：

```bash
find pairs -path "*/insar/filt_*_8rlks_12alks.int" -type f | wc -l
find pairs -path "*/insar/filt_*_8rlks_12alks.unw" -type f | wc -l
find pairs -path "*/insar/filt_*_8rlks_12alks_msk.unw" -type f | wc -l
find pairs -path "*/insar/*_8rlks_12alks*.geo*" -type f | sort | head
find dates_resampled/CHANGE_REFERENCE_DATE/insar -type f | grep "los.geo"
```

人工检查点：

```text
解缠结果必须看。
如果某些 pair 出现大片跳变、明显断裂或异常孤岛，要考虑剔除该 pair 或重跑 unwrap。
```

### 8.6 MintPy 相关

| Slurm 文件 | 作用 | 是否可自动 | 人工检查 |
|---|---|---|---|
| `run_mintpy_load.slurm` | 只跑 `load_data`，生成 `inputs/*.h5` | 可自动 | 检查 ifgramStack / geometryRadar |
| `run_mintpy_all.slurm` | 早期一次跑完整 MintPy | 不推荐 | 最后 plot 报错会导致 Slurm 显示 FAILED |
| `run_mintpy_01_load.slurm` | 推荐新增依赖链第 1 步 | 可自动 | 检查 inputs |
| `run_mintpy_02_ts_velocity.slurm` | 推荐新增依赖链第 2 步，时序 + velocity | 可自动 | 检查 h5 大小和 temporal coherence |
| `run_mintpy_03_geo_kmz.slurm` | 推荐新增依赖链第 3 步，geocode + kmz | 可自动 | 检查 geo 文件 |

注意：`run_mintpy_01_load.slurm` / `02` / `03` 是推荐新增模板，不在你当前 `find` 输出里；当前真实存在的是 `mintpy/run_mintpy_load.slurm` 和 `mintpy/run_mintpy_all.slurm`。

本次 MintPy 成功结果：

```text
timeseries.h5          244M
velocity.h5            204M
temporalCoherence.h5    41M
avgSpatialCoh.h5        41M
geo/geo_timeseries.h5
geo/geo_velocity.h5
geo/geo_temporalCoherence.h5
geo/geo_avgSpatialCoh.h5
geo/geo_velocity.kmz
```

本次重要环境修复：

```text
MintPy 1.6.2 + NumPy 2.4.6 加权反演失败。
降到 NumPy 1.26.4 后，fim 加权反演成功。
```

检查：

```bash
python -c "import mintpy, numpy; print('mintpy', mintpy.__version__); print('numpy', numpy.__version__)"
```

### 8.7 一次性归档所有 Slurm

在成功处理目录运行：

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW
mkdir -p /work/home/panada/insar/archive/alos2_NEW_timeseries_YYYYMMDD/slurm

cp *.slurm /work/home/panada/insar/archive/alos2_NEW_timeseries_YYYYMMDD/slurm/ 2>/dev/null
cp mintpy/*.slurm /work/home/panada/insar/archive/alos2_NEW_timeseries_YYYYMMDD/slurm/ 2>/dev/null
```

建议同时保存 Slurm 作业日志：

```bash
mkdir -p /work/home/panada/insar/archive/alos2_NEW_timeseries_YYYYMMDD/logs
cp *.log *.err /work/home/panada/insar/archive/alos2_NEW_timeseries_YYYYMMDD/logs/ 2>/dev/null
cp mintpy/*.log mintpy/*.err /work/home/panada/insar/archive/alos2_NEW_timeseries_YYYYMMDD/logs/ 2>/dev/null
```

如果要列出当前目录全部 Slurm 文件：

```bash
find /work/home/panada/insar/proc/alos2_stack_NEW -maxdepth 2 -name "*.slurm" | sort
```

## 8. 自动/人工节点总览

### 可以自动串起来的步骤

这些可以用 Slurm dependency 自动往下跑：

```text
DEM 已准备好以后：
create_cmds
cmd1 的可并行子步骤
cmd2 的 form/mosaic/diff/look
cmd4 的 filt/unwrap/geocode
MintPy load_data → invert/velocity → geocode/kmz
```

### 建议人工检查后再继续的步骤

这些不建议完全自动跳过：

```text
1. 原始数据解压和日期目录检查
2. DEM / water mask 检查
3. create_cmds 后检查 cmd_*.sh，尤其 water body / use_wbd_offset
4. cmd1 后检查 cull.off、dates_resampled、lat/lon/hgt/los
5. cmd2 后检查 coherence / interferogram 是否都有 14 个
6. ion_check 后查看电离层长波条纹，决定是否排除 pair
7. cmd4 解缠后检查 unwrap 是否明显失败
8. MintPy 后检查 temporalCoherence / velocity / timeseries
```

<a id="step-9"></a>

## 9. ISCE2 Slurm 模板

以下脚本均假设在：

```bash
/work/home/panada/insar/proc/alos2_stack_NEW
```

### 9.1 通用 ISCE2 脚本头

所有 ISCE2 脚本都建议包含：

```bash
source /etc/profile
module purge
source /work/home/panada/env/scons_env/bin/activate
source /work/home/panada/isce2_env.sh

export ISCE_STACK=/work/home/panada/isce/isce2-main/contrib/stack
export PYTHONPATH=/work/home/panada/isce_pkg:/work/home/panada/isce_pkg/isce/components:$PYTHONPATH
export PATH=/work/home/panada/isce/isce2-install/applications:/work/home/panada/isce/isce2-install/bin:/work/home/panada/local/bin:/work/home/panada/local/gcc-10/bin:$PATH
export LD_LIBRARY_PATH=/public/software/mathlib/hdf5/1.12.2/gnu/lib:/work/home/panada/local/lib:/work/home/panada/local/lib64:/work/home/panada/local/gcc-10/lib64:/opt/rh/devtoolset-7/root/usr/lib64:/opt/rh/devtoolset-7/root/usr/lib/gcc/x86_64-redhat-linux/7:/work/home/panada/local/python3.6/lib:$LD_LIBRARY_PATH
```

### 9.2 cmd1 推荐拆分

cmd1 最慢/最容易出错的是 offset、resample、rdr2geo、geo2rdr。

人工检查：

```bash
find dates -name cull.off | sort
find dates_resampled -path "*/insar/*" -type f | sort | head
find dates_resampled/221221/insar -name "*lat*" -o -name "*lon*" -o -name "*hgt*"
```

如果要并行处理 geo2rdr，可用日期数组：

```bash
#SBATCH --job-name=alos_geo2rdr
#SBATCH --output=%x_%A_%a.log
#SBATCH --error=%x_%A_%a.err
#SBATCH -p wzhctest
#SBATCH -c 8
#SBATCH --mem=16G
#SBATCH --array=0-4%3
#SBATCH --export=NONE

set -e
source /etc/profile
module purge
source /work/home/panada/env/scons_env/bin/activate
source /work/home/panada/isce2_env.sh

export ISCE_STACK=/work/home/panada/isce/isce2-main/contrib/stack
export PYTHONPATH=/work/home/panada/isce_pkg:/work/home/panada/isce_pkg/isce/components:$PYTHONPATH
export LD_LIBRARY_PATH=/public/software/mathlib/hdf5/1.12.2/gnu/lib:/work/home/panada/local/lib:/work/home/panada/local/lib64:/work/home/panada/local/gcc-10/lib64:/opt/rh/devtoolset-7/root/usr/lib64:/opt/rh/devtoolset-7/root/usr/lib/gcc/x86_64-redhat-linux/7:/work/home/panada/local/python3.6/lib:$LD_LIBRARY_PATH
export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK

cd /work/home/panada/insar/proc/alos2_stack_NEW

dates2=(CHANGE_SECONDARY_DATES_EXCLUDING_REFERENCE)
d=${dates2[$SLURM_ARRAY_TASK_ID]}

cd dates_resampled/$d
${ISCE_STACK}/alosStack/geo2rdr.py \
  -date $d \
  -date_par_dir ../../dates/$d \
  -lat ../CHANGE_REFERENCE_DATE/insar/CHANGE_REFERENCE_DATE_2rlks_3alks.lat \
  -lon ../CHANGE_REFERENCE_DATE/insar/CHANGE_REFERENCE_DATE_2rlks_3alks.lon \
  -hgt ../CHANGE_REFERENCE_DATE/insar/CHANGE_REFERENCE_DATE_2rlks_3alks.hgt \
  -nrlks1 2 \
  -nalks1 3
```

需要改：

```text
CHANGE_SECONDARY_DATES_EXCLUDING_REFERENCE
CHANGE_REFERENCE_DATE
处理目录 alos2_stack_NEW
```

### 9.3 cmd2 推荐拆分

cmd2 包含：

```text
form_interferogram
mosaic_interferogram
rdr_dem_offset / affine_transform
rect_range_offset
diff_interferogram
look_coherence
look_geom
```

pair array 模板：

```bash
#SBATCH --job-name=alos_diff
#SBATCH --output=%x_%A_%a.log
#SBATCH --error=%x_%A_%a.err
#SBATCH -p wzhctest
#SBATCH -c 4
#SBATCH --mem=8G
#SBATCH --array=0-CHANGE_NPAIR_MINUS_1%8
#SBATCH --export=NONE

set -e
source /etc/profile
module purge
source /work/home/panada/env/scons_env/bin/activate
source /work/home/panada/isce2_env.sh

export ISCE_STACK=/work/home/panada/isce/isce2-main/contrib/stack
export PYTHONPATH=/work/home/panada/isce_pkg:/work/home/panada/isce_pkg/isce/components:$PYTHONPATH
export LD_LIBRARY_PATH=/public/software/mathlib/hdf5/1.12.2/gnu/lib:/work/home/panada/local/lib:/work/home/panada/local/lib64:/work/home/panada/local/gcc-10/lib64:/opt/rh/devtoolset-7/root/usr/lib64:/opt/rh/devtoolset-7/root/usr/lib/gcc/x86_64-redhat-linux/7:/work/home/panada/local/python3.6/lib:$LD_LIBRARY_PATH

cd /work/home/panada/insar/proc/alos2_stack_NEW

insarpair=(
CHANGE_PAIR_1
CHANGE_PAIR_2
)

pair=${insarpair[$SLURM_ARRAY_TASK_ID]}
IFS='-' read -ra dates <<< "$pair"
ref_date=${dates[0]}
sec_date=${dates[1]}

cd pairs/$pair
${ISCE_STACK}/alosStack/diff_interferogram.py \
  -idir ../../dates_resampled \
  -ref_date_stack CHANGE_REFERENCE_DATE \
  -ref_date ${ref_date} \
  -sec_date ${sec_date} \
  -nrlks1 2 \
  -nalks1 3
```

人工检查：

```bash
find pairs -path "*/insar/*_8rlks_12alks.cor" -type f | wc -l
find pairs -path "*/insar/diff_*_8rlks_12alks.int" -type f | wc -l
ls -lh dates_resampled/CHANGE_REFERENCE_DATE/insar/*8rlks_12alks*
```

### 9.4 cmd3 电离层校正

cmd3 建议分步跑，但中间必须人工看图。

自动步骤：

```text
ion_subband
ion_unwrap
ion_filt
ion_prep
ion_cal
ion_check
```

人工步骤：

```text
查看 fig_ion/*.tif，判断哪些 pair 电离层估计差
```

本次经验：

```text
220316-220413: 可保留
230315-230412: 可保留
220316-221221: 有点可疑，但最终未排除
```

ion_check 需要 `mdx`。如果 `mdx: command not found`，需要添加：

```bash
export PATH=/work/home/panada/miniforge3/pkgs/https/conda.anaconda.org/conda-forge/linux-64/isce2-2.6.4-py310h6627766_2/lib/python3.10/site-packages/isce/bin:$PATH
export LD_LIBRARY_PATH=/work/home/panada/miniforge3/pkgs/https/conda.anaconda.org/conda-forge/linux-64/libgfortran5-15.2.0-h68bc16d_19/lib:$LD_LIBRARY_PATH
```

ion_ls 成功标志：

```text
observation matrix rank 正常
dates_ion/filt_ion_日期_8rlks_12alks.ion 生成
```

ion_correct 成功后：

```text
pairs/*/insar/diff_*_8rlks_12alks.int 被电离层校正
pairs/*/insar/diff_*_8rlks_12alks_ori.int 保存原始版本
```

### 9.5 cmd4 滤波 / 解缠 / 地理编码

cmd4 包含：

```text
filt.py
unwrap_snaphu.py
geocode.py
```

滤波成功检查：

```bash
find pairs -path "*/insar/filt_*_8rlks_12alks.int" -type f | wc -l
```

解缠成功检查：

```bash
find pairs -path "*/insar/filt_*_8rlks_12alks.unw" -type f | wc -l
find pairs -path "*/insar/filt_*_8rlks_12alks_msk.unw" -type f | wc -l
```

地理编码成功检查：

```bash
find pairs -path "*/insar/*_8rlks_12alks*.geo*" -type f | sort | head
```

单独 geocode LOS：

```bash
find dates_resampled/CHANGE_REFERENCE_DATE/insar -type f | grep "los.geo"
```

人工检查：

```text
至少检查 coherence、filtered interferogram、unwrapped phase
如果解缠出现大片跳变/孤岛异常，需要回到 pair 选择或相干性检查
```

<a id="step-10"></a>

## 10. MintPy 流程

### 10.1 MintPy 配置文件

路径：

```bash
/work/home/panada/insar/proc/alos2_stack_NEW/mintpy/alos2_NEW.txt
```

模板：

```text
mintpy.load.processor        = isce

mintpy.load.metaFile         = /work/home/panada/insar/proc/alos2_stack_NEW/dates_resampled/CHANGE_REFERENCE_DATE/CHANGE_REFERENCE_DATE.track.xml
mintpy.load.baselineDir      = /work/home/panada/insar/proc/alos2_stack_NEW/baseline

mintpy.load.unwFile          = /work/home/panada/insar/proc/alos2_stack_NEW/pairs/*-*/insar/filt_*-*_8rlks_12alks.unw
mintpy.load.corFile          = /work/home/panada/insar/proc/alos2_stack_NEW/pairs/*-*/insar/*-*_8rlks_12alks.cor
mintpy.load.connCompFile     = None

mintpy.load.demFile          = /work/home/panada/insar/proc/alos2_stack_NEW/dates_resampled/CHANGE_REFERENCE_DATE/insar/CHANGE_REFERENCE_DATE_8rlks_12alks.hgt
mintpy.load.lookupYFile      = /work/home/panada/insar/proc/alos2_stack_NEW/dates_resampled/CHANGE_REFERENCE_DATE/insar/CHANGE_REFERENCE_DATE_8rlks_12alks.lat
mintpy.load.lookupXFile      = /work/home/panada/insar/proc/alos2_stack_NEW/dates_resampled/CHANGE_REFERENCE_DATE/insar/CHANGE_REFERENCE_DATE_8rlks_12alks.lon
mintpy.load.incAngleFile     = /work/home/panada/insar/proc/alos2_stack_NEW/dates_resampled/CHANGE_REFERENCE_DATE/insar/CHANGE_REFERENCE_DATE_8rlks_12alks.los
mintpy.load.azAngleFile      = /work/home/panada/insar/proc/alos2_stack_NEW/dates_resampled/CHANGE_REFERENCE_DATE/insar/CHANGE_REFERENCE_DATE_8rlks_12alks.los
mintpy.load.waterMaskFile    = /work/home/panada/insar/proc/alos2_stack_NEW/dates_resampled/CHANGE_REFERENCE_DATE/insar/CHANGE_REFERENCE_DATE_8rlks_12alks.wbd

mintpy.reference.date        = CHANGE_REFERENCE_DATE_YYYYMMDD_OR_YYMMDD
mintpy.reference.lalo        = auto

mintpy.troposphericDelay.method = no
mintpy.deramp                   = no
mintpy.topographicResidual      = no
mintpy.networkInversion.weightFunc = fim
```

本次加权成功使用：

```text
mintpy.networkInversion.weightFunc = fim
numpy = 1.26.4
```

### 10.2 MintPy 自动依赖链

这是推荐的一键提交方式。

#### run_mintpy_01_load.slurm

```bash
#!/bin/bash
#SBATCH --job-name=mintpy_load
#SBATCH --output=%x_%j.log
#SBATCH --error=%x_%j.err
#SBATCH -p wzhctest
#SBATCH -c 8
#SBATCH --mem=16G
#SBATCH --time=01:00:00
#SBATCH --export=NONE

set -e
source /etc/profile
module purge
source /work/home/panada/MintPY/activate_mintpy.sh

export PYTHONPATH=/work/home/panada/isce_pkg:/work/home/panada/isce_pkg/isce/components:$PYTHONPATH
export LD_LIBRARY_PATH=/public/software/mathlib/hdf5/1.12.2/gnu/lib:/work/home/panada/local/lib64:/work/home/panada/local/lib:/work/home/panada/local/gcc-10/lib64:/opt/rh/devtoolset-7/root/usr/lib64:/opt/rh/devtoolset-7/root/usr/lib/gcc/x86_64-redhat-linux/7:/work/home/panada/local/python3.6/lib:$LD_LIBRARY_PATH

cd /work/home/panada/insar/proc/alos2_stack_NEW/mintpy

smallbaselineApp.py alos2_NEW.txt --dostep load_data
ls -lh inputs/ifgramStack.h5 inputs/geometryRadar.h5
```

#### run_mintpy_02_ts_velocity.slurm

```bash
#!/bin/bash
#SBATCH --job-name=mintpy_ts
#SBATCH --output=%x_%j.log
#SBATCH --error=%x_%j.err
#SBATCH -p wzhctest
#SBATCH -c 16
#SBATCH --mem=32G
#SBATCH --time=04:00:00
#SBATCH --export=NONE

set -e
source /etc/profile
module purge
source /work/home/panada/MintPY/activate_mintpy.sh

export PYTHONPATH=/work/home/panada/isce_pkg:/work/home/panada/isce_pkg/isce/components:$PYTHONPATH
export LD_LIBRARY_PATH=/public/software/mathlib/hdf5/1.12.2/gnu/lib:/work/home/panada/local/lib64:/work/home/panada/local/lib:/work/home/panada/local/gcc-10/lib64:/opt/rh/devtoolset-7/root/usr/lib64:/opt/rh/devtoolset-7/root/usr/lib/gcc/x86_64-redhat-linux/7:/work/home/panada/local/python3.6/lib:$LD_LIBRARY_PATH

export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK
export OPENBLAS_NUM_THREADS=$SLURM_CPUS_PER_TASK
export MKL_NUM_THREADS=$SLURM_CPUS_PER_TASK
export NUMEXPR_NUM_THREADS=$SLURM_CPUS_PER_TASK

cd /work/home/panada/insar/proc/alos2_stack_NEW/mintpy

smallbaselineApp.py alos2_NEW.txt --end velocity
ls -lh timeseries.h5 velocity.h5 temporalCoherence.h5 avgSpatialCoh.h5
```

#### run_mintpy_03_geo_kmz.slurm

```bash
#!/bin/bash
#SBATCH --job-name=mintpy_geo
#SBATCH --output=%x_%j.log
#SBATCH --error=%x_%j.err
#SBATCH -p wzhctest
#SBATCH -c 8
#SBATCH --mem=16G
#SBATCH --time=01:30:00
#SBATCH --export=NONE

set -e
source /etc/profile
module purge
source /work/home/panada/MintPY/activate_mintpy.sh

export PYTHONPATH=/work/home/panada/isce_pkg:/work/home/panada/isce_pkg/isce/components:$PYTHONPATH
export LD_LIBRARY_PATH=/public/software/mathlib/hdf5/1.12.2/gnu/lib:/work/home/panada/local/lib64:/work/home/panada/local/lib:/work/home/panada/local/gcc-10/lib64:/opt/rh/devtoolset-7/root/usr/lib64:/opt/rh/devtoolset-7/root/usr/lib/gcc/x86_64-redhat-linux/7:/work/home/panada/local/python3.6/lib:$LD_LIBRARY_PATH

cd /work/home/panada/insar/proc/alos2_stack_NEW/mintpy

smallbaselineApp.py alos2_NEW.txt --dostep geocode
smallbaselineApp.py alos2_NEW.txt --dostep google_earth

ls -lh geo/geo_timeseries.h5 geo/geo_velocity.h5 geo/geo_temporalCoherence.h5 geo/geo_avgSpatialCoh.h5 geo/geo_velocity.kmz
```

#### 一次性提交

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW/mintpy

jid1=$(sbatch --parsable run_mintpy_01_load.slurm)
jid2=$(sbatch --parsable --dependency=afterok:$jid1 run_mintpy_02_ts_velocity.slurm)
jid3=$(sbatch --parsable --dependency=afterok:$jid2 run_mintpy_03_geo_kmz.slurm)

echo "load      job: $jid1"
echo "ts/vel    job: $jid2"
echo "geo/kmz   job: $jid3"
```

检查：

```bash
squeue -u panada
sacct -j $jid1,$jid2,$jid3 --format=JobID,JobName,State,ExitCode,Elapsed,MaxRSS
```

<a id="step-11"></a>

## 11. 最终结果说明

MintPy 主结果：

```text
timeseries.h5          每个日期相对参考日期的 LOS 累计形变，单位 m
velocity.h5            平均 LOS 形变速率，单位 m/year
temporalCoherence.h5   时间相干性，0-1，越高越可靠
avgSpatialCoh.h5       平均空间相干性，0-1
```

地理编码结果：

```text
geo/geo_timeseries.h5
geo/geo_velocity.h5
geo/geo_temporalCoherence.h5
geo/geo_avgSpatialCoh.h5
geo/geo_velocity.kmz
```

QGIS 中：

```text
geo_velocity.h5 选择 velocity
geo_timeseries.h5 的 Band 1/2/3... 对应日期顺序
```

本次日期顺序：

```text
Band 1 = 20220316
Band 2 = 20220413
Band 3 = 20221221
Band 4 = 20230315
Band 5 = 20230412
Band 6 = 20231206
```

单位换算：

```text
m      × 100 = cm
m/year × 100 = cm/year
```

<a id="step-12"></a>

## 12. 归档建议

处理成功后，先归档配置、脚本、日志和核心结果。

```bash
mkdir -p /work/home/panada/insar/archive/alos2_NEW_timeseries_YYYYMMDD

cd /work/home/panada/insar/proc/alos2_stack_NEW

cp alosStack.xml cmd_*.sh *.slurm /work/home/panada/insar/archive/alos2_NEW_timeseries_YYYYMMDD/ 2>/dev/null
cp *.log *.err /work/home/panada/insar/archive/alos2_NEW_timeseries_YYYYMMDD/ 2>/dev/null

mkdir -p /work/home/panada/insar/archive/alos2_NEW_timeseries_YYYYMMDD/mintpy
cp mintpy/*.txt mintpy/*.cfg mintpy/*.slurm mintpy/*.log mintpy/*.err /work/home/panada/insar/archive/alos2_NEW_timeseries_YYYYMMDD/mintpy/ 2>/dev/null
```

打包 MintPy 结果：

```bash
tar czf /work/home/panada/insar/archive/alos2_NEW_timeseries_YYYYMMDD/mintpy_results.tar.gz \
  mintpy/timeseries.h5 \
  mintpy/velocity.h5 \
  mintpy/temporalCoherence.h5 \
  mintpy/avgSpatialCoh.h5 \
  mintpy/geo \
  mintpy/inputs \
  mintpy/*.txt \
  mintpy/*.cfg
```

<a id="step-13"></a>

## 13. 换区域最快流程

最短可复用流程：

```text
1. 新建 data/proc/dem 目录
2. 解压并按日期整理 ALOS-2 数据
3. 下载/准备 DEM 和 water mask
4. 修改 alosStack.xml
5. create_cmds.py
6. 检查 cmd_1.sh 里的 water body / use_wbd_offset
7. 跑 ISCE2 cmd1/cmd2/cmd3/cmd4 的 Slurm
8. ion_check 人工筛查
9. 解缠结果人工检查
10. 修改 MintPy 配置
11. MintPy dependency chain 一次性提交
12. QGIS / MintPy 查看结果
13. 归档
```

<a id="step-14"></a>

## 14. DInSAR 与时序的关系

如果新区域只做 DInSAR，不一定要跑完整 MintPy。DInSAR 是单个 pair：

```text
两期 ALOS-2
→ 配准
→ 干涉
→ 去地形相位
→ 滤波
→ 解缠
→ 地理编码
```

时序流程是多个 pair：

```text
多个干涉对
→ MintPy 网络反演
→ timeseries.h5 / velocity.h5
```

建议：

```text
先用本模板复制新区域流程；
如果只做 DInSAR，先选一对高相干日期跑 cmd1/cmd2/cmd4；
如果要稳定形变趋势，再跑完整 stack + MintPy。
```

<a id="step-15"></a>

## 15. 自动化方式：哪些可以自动检查后继续

### 15.1 Slurm dependency 的基本逻辑

Slurm 可以用依赖链自动往下跑：

```bash
jid1=$(sbatch --parsable step1.slurm)
jid2=$(sbatch --parsable --dependency=afterok:$jid1 step2.slurm)
jid3=$(sbatch --parsable --dependency=afterok:$jid2 step3.slurm)
```

含义：

```text
step1 成功，才跑 step2
step2 成功，才跑 step3
如果某一步失败，后面的任务不会跑
```

### 15.2 自动检查脚本的做法

每个 Slurm 脚本末尾都可以放检查命令。只要检查失败，脚本 `exit 1`，后续依赖任务就不会继续。

例如 cmd2 的 diff 后检查 14 个结果：

```bash
expected=14
actual=$(find pairs -path "*/insar/diff_*_8rlks_12alks.int" -type f | wc -l)
echo "diff interferograms: $actual / $expected"
if [ "$actual" -ne "$expected" ]; then
    echo "ERROR: diff interferogram count mismatch"
    exit 1
fi
```

例如 MintPy load_data 检查：

```bash
test -s inputs/ifgramStack.h5
test -s inputs/geometryRadar.h5
```

### 15.3 推荐自动链

以下步骤可以自动串起来：

```text
run_offset_array.slurm
→ run_resample_array.slurm
→ run_cmd1_tail.slurm
→ run_geo2rdr_array.slurm
→ run_form_array.slurm
→ run_mosaic_array.slurm
→ run_radar_dem_offset.slurm
→ run_rect_range_offset.slurm
→ run_diff_array.slurm
→ run_look_array.slurm
→ run_look_geom.slurm
```

但中间建议在这些节点停下来人工看：

```text
cmd1 完成后：检查 dates_resampled / geometry 是否完整
cmd2 完成后：检查 coherence 和 diff interferogram
cmd3 ion_check 后：必须看 fig_ion
cmd4 unwrap 后：必须看解缠质量
MintPy 完成后：检查 temporalCoherence / velocity 合理性
```

所以推荐实际自动链分成四段：

```text
自动段 A：cmd1 到几何准备
人工检查 A：lat/lon/hgt/offset 是否完整

自动段 B：cmd2 到 look_geom
人工检查 B：干涉图 / 相干性 / affine_transform

自动段 C：cmd3 到 ion_check
人工检查 C：fig_ion 长波条纹筛查

自动段 D：ion_ls → ion_correct → cmd4 → MintPy
人工检查 D：unwrap / MintPy 最终结果
```

<a id="step-16"></a>

## 16. 换区域一键替换脚本模板

可以做一个“半自动”替换脚本。它不能替你判断 pair 好坏，但可以把目录、参考日期、pair 数组等批量替换掉。

强烈建议：只在复制出来的新目录运行，绝不要在成功案例原目录直接运行。

### 16.1 推荐复制新项目

```bash
OLD=/work/home/panada/insar/proc/alos2_stack
NEW=/work/home/panada/insar/proc/alos2_stack_NEW

mkdir -p "$NEW"
cp "$OLD"/*.slurm "$NEW"/
cp "$OLD"/alosStack.xml "$NEW"/
cp "$OLD"/cmd_*.sh "$NEW"/ 2>/dev/null
mkdir -p "$NEW/mintpy"
cp "$OLD"/mintpy/*.slurm "$NEW"/mintpy/ 2>/dev/null
cp "$OLD"/mintpy/*.txt "$NEW"/mintpy/ 2>/dev/null
cp "$OLD"/mintpy/*.cfg "$NEW"/mintpy/ 2>/dev/null
```

### 16.2 一键替换脚本

在新目录写：

```bash
nano retarget_project.sh
```

内容：

```bash
#!/bin/bash
set -e

# ===== CHANGE THESE =====
OLD_PROC="/work/home/panada/insar/proc/alos2_stack"
NEW_PROC="/work/home/panada/insar/proc/alos2_stack_NEW"

OLD_NAME="alos2_noto"
NEW_NAME="alos2_NEW"

OLD_REF="221221"
NEW_REF="CHANGE_REF_YYMMDD"

OLD_REF8="20221221"
NEW_REF8="CHANGE_REF_YYYYMMDD"

OLD_DEM="/work/home/panada/insar/data/dem/dem_aw3d30.dem.wgs84"
NEW_DEM="/work/home/panada/insar/data/dem/NEW/CHANGE.dem.wgs84"

OLD_WBD="/work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd"
NEW_WBD="/work/home/panada/insar/data/dem/NEW/CHANGE.wbd"

OLD_DATA="/work/home/panada/insar/data/alos2"
NEW_DATA="/work/home/panada/insar/data/alos2_NEW"

# ===== DO NOT EDIT BELOW unless needed =====
cd "$NEW_PROC"

files=$(find . -maxdepth 3 \( -name "*.slurm" -o -name "*.xml" -o -name "*.txt" -o -name "*.cfg" -o -name "cmd_*.sh" \))

for f in $files; do
    sed -i \
      -e "s#${OLD_PROC}#${NEW_PROC}#g" \
      -e "s#${OLD_NAME}#${NEW_NAME}#g" \
      -e "s#${OLD_REF8}#${NEW_REF8}#g" \
      -e "s#${OLD_REF}#${NEW_REF}#g" \
      -e "s#${OLD_DEM}#${NEW_DEM}#g" \
      -e "s#${OLD_WBD}#${NEW_WBD}#g" \
      -e "s#${OLD_DATA}#${NEW_DATA}#g" \
      "$f"
done

echo "Done basic replacement."
echo "Now manually check dates arrays, pair arrays, DEM/WBD files, and alosStack.xml."
grep -R -n "alos2_stack\|reference date\|dem for\|water body\|insarpair=\|dates2=" .
```

运行：

```bash
chmod +x retarget_project.sh
./retarget_project.sh
```

### 16.3 为什么不能完全一键

这些必须人工决定：

```text
新区域有哪些日期
哪个日期作为参考日期
哪些 pair 要处理
哪些 pair 电离层图不好要排除
解缠是否成功
QGIS/MintPy 结果是否合理
```

所以可以“一键替换路径和旧日期”，但不能“一键保证科学结果正确”。

<a id="step-17"></a>

## 17. water mask 本次怎么来的、以后怎么做

本次 water mask 的情况：

```text
最初尝试用 ISCE2 的 wbd.py 下载 SRTMSWBD，但服务器/网址/Earthdata 出问题。
后来使用 ASF water mask 数据，下载后裁剪为 asf_watermask_clip.wbd。
最终路径：
/work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd
```

关键点：

```text
ISCE2 / MintPy 不只是需要一个 wbd 二进制文件，还需要对应的 .xml/.vrt 或 .rsc 元数据。
alosStack.xml 里的 <property name="water body"> 必须指向准确的 wbd 文件绝对路径。
MintPy load_data 后会把 reference date 下的 8rlks_12alks.wbd 写入 geometryRadar.h5 的 waterMask。
```

本次遇到的问题：

```text
fixImageXml.py 不能直接修 asf_watermask_clip.wbd，因为当时没有 asf_watermask_clip.wbd.xml。
后来流程能跑通，是因为 alosStack/rdr2geo/look_geom 生成了参考日期几何目录下的 wbd 产品：
dates_resampled/221221/insar/221221_8rlks_12alks.wbd
```

以后换区域建议：

```text
1. 准备覆盖研究区的 water mask。
2. 在 alosStack.xml 中设置：
   <property name="water body">/absolute/path/to/your.wbd</property>
3. 不要在 offset 阶段使用 -use_wbd_offset，除非确认 water mask 格式完全适配。
4. create_cmds.py 后检查 cmd_1.sh：
   grep -n "use_wbd_offset\|estimate_slc_offset" cmd_1.sh
5. MintPy 不直接读取原始 asf_watermask_clip.wbd，而是读取：
   dates_resampled/REF/insar/REF_8rlks_12alks.wbd
```

如果不想用 water mask：

```text
alosStack.xml 中 water body 可以设为 None。
但 geocoding / mask 质量会少一个水体约束。
```

<a id="step-18"></a>

## 18. 按本次真实 Slurm 审计后的准流程

这一节是根据你实际贴出的 33 个 `*.slurm`、`alosStack.xml` 和 `mintpy/alos2_noto.txt` 整理的。以后换区域时，优先看这一节。

<a id="step-18-1"></a>

### 18.1 真实 Slurm 分类

主线可复用脚本：

| 文件 | 当前核数/并发 | 作用 | 换区域必须改 |
|---|---:|---|---|
| `run_offset_array.slurm` | `4核 x 3并发` | 非参考日期 SLC offset，生成 `dates/*/cull.off` | `dates2`、数组范围、参考日期、DEM/WBD/处理目录 |
| `run_resample_array.slurm` | `4核 x 2并发` | 所有日期重采样到公共网格，生成 `dates_resampled` | `dates`、数组范围、处理目录 |
| `run_cmd1_tail.slurm` | `16核 x 1` | `mosaic_parameter` + 参考日期 `rdr2geo`，生成 `lat/lon/hgt/los/wbd` | 参考日期、日期列表、DEM/WBD/处理目录 |
| `run_geo2rdr_array.slurm` | `8核 x 3并发` | 非参考日期几何 offset，生成 `*_rg.off` / `*_az.off` | `dates2`、参考日期文件名、数组范围 |
| `run_form_array.slurm` | `4核 x 8并发` | 每个 pair 形成干涉图 | `insarpair`、数组范围 |
| `run_mosaic_array.slurm` | `4核 x 8并发` | 每个 pair 的 frame/swath mosaic | `insarpair`、数组范围 |
| `run_radar_dem_offset.slurm` | `16核 x 1` | 估计 radar/DEM affine transform | 参考日期、DEM/WBD、`-amp` 选择的 pair |
| `run_rect_range_offset.slurm` | `4核 x 5并发` | 矫正 range offset，生成 `*_rg_rect.off` | `dates2`、参考日期、数组范围 |
| `run_diff_array.slurm` | `4核 x 8并发` | 生成差分干涉图 `diff_*_8rlks_12alks.int` | `insarpair`、参考日期、数组范围 |
| `run_look_array.slurm` | `8核 x 8并发` | 多视并生成相干性/幅度 | `insarpair`、数组范围 |
| `run_look_geom.slurm` | `8核 x 1` | 多视几何文件，生成 `8rlks_12alks` 几何 | 参考日期、WBD |
| `run_filt_array.slurm` | `16核 x 4并发` | Goldstein 滤波，生成 `filt_*.int` | `insarpair`、数组范围 |
| `run_unwrap_array.slurm` | `8核 x 8并发` | SNAPHU 解缠，生成 `.unw` / `_msk.unw` | `insarpair`、数组范围、WBD |
| `run_geocode_array.slurm` | `16核 x 4并发` | geocode `.cor/.unw/_msk.unw` | `insarpair`、参考日期 track、DEM |
| `run_geocode_los.slurm` | `16核 x 1` | geocode 参考日期 LOS | 参考日期、DEM |
| `mintpy/run_mintpy_load.slurm` | `8核, 16G` | MintPy `load_data`，生成 `inputs/*.h5` | MintPy 配置路径 |

电离层校正脚本：

| 文件 | 当前核数/并发 | 作用 | 是否可自动 |
|---|---:|---|---|
| `run_ion_pairup.slurm` | `4核 x 1` | 建立 `pairs_ion` 目录/链接 | 可自动 |
| `run_ion_unwrap_array.slurm` | `8核 x 4并发` | ion 子带解缠 | 可自动 |
| `run_ion_filt_array.slurm` | `8核 x 4并发` | ion 滤波/拟合 | 可自动 |
| `run_ion_prep_array.slurm` | `8核 x 4并发` | 准备 `ion_cal` 输入 | 可自动 |
| `run_ion_cal_array.slurm` | `8核 x 4并发` | 计算 pair ion 校正量 | 可自动 |
| `run_ion_check.slurm` | `8核 x 1` | 生成 `fig_ion` 检查图 | 必须人工筛查 |
| `run_ion_ls.slurm` | `8核 x 1` | pair ion 最小二乘到日期 ion | 人工筛查后再跑 |
| `run_ion_correct_array.slurm` | `8核 x 4并发` | 把 ion 校正应用回 `pairs` | 可自动 |

保留但不推荐作为主线：

| 文件 | 原因 |
|---|---|
| `run_cmd1.slurm` | 直接跑完整 `cmd_1.sh`，某一步失败后后面会连锁失败。建议用拆分版。 |
| `run_ion_array.slurm` | 依赖 `sbatch run_ion_array.slurm subband/unwrap/filt/prep/cal` 传参数，容易忘。建议用分开的 ion 脚本。 |
| `run_ion_subband_one.slurm` / `run_ion_subband_retry.slurm` / `run_ion_subband_230412_231206.slurm` | 本次补救失败 pair 用，换区域不作为主线。 |
| `run_quicklook.slurm` | Python/PIL 版本，之前因 `PIL` 缺失失败。 |
| `run_quicklook_gdal.slurm` | 可选画图脚本，可保留。 |
| `mintpy/run_mintpy_all.slurm` | 能产出主要结果，但最后 plot 阶段可能让 Slurm 状态显示 FAILED。建议拆 MintPy 后处理步骤。 |
| `run_smd1.slurm` | 你贴出的内容为空，归档即可，不进入流程。 |

<a id="step-18-2"></a>

### 18.2 换区域准确必改清单

换区域主要改这些固定项：

```text
1. 处理目录：/work/home/panada/insar/proc/alos2_stack
2. 原始数据目录：/work/home/panada/insar/data/alos2
3. DEM：/work/home/panada/insar/data/dem/dem_aw3d30.dem.wgs84
4. water mask：/work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd
5. 参考日期：221221
6. 全部日期：220316 220413 221221 230315 230412 231206
7. 非参考日期：220316 220413 230315 230412 231206
8. 全部 pair：14 个 pair
9. Slurm array：pair 脚本 0 到 pair数-1；日期脚本 0 到 日期数-1
10. reference date 文件名：221221.track.xml、221221_2rlks_3alks.*、221221_8rlks_12alks.*
11. `run_radar_dem_offset.slurm` 的 `-amp`：换区域时选一个和参考日期相关、质量较好的 pair
12. MintPy 配置：`mintpy/alos2_noto.txt` 里的 metaFile/baselineDir/unwFile/corFile/geomFile/reference.date
```

`alosStack.xml` 当前关键配置：

```xml
<property name="data directory">/work/home/panada/insar/data/alos2</property>
<property name="dem for coregistration">/work/home/panada/insar/data/dem/dem_aw3d30.dem.wgs84</property>
<property name="dem for geocoding">/work/home/panada/insar/data/dem/dem_aw3d30.dem.wgs84</property>
<property name="water body">/work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd</property>
<property name="reference date of the stack">221221</property>
<property name="use water body to dertermine number of matching offsets">None</property>
```

最后一行保持 `None` 是对的：它避免在 offset 匹配数量估计时使用不兼容 water mask。注意这不等于完全不用 water mask；后面的 `rdr2geo/look_geom/unwrap/geocode` 仍然可以用 WBD 做掩膜。

<a id="step-18-3"></a>

### 18.3 一键机械替换脚本

建议先复制旧工程，再在新工程里运行替换脚本。不要直接在旧成功目录上替换。

```bash
cd /work/home/panada/insar/proc
cp -a alos2_stack alos2_stack_NEW
cd alos2_stack_NEW
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
NEW_DEM="/work/home/panada/insar/data/dem/NEW.dem.wgs84"

OLD_WBD="/work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd"
NEW_WBD="/work/home/panada/insar/data/dem/wbd_1_arcsec/NEW_watermask.wbd"

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

这个脚本只能做机械替换。`run_radar_dem_offset.slurm` 的 `-amp` 仍建议人工确认，因为它应该选一个和参考日期相关、质量较好的 pair。

<a id="step-18-4"></a>

### 18.4 自动提交链：到几何完成

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW
nano submit_01_geometry.sh
```

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

检查：

```bash
find dates -name cull.off | sort | wc -l
find dates_resampled -path "*/insar/*_rg.off" -type f | sort | wc -l
find dates_resampled -path "*/insar/*_az.off" -type f | sort | wc -l
find dates_resampled/CHANGE_REF_YYMMDD/insar -name "*lat*" -o -name "*lon*" -o -name "*hgt*" -o -name "*los*"
```

<a id="step-18-5"></a>

### 18.5 自动提交链：干涉图到多视

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW
nano submit_02_pairs.sh
```

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

检查：

```bash
find pairs -path "*/insar/*_8rlks_12alks.cor" -type f | wc -l
find pairs -path "*/insar/diff_*_8rlks_12alks.int" -type f | wc -l
find dates_resampled/CHANGE_REF_YYMMDD/insar -name "*8rlks_12alks*" | sort | head
cat dates_resampled/CHANGE_REF_YYMMDD/insar/affine_transform.txt
```

<a id="step-18-6"></a>

### 18.6 自动提交链：ion 到人工筛查点

这段会自动跑到 `fig_ion`，然后必须人工看图。

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW
nano submit_03_ion_until_check.sh
```

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
```

人工筛查：

```bash
find fig_ion -type f | sort
```

如果某些 pair 不好，不是删除结果，而是在 `run_ion_ls.slurm` 的 pair 列表里排除它们，再继续。

<a id="step-18-7"></a>

### 18.7 ion 筛查后继续到应用校正

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW
nano submit_04_ion_apply.sh
```

```bash
#!/bin/bash
set -euo pipefail
cd /work/home/panada/insar/proc/alos2_stack_NEW

jid_ls=$(sbatch --parsable run_ion_ls.slurm)
jid_correct=$(sbatch --parsable --dependency=afterok:$jid_ls run_ion_correct_array.slurm)

echo "ion_ls      $jid_ls"
echo "ion_correct $jid_correct"
```

检查：

```bash
find dates_ion -type f | sort | head
find pairs -path "*/insar/diff_*_8rlks_12alks_ori.int" -type f | wc -l
```

<a id="step-18-8"></a>

### 18.8 自动提交链：滤波、解缠、地理编码、MintPy

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW
nano submit_05_final_mintpy.sh
```

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
echo "Note: mintpy_all may show FAILED at final plot stage even if h5 products are created."
```

最终检查：

```bash
find pairs -path "*/insar/filt_*_8rlks_12alks.int" -type f | wc -l
find pairs -path "*/insar/filt_*_8rlks_12alks.unw" -type f | wc -l
find pairs -path "*/insar/filt_*_8rlks_12alks_msk.unw" -type f | wc -l
find pairs -path "*/insar/*_8rlks_12alks*.geo*" -type f | sort | head
ls -lh mintpy/*.h5 mintpy/geo/*.h5 2>/dev/null
```

<a id="step-18-9"></a>

### 18.9 怎么检查这份文档和脚本是否一致

每次换区域后，在新工程目录跑：

```bash
cd /work/home/panada/insar/proc/alos2_stack_NEW

echo "=== old strings should be gone ==="
grep -R "/work/home/panada/insar/proc/alos2_stack\|220316\|220413\|221221\|230315\|230412\|231206" -n -- *.slurm mintpy/*.slurm alosStack.xml mintpy/*.txt 2>/dev/null

echo "=== syntax check ==="
bash -n *.slurm
bash -n mintpy/*.slurm

echo "=== key paths ==="
grep -R "NEW.dem.wgs84\|NEW_watermask.wbd\|CHANGE_REF" -n -- *.slurm mintpy/*.slurm alosStack.xml mintpy/*.txt 2>/dev/null
```

第一条 `grep` 如果还能看到旧日期，不一定全错，因为你可能新区域日期刚好相同；但如果旧处理目录、旧 DEM、旧 WBD 还出现，就需要改。

<a id="step-18-10"></a>

### 18.10 water mask 本次到底怎么处理

本次不是用 ISCE2 官方 `wbd.py` 下载的传统 `swbdLat_*.wbd`，因为 NASA/USGS 路径和服务器环境一直不顺。后来改用 ASF water mask，裁剪/转换成：

```bash
/work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd
```

ISCE2 真正需要的是：

```text
asf_watermask_clip.wbd
asf_watermask_clip.wbd.xml
asf_watermask_clip.wbd.vrt
```

重点不是文件名必须叫 `swbdLat`，而是 XML 里 `FILE_NAME` 必须指向真实绝对路径，并且尺寸/坐标能被 ISCE2/GDAL 读到。

检查：

```bash
ls -lh /work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd*
grep -A2 -n "FILE_NAME" /work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd.xml
gdalinfo /work/home/panada/insar/data/dem/wbd_1_arcsec/asf_watermask_clip.wbd.vrt | head -50
```

换区域时，water mask 有两种选择：

```text
方案 A：继续从 ASF 下载覆盖新区域的 water mask，裁剪成 ISCE2 可读的 .wbd/.vrt/.xml。
方案 B：如果没有水域或暂时不想处理，alosStack.xml 里 water body 可设 None，但 unwrap/geocode 掩膜效果会变弱。
```

当前推荐：保留 water mask 路径，但 `use water body to dertermine number of matching offsets` 保持 `None`。
