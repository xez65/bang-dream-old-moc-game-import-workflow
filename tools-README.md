# M02 census tools (kept out of `tmp/` so they survive evidence cleanup)

These are the scripts that produced `issues/02-load-probe.md`'s Resolution. Original run
location was `tmp/`; this copy is the stable one for later tickets (M03/M05/M08/M09).

## m03_core_census.py — per-parameter deformation census (native core, ctypes)

Drives every parameter of every candidate moc3 to min and max and byte-compares the
drawable vertex/uv buffers. This is the only trustworthy way to answer "is this
parameter wired to the mesh": the Unity wrapper path (`CubismModel.ForceUpdateNow`)
contaminates the baseline and reported a bogus `live=82/87` (see the ticket's
"方法学" section). Do not re-implement the census in C#.

```sh
# listFile: one absolute moc3 path per line; outDir gets report.txt
python .scratch/moc3-showcase/tools/m03_core_census.py <listFile> <outDir>
```

Env knobs:
- `FENG_CORE_DLL` — absolute path to a `Live2DCubismCore.dll` to load instead of the
  main project's (5.1.0). Point it at the worktree copy for the 6.0.1 pass.

Three native-layer traps already handled inside (keep them if you edit):
1. `csmSizeInt` is `size_t` -> `ctypes.c_size_t`, not `c_uint32`.
2. The moc buffer must be `AlignofMoc=64` aligned (model: 16), otherwise
   `csmReviveMocInPlace` returns NULL for perfectly valid files.
3. Core 6.0.1 renamed `csmGetDrawableRenderOrders` -> `csmGetRenderOrders`.

Positive controls to re-run alongside any future census: official samples report
Haru 38/42, Rice 91/96, Natori 82/96 live parameters; every `CONTROL` diff must be 0.

## m03_summarize.py

```sh
python .scratch/moc3-showcase/tools/m03_summarize.py <reportA.txt> <reportB.txt> [labelA] [labelB]
```

Parses two census `report.txt` files (one per core build) into a per-model table:
load ok / `live_verts` / `live_any` / dead count, plus per-group `live/total`
(head, eye, mouth, brow, phys, hair, arm, body, other) and a control-dirtiness check.
Labels default to `core5.1` / `core6.0.1`.

## m03_exp_param_intersection.py

```sh
python .scratch/moc3-showcase/tools/m03_exp_param_intersection.py <reportA.txt> <reportB.txt>
```

Reads every `*.exp3.json` under `Assets/Live2D/Models/FengChuanXiang`, collects the
driven `Parameters` / `PartOpacity` ids, and intersects them with the live parameter
sets recorded in those census reports, to answer "how many expressions actually change
geometry".

## Relocation check (2026-09-19)

Both root-path lookups were rewritten after the move out of `tmp/`, and the census was
re-run from this directory on 2 candidates: `tmp/moc3-tools-smoke-20260919-0020/`.
It reproduced the M02 numbers exactly — `89c07a6b` loaded on the 5.1.0 core
(`params=87 drawables=209 parts=16 declaredTextures=2`, `live_verts=50 live_any=52
dead=35`, groups `head 6/6, eye 12/13, mouth 5/6, brow 8/8, phys 3/14`) and the v6 gate
asset still failed with `above_core=1 consistency=0 reason=revive_null`. The intersection
script likewise reprinted `198/218` vs `0/218`.

## M12-D3-E2 scripts (2026-09-20) — 空 draw-order-group 表的磁盘补丁

结论与四条 checker 律写在 `evidence/m12-e2-groupfix-20260920/README.md`，这里只登记脚本分工。

- **`m12_e2_fix12.py` = 交付脚本**（唯一需要重跑的）。读 `89c07a6b`，按 R1 律填 types/indices/group_indices 三段 + meta（**MAX 在前、MIN 在后**），搬空槽、补尾部 768 B，写 `tmp/m12-e2-20260920/FengChuanXiang-baked-groupfix.moc3`，然后自己做 consistency / revive / renderOrder 单调性 / 顶点 sha1 / 501 项压力断言。字节确定性已核（SHA256 `ea9ba1ba…869d`）。
- `m12_e2_probe2.py` = CRASH-vs-REJECT 探针台（18 组；每组一个子进程，`native()` 会把 access violation 吞成 OSError，所以必须把 `call_ok` 和 `cons` 分开记）。
- `m12_e2_fit.py` = 布局反推（在两份能过检的件上 100/100 验证 dense 布局模型），产出 `tmp/.../fit.txt`。
- `m12_e2_rule_probe.py` / `m12_e2_fix11.py` = 逆 R1 派生律（三份文件 ×三种 indices 顺序）。
- `m12_e2_fix8.py` = 因子阶梯 bisect；`m12_e2_fix10.py` = (inplace|eof) × (sentinel|rank|draw|maxmin|zero) 网格，决定性格是 `maxmin`。
- `m12_e2_patch.py`(fix1) / `fix2` / `fix3` / `fix6` / `fix7` / `fix9` = **被证伪的假设，只留作记录，不要照着改**（fix2/3/6/7 的"被拒"结论被下面的插入坑污染）。

编辑这一族脚本时的两条硬规矩：
1. 任何写向文件尾的切片之前，先按最终长度 `data += b"\x00" * (need - len(data))` 扩容 —— `bytearray[a:a+n] = …` 越过 EOF 是**插入**，会把整段尾平移、静默毁掉对照实验。
2. 尾部塌成同一偏移的空槽（slots 89..101）只能**整块搬**（SOT 必须随槽号非递减）；slot 101 在 baked 里占 768 B 活数据，不可截断。

## M12-D1 / E1 脚本（2026-09-20）— 磁盘几何诊断与 Unity 对照

结论写在 `evidence/m12-diagnosis-20260920/README.md` 与 `evidence/m12-e1-unity-20260920/README.md`。

- `m12_geom_dump.py <outDir>` = A/B/C 三路几何对拍（Cubism 2 纯 python core vs 5.1.0 native vs Umamo 文本转储），带 Haru/Natori/Rice 正对照。
- `m12_raster.py` / `m12_raster2.py <moc3> <tex0.png> [tex1.png …] <outPrefix> [--flipv]` = 纯 python 仿射光栅（v2 是真逐像素贴图采样，作数的是 v2）。**`--flipv` 的语义必须记住**：`v2 = 1-v if flipv else v; row = v2*(H-1)` ⇒ 默认 = "core 的 v 从 PNG 顶数"，**`--flipv` 才是 Unity/GL 的底原点约定 = Unity 实际看到的画面**。E1/E3 的所有"像 Unity 吗"对照一律跑 `--flipv`。
- `m12_e1_compare.py <stampDir>` = 吃 `Assets/Editor/FengMoc3E1Probe.cs` 的 dump，与 native 同件同姿态逐项对拍，用来分辨"参数默认值没落到 native / part 透明度 / SDK 更新链搬顶点"三条嫌疑（实测三条全清白）。模型空间比较即可，不需要拟合缩放偏移。
- `m12_e1_holes.py` = 把 native 光栅放进 **Unity 相机视框**（723.744 px/模型单位）做逐像素归属，算真空洞。⚠️ 必须用相机视框，用 drawable bbox 会把"超出视框"误报成"没画"（早期 0.6775 那个假数字就是这么来的）。
- `m12_e1_winding.py` = 面积正负号普查 + 三种绕序重光栅（实测 CW 只占 7.2% 面积、209 件全 DoubleSided ⇒ 剔除不是主因）。
- `m12_e1_uvcontrol.py` = **官方件对照组**（Haru/Mao/nn/llny vs 自家转换件）用不透明度命中率判 V 约定。指标本身判别力弱（两种读法都 0.3–0.55，逐 drawable 计数与总量互相矛盾），**只当旁证，权威判据是光栅出图 + 官方件反向**。

## M12-E3 脚本（2026-09-20）— UV 表 V 镜像：机制证明 + 交付补丁

结论与产物哈希写在 `evidence/m12-e3-uvfix-20260920/README.md`。

- **`m12_e3_uvflip.py` = 交付脚本**（唯一需要重跑的）。读 `Assets/Live2D/Models/FengChuanXiang/FengChuanXiang-groupfix.moc3`，对 UV 表 `[1210624, 1247200)` 做 `y → 1-y` 原位重写（零尺寸变化、无重定位），出 `tmp/m12-e3-uvflip-20260920/FengChuanXiang-uvfix.moc3`，并自带全套断言：consistency=1、顶点 sha1 不变、drawOrder/textureIndex/renderOrder 逐个不变、**改动字节 100% 落在 UV 表内**、三项数值偏差（file vs 1-file / core vs patched file / core vs 1-file）全 0。
- `m12_e3_coreflip.py` = 机制证明：`csmGetDrawableVertexUvs` 交回 `1 - v_file`，在 Haru/Mao/nn/groupfix/uvfix 五份上偏差为 0，出 `core_v_flip_census.json`。**这条律是所有后续普查/补丁的前提**：文件存 top-origin，Unity 采到的是 core 镜像后的值。

E3 这一族脚本里的两条必踩坑（已在码内处理，编辑时别退回）：
1. Count Info 的 **`counts[15]`（UVS）数的是 float 分量、不是点数**（9144 = 2×4572），断言要写 `declared == span_points * 2`。
2. `init_model()` 的**三个返回值必须全部持有**（`model, moc_keep, model_keep = init_model(...)`）。指针换算文件偏移用 `p - moc_keep.ptr`；只留 `model` 会让 buffer 被 GC，getter 静默返回 NULL，然后在线段上炸 `NULL pointer access`。

## M05-甲 脚本（2026-09-20）— 嘴部"多部件同时出现"的三层判据

结论与全部数字写在 `evidence/m12-m05a-mouth-mask-20260920/README.md`。三个工具对应三层证据，逐层排除：数据（画了哪些部件）→ 层序（谁压着谁）→ 像素（遮罩生没生效）。

- `m12_mouth_census.py [moc3]` = 嘴相关 drawable 的原始普查（id / 贴图页 / 顶点数 / 默认姿势与各嘴参数极值下的透明度、moc3 有没有 part 表）。输出前缀走环境变量 `OUT_PREFIX`（默认 `tmp/m12-mouth-census`）。
- `m12_mouth_compare.py [moc3] [--out PREFIX]` = **数据层**：默认件 `Assets/.../FengChuanXiang-uvfix.moc3`；moc2（`Work/凤川祥/live2d/model.moc`，纯 python Cubism 2 reader）↔ moc3 逐 drawable 配对（本票实测 **209/209**），再按 **7 个状态**（默认 + 6 组嘴参数极值）各拍一次快照，比对"该不该画"的集合差（实测 7 个状态的应画数逐个相同、默认姿势 disagree **0**）。写 `<PREFIX>.json`（默认 `tmp/m12-mouth-20260920/mouth_compare`）。
- `m12_mouth_order.py` = **层序层**：无参数、固定写 `tmp/m12-mouth-20260920/mouth_order.json`。比 4 个嘴组（`B_MOUTH.03..06`）内部的画法序列（moc2 侧 `interpolatedDrawOrder`，moc3 侧 `csmGetDrawableRenderOrders` + 裸 drawOrder），并给全局两两反序计数（实测 12,880 对 **0** 反序；moc2 有 2,686 对 drawOrder 并列，按 ArtMesh 索引破平）。
- `m12_mouth_render.py <moc3> <unityPng> <tex0> <tex1> <outPrefix> [--sort global|page] [--web frame.png]` = **像素层，也是这族里唯一真正测遮罩的工具**（`m12_raster2.py` / `m12_e1_holes.py` 一律忽略 `csmGetDrawableMasks`，所以"遮罩有没有生效"以前从没被对账过）。把同一件按 **4 种遮罩假设**各光栅一遍并在嘴部框内与 Unity 抓图对账：
  - `nomask` = 遮罩乘数 1；`masked` = 按 moc3 遮罩表正确裁切（mask 源自己光栅成覆盖图，取 max，`InvertMask` 取反）；`culled` = 乘数 0（被遮部件整体不画）；`half` = 乘数 0.5。后三者对应"遮罩属性块没填上"时 `UnlitMasked.mat` 可能落成的三种状态（`CubismCG.cginc:21-33/37-53/99-102` 里退化 tile 会把裁切变成**乘一个常数**，不是剔除）。
  - `--sort global`（默认，**要的就是这个**）按 `(renderOrder, drawOrder, index)`，即 Unity `BackToFrontZ + z=-renderOrder*1e-5` 的实际绘画序；`--sort page` 先按贴图页 = `m12_raster2.py` / `m12_e1_holes.py` 的老键，留作反对照。
  - `--web` 会把嘴部框映射进 Cubism 2 烘帧像素空间（`x3=(x2-1000)/2000`、`y3=(1250-y2)/2000` × 烘焙 `s/tx/ty`）出一栏并算 MAD。
  - **配准**：先搜 `dx,dy ∈ [-4,4]` 使全图 MAD 最小，再按该平移打分（本件实测偏移 **(1,−1)**、全图 MAD 8.28→5.43）。⚠️ 不配准时 2 px 的视框差会被读成"嘴部内容不同"。
  - **归属（本票内修过的坑）**：`ident` 必须按**最终贡献 alpha**（贴图 alpha × 透明度/遮罩乘数）记账。早先按贴图 alpha 记 ⇒ 被裁掉的像素仍算该 mesh 的"占有像素"，`own_masked` 与 `own_nomask` 假性相等、四通道表全废。改后 PNG 逐字节不变，只有 JSON 归属表变。

律 **R3**（本票用像素钉死，后续所有离线对照必须照做）：Unity 的有效叠序 = **全局 renderOrder 名次**，不是"贴图页优先"。嘴部框里两种键的差距是 **32.33 vs 94.06** MAD、对 web 是 **6.28 vs 88.79**；`page` 键下嘴部最上层像素直接归 0。

## M05-乙2 / M05 主票脚本（2026-09-21）— 带姿势的遮罩抓取与 87 参数逐条对账

结论与全部数字写在 `evidence/m12-m05b2-posed-mask-20260921/README.md`（九节）与 `evidence/m12-m05c-param-sweep-20260921/README.md`（十节）。**这一族是"量产链"：公布出来的每个数字都有已入库的生成器，重跑必得同一张表。**

调用顺序（M05 主票，一轮 Unity + 纯磁盘打分）：

1. `m05c_unity_csharp_check.sh` = **不起 Unity 的 C# 编译预检**（Unity 自带 Roslyn：`Data/DotNetSdk/dotnet.exe` + `sdk/8.0.318/Roslyn/bincore/csc.dll`，`-nostdlib+ -define:UNITY_EDITOR`，引用 netstandard 2.1 + UnityEditor/UnityEngine 各 Module + `Library/ScriptAssemblies/Live2D.Cubism.dll` + Newtonsoft，镜像 Unity 给 `Assembly-CSharp-Editor` 的引用集）。改完 `Assets/Editor/FengMoc3MaskProbe.cs` 先跑它，0 错误才起 batchmode —— 省一轮 30 秒 Unity 也避免"跑一半编译失败"。**注意它只保证编得过，不保证跑得对。**
2. `python m05_param_coupling.py <moc3> <tex0.png> <tex1.png> <outDir>` = **Unity 之前的离线逐参数普查**：原生 core 记 `d_verts`/`max_vert_delta`，同一参数值再交离线光栅（`--sort global`）算"应画面积增量 / 遮罩相关像素 / 三档可见性变化"，据此把 87 个参数分成 strong/weak/zero ⇒ 产出 `sweep_manifest.json`（探针的输入，探针会把它的 md5 登记进报告）。**存在的理由**：M05-乙2 实测"core 动了 28 张网格"与"画面动了 106 px"是两件事，选端点必须以两份底片为准。
3. 探针：`powershell -File tmp/run-feng-frame-verify.ps1 -Method FengMoc3MaskProbe.RunM05C -LogFile <log>`（**禁 `-nographics`**；起前查 commit 余量）。S 段 = 编辑器内手动泵，**不进 PlayMode**；每参数一张 1024² + `mask_probe_phaseS.json`。
4. `python m05_sweep_eval.py <probeOutDir> <outDir> [--raster mouth,eye,brow] [--web png]` = 把每张抓图交回 `m12_mouth_render.py`（`m12_mouth_render.py:101` 只做 source-over 合成，件内实测无非 normal 混合 ⇒ 与真值一致）出**同参数值**的四假设底片 + 嘴框四假设对照（遮罩活性闸）。
5. `python m05_region_score.py <evalDir> <probeDir> [--margin 8] [--min-px 30]` = 在**该参数自己移动的那个框**内给四假设打分（框从探针 `changed_vs_neutral.bbox` 取，**必须翻 y**：`y_png = 1023 - y_probe`，`GetPixels32` 的 0 号是左下）；结尾有 `eyes_above_mouth` 顺序断言，翻错 y 会当场失败。⚠️ 该 JSON 必须由**当前版本的脚本**产出，否则等于发布旧结论（本轮就抓到一次：证据目录里那份是脚本最后一次编辑之前写的，sanity 键还是旧的 `eye_rows/mouth_rows`、`eyes_above_mouth:false`）。
6. `python m05_registration_floor.py <evalDir> <probeDir> <regionJson> [--radius 3]` = **方法学地板**：把框拆成"该参数没动过的像素"，那些像素上的残余 MAD 与遮罩对错无关。实测 eye 中位 53.2 / brow 59.6 / mouth 12.6，±3 px 位移搜索后 25.5 / 35.6 / 4.3，24/24 行偏好 `dy∈{-1,-2}, dx∈{0,+1}` ⇒ **眼/眉从此只做 masked vs culled 的相对判据**，绝对判定只在嘴框可用。
7. `python m05_sweep_ledger.py <evalDir> <probeDir> <outDir>` = 台账 `sweep_ledger.md` + `.csv`（87 行，三处 JSON 的重排，不产生新数字）。**今后台账只由它生成**（此前 csv 把"区内 masked MAD"串进了"嘴框 MAD"列）。
8. `python m05_expression_census.py <probeReport> <outJson>` / `python m05_motion_census.py <probeReport> <outJson>` = **不靠 Unity 把"表情 / 动作切换"两项答成可核事实**：把 218 份 `exp3.json`、397 份 `motion3.json` 的参数名与扫描的逐参数判定联接（实测 3 个表情参数死 ⇒ 40/218 缺一笔；动作引用 85 个 id 里 35 个空转 ⇒ 0 段整段死）。M09/M10 的分子集合由此定，不必盲抓。

另有 `m05b2_pose_eval.py <probeOutDir> <outDir> [--shots pat] [--web png] [--only feng]` = M05-乙2 那张表的驱动（`python` 侧统一入口，内部仍调 `m12_mouth_render.py --set ID=value`）。

**两条脚本级坑（改这一族前必读）**：
1. `m12_mouth_render.py` 自本轮起支持 `--set ID=value`（夹到参数自身 min/max，未知 id 打 `POSE-MISS`），**位置参数解析改为显式跳值** —— 旧写法下 `--sort global` 的值会挤掉位置参数。
2. 1024² 截图 ~4 MB 一张，探针侧必须每张 `Pixels.Remove(fileName)`，否则 90+ 张会把 commit 吃穿。


## M04 门槛 A 件（2026-09-21）— 存盘场景 + 1280×720 证据图

结论与全部数字写在 `evidence/m04-scene-showcase-20260921/README.md`（八节）。本票**没有新增 `tools/` 脚本**，但留下两条跨票约束和一个可复算小工具：

- 探针 `Assets/Editor/FengMoc3MaskProbe.cs` 新增 `Mode.M04` / `RunM04()`：同一进程内 `SaveScene` → `OpenScene` 重开 → `AdoptSubjectFromScene`（从 `GetActiveScene().GetRootGameObjects()` 认领，断言恰好 1 个 `CubismModel` + 1 个 `Camera`）→ 真 PlayMode 抓图。**"存盘的 .unity 能不能被重新认领并渲染"由此变成实测事实，不再靠 `[ExecuteInEditMode]` 的源码推断。**
- **尺寸纪律（要紧）**：`RenderShot` 被拆成 `RenderShotInternal(..., bool pinAspect)`。1024² 一律走**不改 `Cam.aspect`** 的 `RenderShot`（历史帧与 M05-乙2/M05 主票字节可比的唯一前提）；非方形（1280×720）走新的 `RenderShotSized`，它显式钉 `Cam.aspect = w/h`、渲完 `targetTexture=null` 并还原 aspect。**给 1024² 路径加 aspect 会当场作废全部跨票对拍。**
- **几何基线顺序**：每条腿必须在**该腿自己那张已停稳的 neutral** 上 `CaptureGeometryBaseline` 之后才算 `GeometryDelta`（M05-乙2 第 9 条那个基线污染的同类坑；M04 里中性孪生帧的基线取在 angle_z20 孪生之前，实测 `moved=0 / max_delta=0` 自洽）。
- `evidence/m04-scene-showcase-20260921/check_frames.py`（纯 numpy+PIL，按自身目录找图，故与证据同目录、不搬进 `tools/`）= **对识图结论做像素否证/证实的范式**：比两图的非背景外接框与逐行宽度 ⇒ 判"某部分有没有出画"；给变化像素的 change box ⇒ 判"动的是哪个身体带"。跑法：`python .scratch/moc3-showcase/evidence/m04-scene-showcase-20260921/check_frames.py`。
- 计数口径提醒：探针内置差分与 `check_frames.py` 的 `sum|ΔRGB|>12` 对同一对图给出 28,351 / 32,408 两个数 ⇒ **公布 changed px 必须同时给阈值**（沿用 M05 主票"每个数字都有生成器"的纪律）。


## M08 物理翻译与实测脚本（2026-09-21）

结论与数字在 `evidence/m08-physics-translate-20260921/README.md`（磁盘侧）与 `evidence/m08b-physics-unity-20260921/README.md`（Unity 侧）。

1. `python m08_translate_physics2to3.py` = Cubism 2 `physics.json` → `.physics3.json` 的**确定性翻译器**。凡"不是逐字段照搬"的都必须在 `derivation.json` 里推导出来（最关键一条：v2 的 src 权重是**反序逐次 lerp**、v4 是**归一化份额求和**，两种权重不是同一种数，故解 v2 递推的稳态来定 v4 权重）。重跑必得同一份字节（冻结 md5 `b88b82d1adfc440186d99f69293a9b0c`）。
2. `python m08_check_physics3.py` = 静态校验器，只回答"这份件**能不能**被本仓 SDK checkout 装载、它碰到的每个 id 是否都在门禁 moc3 上"（C1 键/枚举对 `CubismPhysics3Json.cs`、C2 Meta 计数自洽、C3 id 存在、C4 输入参数量程对称）。**它从不宣称"模型会摆"** —— 那是 乙 的 Unity 票。
3. `node m08b_v2_swing_reference.mjs` = **v2 参照曲线**：跑**官方** Cubism 2.1 `PhysicsHair` 本体 + 真实 `model.moc` 的参数量程（不在 Node 里重写公式，因为 src 归一化要用 core 内部的 min/max）。协议 `1/30 s` 一步、"阶跃 + 释放"场景 ⇒ `v2_reference_curves.json`（fps 30 / hold 30 / tail 90；md5 `830dda24…`）。⚠️ 每步口径与 Unity 侧**共用**：输入恒定、物理在上一步基础上继续，**不**逐帧 `reset()`（那是烘帧线的淡入需要）。
4. `node m08b_fit.mjs` = 两侧共用的**唯一**拟合器（从 3 里拆出来）。拆的理由写在其头部：两份拟合算术会在两个待比数字之间插入实现差异，届时"频率不同"就无法归因给运行时。拆分为**已核对的 no-op**（重跑产物与已入库那份字节相同）。
5. `node m08b_compare_v2_v4.mjs [--unity … --v2 …]` = 打分三问（有没有响应 / 落到网格没有 / 符号与频率），**容差冻结在它的 `cuts` 里**：`amplitude_ratio_below 0.8`、`decay_sigma_relative_error_above 0.2`、`pointwise_abs_r_below 0.5`；逐点 r 用 lag 0 与 lag +1、取 \|r\| 大者（lag 1 是 `CubismPhysicsRig.Evaluate` 在 `dt = 1/Fps` 下的性质，不是缺陷）。**这三条也是 M10 裁决 A 打分件沿用的容差来源。**
6. `python m08b_where_census.py <probeDir>` / `m08b_sheet_detail.py` / `m08b_sheet_montage.py` = 找"变化最强在哪"再把摆动帧拼成一张标注拼图交识图（每格除三个物理输出外全为默认值，所以格间差只能来自被翻译的 rig）。


## M09 表情脚本（2026-09-21）

结论与数字在 `evidence/m09a-expression-predict-20260921/`（甲 预测）与 `evidence/m09b-expression-unity-20260921/README.md`（乙/丙）。

1. `python m09_expression_predict.py` = **零 Unity 的 218 张预测器**：从**已导入的 `.exp3.asset`**（不是 json）推出官方 `CubismExpressionController` 单张套用一次必然落到的参数值，再问 M05-C 的逐参数普查这些值对网格意味着什么 ⇒ 每条结论都可被 乙 否证。**没有可否证对象的测量只是截图堆积。**
2. `python m09b_manifest_check.py [tmp/m09_shot_manifest.json]` = C# harness 会读的每条 JSON 路径的磁盘预检（run 2 死在第一张图的装箱转换 bug 上，之后整条腿没跑过 ⇒ 缺键/错键在这里 0 Unity 抓出来）。
3. `python m09b_reconcile.py <probeDir> <jiaDir> <outDir>` = 丙 的 **218 行甲/乙对账台账**（md + csv + json）。**里面没有任何一个新测的数字**，全部来自 `mask_probe_phaseM.json`（Unity 做了啥）与甲 冻结预测件（甲 冻结了啥）⇒ 同输入重跑必逐字节复现同一张表。
4. `python m09b_keyframes.py` = 按**规则**挑出真正承载结论的帧、对探针报告里的 md5 复核字节、拷进 `frames_key/` 入 git（确定性：同样输入同一套帧，不看图后手挑）。
5. `python m09b_sheet_montage.py` = 两张标注拼图（A = `changed_px` 最大五张的强端；B = 非零最小五张 + 一张零档对照的弱端），每行四格同框裁切：neutral / expression / ×6 有符号差值 / 1:1。**差值图 + 原图必须同框**，否则会把"弱但真实"读成"坏"（判据规则已写进 `issues/05`）。


## M10 动作/物理接线脚本（2026-09-22 → 09-23）

结论与闸门表在 `evidence/m10a-wiring-migrate-20260922-run3/`（甲 A1–A5）、`evidence/m10b-unity-run3b-20260922/README.md`（乙 14 闸 + 裁决 A 打分节）、判据措辞在 `issues/10-motions.md`。

1. `python m10_migrate_model3.py <outDir>` = 甲 的**迁移器**（零 Unity）：门槛件 `FengChuanXiang-uvfix.model3.json` 只出 `Moc/Textures/DisplayInfo/FileFormats`，内容件 `FengChuanXiang.model3.json` 只出 `Motions`(397)/`Expressions`(218)，拼成**同目录新文件** `-wired.model3.json` + 落 M08 冻结物理件。键的形状照 `CubismModel3Json.cs` 的解析器写（`Motions` 必须是 dict、`Expressions` 是 list of `{Name,File}`、`Physics` 是单个相对路径字符串）。**A1 与 A4b 的分工**：`Physics` 是本工具自己要创建的文件，落盘前要求它存在是顺序矛盾（run 1 因此永远 FAIL）⇒ A1 只对 619 条既有引用查存在，新建那条连同"磁盘字节 == 内存字节"归 A4b。A3 = 门槛件三件套 + `.controller` + mask 资产 + 存盘场景共 7 文件写前后 md5 逐个全等（**这就是"另开一份 wired、不就地改 uvfix"的实测护栏**：就地改会让下次 reimport 无条件重写 `-uvfix.prefab` 并 `CopySerialized` 覆盖场景引用的 `CubismMoc`，而 M04/M05-C/M09-乙 的字节锚点正是 `Instantiate(-uvfix.prefab)`）。
2. `python m10_publish_unity_inputs.py <甲证据目录> <out.json> [--commit-free GB]` = 把 乙 的**全部靶子**冻结成固定路径 `tmp/m10_shot_manifest.json`（探针不许猜带日期的证据目录）：`protected_md5`(7) / `fade_surface_backup`(75，字节备份在 `tmp/moc3-m10a-fadebackup-20260922/`) / 15 段 picks（带**完整** `live_params` 与取样窗）/ `switch_pair` / 12 张表情 + 名字→`CurrentExpressionIndex` 映射 / `physics_v2_signs`。
3. `python m10_fade_name_collision_census.py [outDir]` = **SDK 匹配规则的磁盘复现**：`CubismFadeMotionImporter.cs:176` 用 `Path.GetFileName` 把匹配键截成基名 ⇒ 397 段（11 个角色目录、基名高度重复，只有 71 个不同基名）走的是 reuse 分支，而 reuse 分支不重新分配数组（`CubismFadeMotionData.cs:115-164`）→ 后一条曲线数超过先入者就 `IndexOutOfRange` **中断导入**。按 SDK 顺序逐步模拟，量化 `halt_index_out_of_range`(128) / `silent_overwrite`(198)。**存在的理由：这条崩溃不启动 Unity 就能完整复现，而 Unity 侧只看到一条不含业务信息的栈。**
4. `python m10_anim_event_preflight.py` = 397 个 `.anim` 的 `InstanceId` 事件普查（M10-⑤ 的来源）：272 条有且 id 两两不同、**125 条没有** ⇒ 那 125 条按构造永远绑不上 fade（参数照样会动，坏的是混合/切换）。跑法纯磁盘，**用来在 20–25 分钟的 wired 导入之前保住那一轮**。
5. `python m10_add_fade_bound_pair.py [--candidates|--check]` = G4b 确认腿的**加法**改 manifest（每个 pick 加 `clip_fade_event_id`、顶层加 `switch_pair_b`，其余字段与已发布件逐项相等；`--check` 复验"与已发布件相同 = True"）。选对的两个硬约束是"两侧都可绑"+"两侧曲线真的不同"，第二条来自 G4 的闸形 `shared.Count > 0`。计数用其 `differing_params()` —— 探针 `M10DifferingIds()` 的**磁盘照搬**（type-flag 分段 + Unity 三次 Hermite），**先对过真机读数**（冻结对磁盘 35 == run 2 现场 35）再拿它预算新对（`anon → mutsumi`：61 共享 / 35 差异）。⚠️ 脚本头部记着一次纠错：早先按 `.motion3.json` 里**根本不存在**的 `Values`/`Times` 比对，得到过"0 条差异"的假结论 ⇒ **禁止按字段名猜曲线**。
6. **探针侧不需要新脚本**：`-Method FengMoc3MaskProbe.RunM10`（主轮）与 `RunM10Anchor`（锚点腿）；`UNITY_EXIT:1` 要分清真红项与"编译失败"（后者无证据目录 + 日志有 `error CS`）。
7. `python m10_g5_criterion_a_score.py [--regime R2_animator_off] [--print-only] [--unity … --v2 … --out-dir …]` = **丙：按用户裁决 A 给门槛 B"物理"项打分（零 Unity）**。默认输入即本仓现状：run 3b 的 `mask_probe_m10.json`（`M10_G5_diagnostic` 腿）+ M08-乙 的 `v2_reference_curves.json`（scenario `body_z_pos`，driven `PARAM_BODY_ANGLE_Z=+10`），两份输入的 md5 写进产物里。**输入来源**：默认读 `tmp/` 那次运行的 `mask_probe_m10.json`，`tmp/` 被清时自动回落到已入库的 `evidence/m10b-unity-run3b-20260922/mask_probe_m10.json` —— 两处 md5 已核对同为 `2e2735d7…` ⇒ clone 之后也能重跑复现同一张表（本次入库的 json 记录的就是已入库副本这条路径；用副本重跑时 `.md` 与逐位读数完全不变）。做三件事：① 两相对 v2 **同号**；② 峰值**幅度比 ≥0.8**；③ 逐点 **\|r\| ≥0.5**（lag 0/1 取大，与 5 同量法）⇒ **容差全部沿用 5 的 `cuts`，本脚本不新增阈值**。两侧协议同为 30 fps / hold 30，故做**同窗口对齐**：Unity 释放相只 30 帧，v2 尾部 90 步被截到同样帧数，v2 全长尾峰值一并发表当上下文；比值 >1 只登记不设上界（是否碍事交 M11）。产物 = 证据目录里的 `m10_g5_criterion_a.json` + `.md`（含 markdown 表、PASS 之外必须发表的释放相偏置、rig 是否收到驱动值、其余四个 regime 的对照），**退出码 0 = PASS**。跑完 2026-09-23：`VERDICT=PASS`，三条边界（负向场景在该 regime 无读数 / 更长衰减段未对账 / 物理输出的像素影响未抓帧）写进 `.md` 末尾。
8. `python m10_restore_fade_surface.py [--dry-run]` = **把 wired 导入弄脏的共享写面按字节还原回 甲 发布态**（`.model3.json` 被 asset refresh 看到就会跑官方导入链，写的是**按目录名派生的共享面**：`<Dir>/<DirName>.fadeMotionList.asset` 与 `<motion>.fade.asset`，连"顺手编译一下探针"那次都会写 ⇒ 不还原下一轮测的就不是"第一次 wired 导入"）。范围只到 `Assets/Live2D/**`（红线：永不提交）：75 个甲 备份过的 `.fade.asset` 逐字节覆盖回备份并复核 md5；删除可再生产物 `-wired.{asset,controller,prefab}`(+`.meta`) 与 `FengChuanXiang.fadeMotionList.asset`(+`.meta`)；**只核对不改** `expressionList` 与 manifest 里 7 个 `protected_md5`。**`-wired.model3.json` 是 甲 写的输入件，按设计保留。** 2026-09-23 丙 执行：`restored=75/75`、删 8、`expressionList b5f32979… == 甲`、受保护 7/7、`VERDICT=CLEAN`（exit 0），复跑 `--dry-run` 得 `drifted=0`（幂等）⇒ 输出存 `evidence/m10b-unity-run3b-20260922/m10c_restore_fade_surface_20260923.txt`。
   ⚠️ **2026-09-24 起停用**：见 `issues/05` 的 M10c-③ —— 强制导入已把 37 条陈旧 `MotionLength` 就地修好，而这个工具会按 甲 的字节备份把**陈旧态写回去**。红线"不许重导"的可保留边界已重述为"不为了改盘而跑导入、且不提交 `Assets/Live2D/**`"。

## M10c-丙 的表格形状闸（2026-09-24）

- `python m10c_table_integrity_check.py [--files-from MANIFEST] [FILE ...]` = **零 Unity 的 markdown 表格完整性检查**。**存在的理由**：丙 要在三处台账（仓内记录、`issues/05`、`map.md`）的超长表格行里做追加式修订，而那种行上有两种破坏肉眼看不见 —— ① 行尾那个分隔 `|` 被吃掉，② 代码跨度里写了裸竖线（`\|x\|`、`\|r\|`、`\|差\|` 这类绝对值记号，本仓约定转义）。两者都会把已发表的表格**静默改形**。
- **判据**：按"未转义竖线数"给每块表格统计宽度，出现两种以上即 `BAD` 并列出偏离行号，`TOTAL_BAD>0` 时 exit 1。**不猜哪一行是对的**（不设"多数行宽"），所以每条偏离都要人去看。本轮把 5 处存量格式债一次清掉（`issues/05` 的 M08b-①②、M10-⑦⑧ 与 `map.md` 的 M10c 行），清完 `TOTAL_BAD=0`。
- **用法受本机红线约束**（"中文一律不进命令行参数"）：被检文件走 **已入库的 UTF-8 清单** `tools/m10c_table_check_manifest.txt`，每行 `repo|相对仓库根` 或 `scratch|相对本图目录`（无前缀则按当前目录解析），空行与 `#` 注释忽略，解析不到文件即硬失败 ⇒ clone 下来一条命令即可复跑本轮检查（`tmp/` 只留输出）。
- **阳性对照**（律 6 的同一纪律）：一个故意写坏的文件必须报 `BAD` —— 闸本身坏掉时它给不出绿。本轮已实测（`cells=3/4/6` 三档宽度同时报出、exit 1）。
- 本票其余 M10c 生成器（普查、发布输入件、两个带阳性对照的编译闸、K8 归因、陈旧值来历、还原包含性核对）的逐项描述在 `凤川祥/_bake/记录_实时moc3线.md` §8。


## M11 开票：表格清单另开一份（2026-09-24，零 Unity）

- 新清单 `tools/m11_table_check_manifest.txt` = M10c 那五份 + **M11 票面 `issues/16-m11-newwebgal-comparison.md`**，用同一条命令复跑：`python m10c_table_integrity_check.py --files-from m11_table_check_manifest.txt`。M11 的 甲/乙/丙 每轮改完台账表格都跑它（本票要在 `记录` §1/§3/§9、`map.md`、`issues/05`、票面 §3 判据表里做长行追加，正是这个闸的适用形状）。
- **为什么不直接往 `m10c_table_check_manifest.txt` 里加一行**：那份清单的**条目构成已被 M10c-丙 的发表件逐条引用**（`map.md` 的 M10c 行与证据 `README.md` §10 写着"复现 修前 = 5 时从 `9669a04^` 取件、只能取出**四份** blob"）。改它会留下一条对不上当前文件的已发表说明 ⇒ 按"已发表只追加不改写"的同一纪律，旧清单原样冻结，新腿用新清单。
- **开票本轮实测**：`TOTAL_BAD=0`（exit 0）。票面 `issues/16` 的两张表被识别且各自等宽 —— §1 靶子表 9 行 × 10 列、§3 判据表 10 行 × 4 列。开票过程中 §3 的 R5 行原本在代码跨度里写了裸竖线（绝对值记号，会把该行切成 6 列），已按本仓约定转义 —— **纯格式修复，判据、阈值、读数一字未动**。
- 新工具 `tools/m11_append_only_audit.py`（零 Unity、只读）= **追加式修订的机器见证**：把每份被改文档的 HEAD 版（`git show HEAD:<路径>`）与工作树逐行对拍，报出 REPLACE / DELETED / INSERTED 三种块，并对每个 REPLACE 判定"新行是否以旧行前 200 字开头且不短于旧行"（= 追加）还是"旧文本被动了"（`!! NON-APPEND`）；**任何 HEAD 行整体消失 → `TOTAL_DELETED_HEAD_LINES>0` + exit 1**。存在的理由就是本轮自己撞到的那次事故（见票面 §10）：在 5,000 字一行的表格里用整行做 `old_string` 追加，会**静默删掉一条已发表行**，肉眼看不出来。用法：`cd .scratch/moc3-showcase && python tools/m11_append_only_audit.py`（默认文件集 = 本票要动的五份；`issues/16` 在入库前没有 HEAD blob，故不列入，丙 时再加）。报告写 `tmp/m11_append_only_audit.txt`（`tmp/**` 不进 git），控制台只印 ASCII 文件名。
- **开票本轮实测**：`TOTAL_DELETED_HEAD_LINES=0`（exit 0）。五份文档共 8 处 REPLACE + 2 处纯 INSERT 块，其中 **7 处是"旧文本原样 + 尾部追加"**；唯一一处 `!! NON-APPEND` = `map.md` 的 M11 行第 3 格由"待新建"改判为"已开票"（票的状态位，不是读数），已另用逐字包含检查证明**该行原有的 ①…⑤ 全部 1,286 字原样保留在改后的行里**，且行内明写"下面保留的是开票前已有的量化前置与移交账（原文一字未改）"。

## M11-甲 腿工具登记（2026-09-24 → 09-25，全部零 Unity）

九件新工具都在 `tools/` 下、随本票首次提交入库；各自产出的证据目录同名对应，此处只记"是什么、怎么跑、别踩的坑"。

- `m11_baseline_census.py`（甲1，只读磁盘）= §1 靶子表逐格对账：7 段的 `pngCount` / `.anim` 时长 / `instance_id` / bound 与否 / 表情 acc，产出 `evidence/m11a-baseline-census-20260924/`。它同时证明"参考侧没有动作+表情合成件"（用户 2026-09-24 裁"先不加"，G6）。
- `m11_web_floor.mjs` + `m11_web_floor_score.py`（甲2）= web 实拍 ↔ 烘帧 的 floor (i)。`.mjs` 用本机 headless Edge + `bake-lib.mjs` 的 `Cdp`/`launchEdge` 抓取（抓法不需要任何新工具，见票面 §8 断点条②），`.py` 打分。**它的 `parse_args()` 在 import 期就跑**，凡要复用它的 `register/measure/place/BakeCache/erode` 都得按 `m11_r0_controls.py` 那套"换 argv + 指一个 scratch `--out` + 永不调 `main()`"来做。 successor 链：`-wholeclip` → `-fixedsweep` → **`-fixedsweep2`（现用）**，旧目录一字未改。
- `m11_web_web_score.py`（甲2 的另一半）= 同 URL 两次复抓的 web↔web 自残差，把"纯抖动"与"两台渲染器不一致"分列；`jitter_mad` 是**同一 label 最佳同态对的 mean**（不是所有对的均值），定义在该 JSON 的 `jitter_definition`。目录 `-fixedfloor`（现用）。
- `m11_registration_framing.py`（甲3）= 烘帧 1024² ↔ Unity 1024² 的相似变换配准 + floor (ii) 逐框地板 + 剪影见证。它的 `iou_at()` 是"逐放置重渲染"的独立通道（B10 要求），与便宜的 `iou_sweep()` 互为对拍。**其 8 个姿势只呈现 1 个剪影 ⇒ 闸 B7 = `MISSING_RULER`（姿势响应没有尺）**，凡引用 floor (ii) 都必须带这条条件（甲5 的 T6 结构上强制）。
- `m11_physics_bias_ruler.py`（甲4）= R5 那把"常数偏置"新尺（偏置 ÷ 该段两侧自身散布，闸 1.0），4 种散布定义 × 5 种合并法 = 20 格；`CLOTHES_A` 当前 **DEFINITION-DEPENDENT**（10/20 跨线）—— 这是它的发表状态，不是 bug。
- `m11_r0_controls.py`（甲4）= §3 R0 三件对照跑在已发表尺上（import 而非抄）。**当前唯一红 = C1 阳性错帧**：13/336 格把参考侧错帧后残差没变差 ⇒ 任何 (a)/(b)/(c) 引用 R0 之前，乙 必须在真 Unity 帧上重跑控制 1、3。
- `m11_score_chain.py`（甲4）= 交付给 乙 的 7 步命令清单（含 R7 遮罩闸、逐段出数、md5 逐张回报），本身不跑 Unity。
- `m11_thresholds_freeze.py`（甲5，本轮）= **帧号名单 + §4 的"底 × 2"阈值代入冻结**。跑法 `python tools/m11_thresholds_freeze.py`（默认 `--out=evidence/m11a-thresholds-20260925-fixed1`；`--skip-frame-hashes` 只用于烟测，那道闸此时发 `None` 不发假红）。它只做两件事：把 §3 的规则解成每段 20 帧 / 全票 140 帧并逐张取参考侧 md5；从两份已发表底 + 抖动件里**读**出阈值乘系数 2（T0 钉死系数）。它**新测**的只有 R1 两列几何底（G3），且必须先用甲3 自己的 `place()/iou_at()` 复现已发表 bbox/像素数/IoU（T5，实测 8/8、IoU 最大差 0）才允许出数。**门槛 B 一项不判（T7）**。
  - **`-fixed1` 的来历**：同日更早那次运行 `evidence/m11a-thresholds-20260925` 把两条 R1 面积行的 `limit_value` 写成了**底**（9819 / 2297），真正的 ×2 上限藏在第二个字段 `limit_value_px` 里；本文件所有消费方（含 乙 的 `for_yis_filing` 清单）一律先读 `limit_value` ⇒ **没有闸会红**。旧目录字节未动、不覆盖、不删，新目录 JSON 的 `supersedes` 块写明错处与正确上限（19638 / 4594 px）。
  - 防重犯 = 新增闸 **T9**（`t9_defect_rows()`，故意放模块级，好让已发表文件能被重新打分）。非空转实证，逐字可跑（在 `.scratch/moc3-showcase/` 下，输出 4 条、全落在那 2 行）：

```python
python -c "import importlib.util as u,json; s=u.spec_from_file_location('m','tools/m11_thresholds_freeze.py');
m=u.module_from_spec(s); s.loader.exec_module(m); d=json.load(open('evidence/m11a-thresholds-20260925/thresholds_frozen.json',encoding='utf-8'));
[print(x) for x in m.t9_defect_rows(d['thresholds'])]"
```

  - 本轮实测（`-fixed1`）：`verdict = FROZEN`、`checks_failed = []`、T0..T9 十道全绿、阈值 39 行（35 带上限 / 4 按缺尺或沿用既有尺发表）、帧 140/140 且 140 个互异 md5、墙钟 30.1 s。细则与两处坑（两种 alpha 约定在重采样下连未配准差值符号都相反；floor (ii) 只覆盖静脸）见 `evidence/m11a-thresholds-20260925-fixed1/README-20260925.md`。

## M11-甲7 追加腿：R2-ABS 的尺子 + `-fixed2` 冻结（2026-09-25，零 Unity）

起因 = 用户 2026-09-25 裁 HITL 问题 A："A要绝对判据，角色做出改变嘴型的动作时，呈现图前后一定要有差别，M11乙放行"。**这条腿只做尺子与界限，不做任何门槛 B 判定**（尺子侧 A10、冻结侧 T7 各自见证）。

- `m11_mouth_abs_ruler.py`（新）= **R2-ABS 的尺子**。跑法 `python tools/m11_mouth_abs_ruler.py`（默认 `--out=evidence/m11a-mouth-abs-ruler-20260925-fixed2`，`--quiet` 只写文件）。产出 `verdict = FROZEN`、A1..A12 十二闸全绿、**13 条界限**，墙钟 6.1 s；2026-09-25 复跑到空 scratch 目录 ⇒ `rc=0`、与发表件除 `run_date`/`elapsed_s` 外逐字段相同。它先证三件事才发界限：**底 = 实测 0 px**（同一中性状态四条腿两进程重抓，四件 md5 全 `ae25133ba68ff48bead06dbfbd971546`，6 对逐框 0、整帧 0）；**尺子在真实 Unity 字节上会开火**（M05-C 四个单参数扫参件复算得 106 / 948 / 179 / 67 px，`delta_vs_m13 = 0`，配准按甲3 发表值复用、IoU 复现到 0 差）；**140 帧里 0 对可归因给嘴**（段 1 那三帧同时动 16–17 个非嘴参数；嘴族全平的 130 帧上同一固定框仍收 191 / 401 / 5,124 / 195 px）。于是界限只架在**归因干净**的三种装置上：L1 单参数对照（1 px）、L2 钉嘴孪生（1 px）、L3 平嘴段泄漏哨（**恰好 0 px**，非 0 是"嘴路泄漏"的发现而不是通过）。
- **别把 M13 的 2.8 当尺**：§4 第 106 行（2026-09-23"维持相对判据"，提交 `c04c4ef`）对眼/眉继续有效，22.215982 / 36.040276 / 2.808207 仍是**读数**，尺子侧 **A8** 与冻结侧 **T6** 双保险拒绝它们进界限位。R2-ABS 是"对着实测 0 px 底的像素计数"，形状不同。
- `m11_thresholds_freeze.py` 的变更（**对上一条 bullet 里"默认 `--out=…-fixed1` / 39 行 / 30.1 s"的带日期更正**）：默认 `--out` 现为 `evidence/m11a-thresholds-20260925-fixed2`，阈值 **52 行 = 原 39 行逐字段不变 + 13 行存在性界限**，输入多一件（`mouth_abs_ruler` 的 JSON，md5 `2aa3c2a9111720ecc3363230bfaefe4b`，缺件即拒绝冻结），T4 的数值池 11,758 → 12,124，本轮墙钟 3.1 s（scratch 复跑 2.9 s、除 `run_date`/`elapsed_s` 外逐字段相同）。140 帧名单**一字未改**：与 `-fixed1` 除 `run_date` 外逐字段相同，七个逐段 `list_md5_digest` 全部相等（值见 `-fixed2/README-20260925.md` §2）。
- **新闸 T10** `existence_rows_are_the_rulers_bars_and_not_a_fixed_box_count` = 本轮的防重犯：13 行的 id / 装置 / 线 / 像素数**必须逐条等于尺子 `bars.rows`**（不许重打）、尺子必须 `FROZEN` 且 `checks_failed = []`、底必须 0 px、存在性行不得带系数、界限不得等于"固定框能收来的任何计数"（漂移最大值 ∪ 参考侧固定框读数 ∪ 能力对照值），L2 对子与 L3 的 as-played 半边必须落在冻结的 140 帧内、L3 覆盖段必须恰等于尺子的平嘴段集合。来历：`-fixed1` 前身曾把地板写进 `limit_value` 且当时无闸变红 ⇒ 这一轮把"不许重打"变成机器条件。T10 实现在 `main()` 内（它要当轮的尺子对象与冻结名单），复现 = 复跑到空 scratch 目录；**`t9_defect_rows()` 仍在模块级**，可对旧文件重新打分。本轮实测：

```
t9_defect_rows(-fixed1 的 39 行)        ->  []
t9_defect_rows(首跑 20260925 的 39 行)  ->  4 条，全落在 R1 那 2 行（19638 / 4594 px 才是上限）
```

- 踩过的坑（都已在盘上留痕）：① **选择器按 `column` 匹配线名会漏** —— 线名在 `scope`（`R2-ABS/L3/seg3`）里，`column` 只有 `R2-ABS/L3 …`，T10 首轮因此把 L3 段集合比成空；② **L3 的抓取量要按 `mouth_family_flat` 过滤** —— 段 1 有 16 帧嘴族平不代表它平嘴，未过滤会得 7 段 / 14 抓 / 25 总数（正确的是 6 段 / 12 抓 / **23**）；③ **`max()` 直接吃字符串键会取错帧** —— 帧号在 JSON 里是 `"70"` 这类键，故提出模块级 `worst_flat_frame()`（转 int、并列取最早帧，让"取哪一帧"是规则而不是当轮选择）；④ 尺子自己的 `-fixed1 → -fixed2` 那条改动是 **A6 把 M-C 的字符串帧键与 M-F 的整数键相比而误红**，改的是闸的比较、没有判定变化。
- 输出编码沿用本票约定：`.txt` 写手用 `encoding="ascii", errors="backslashreplace"`（中文成 `\uXXXX` 转义），`.json` 原样 UTF-8；诊断脚本自己写 UTF-8 文件再 Read，中文绝不进命令行参数。
- 证据与台账：尺子 `evidence/m11a-mouth-abs-ruler-20260925-fixed2/README-20260925.md`（含 13 条界限全表与"交给乙的 23 抓"）；冻结 `evidence/m11a-thresholds-20260925-fixed2/README-20260925.md`；票面登记在 `issues/16` 的 §6 甲7、§8 A/B 与断点条。

## M11-丙 追加腿：零 Unity 打分器 + 遮罩四假设子腿（2026-09-25，零 Unity）

起因 = 票面 §6 丙 的收口腿（对账 README + 逐项三分判定 + 两项表态）。两支新工具都不起 Unity、不渲染、不写资产、不改任何已发表读数，只在 乙 run 5 的 163 张真机抓图 + 甲 的冻结件 + 帧库上做离线复算。

- `m11_bing_score.py`（新，主腿）= **门槛 B 五项的三分判定打分器**。跑法 `python tools/m11_bing_score.py`（默认 `--out=evidence/m11c-score-20260925-fixed3`）。`main()` 带**两条覆写守卫**：① `--limit` 冒烟轮**不许**落进 `evidence/`（冒烟只写 scratch）；② 目标目录里已有同名 `m11_bing_score.json` 即**拒绝覆写**（缺陷 M11-⑨ 第 ⑤ 条的落点：首发那一个目录当时只有守卫 ①、缺守卫 ②，且 `--out` 默认值正是它自己那个目录）。本轮跑 **31 闸 / 8 红**（`Q3a`/`Q5b`/`Q6a`/`Q7b`/`Q7d`/`Q7f`/`Q9a`/`Q9b`）、墙钟 434.4 s、stages `Q0,Q1,Q2,Q3,Q4,Q5,Q6,Q7,Q9,Q8`；Q0 六道输入钉全绿（163 抓图逐张 md5、140 参考帧独立复算哈希、52 行冻结表 + 唯一取景常量、四个嘴框探针↔计划逐框相等）是后面所有数的前提。发**五项 §5 五件齐发**（尺 / 噪声底 / R0 三件 / 归因 / 能否改论文）：嘴 (a) 追平（R2-ABS 存在性、底实测 0 px）、动作切换 (b) 候选（结构 (a) + 数值 5/6 超派生交界限）、眼 / 整体形状 / 物理 = MISSING-RULER，五项第 (v) 件逐条否 ⇒ **门槛 B 仍 0/5**。写入边界统一 `clean()`/`scrub()`（`backslashreplace`），把 run 5 账本两处 mojibake 清洗成可见 `\uXXXX` 转义而非删除（M11-⑦）。三个被取代的打分目录（`-20260925`、`-fixed1`、`-fixed2`）字节原样留档，其 sha256 记在现役 json 的 `superseded_first_published_run` 并已独立复算。现役目录 = `evidence/m11c-score-20260925-fixed3/`（`m11_bing_score.json` 1,220,714 B / `m11_bing_score.txt` 27,138 B / `README-20260925.md`）。
  - **它复用甲 的仪器、不重造尺**：Q1..Q7 调 `m05_region_score.py` / `m11_registration_framing.py` 的函数（不跑其 `main()`），在 1024² 全帧复算 R1..R7；`Q1a` 甲 的 Python 像素规则与探针的 C# 规则在 27+27 次比对上差 **0 px**（同一把尺的两个实现）。它**不重搜取景**（故 Q4/Q5 的 MAD 列只是甲3 那一个变换下的读数）、**不判眼/眉绝对达标**（2026-09-23 裁决）、`--limit` 冒烟轮不发表任何东西。
- `m11_mask_channel_vote.py`（新，遮罩四假设子腿）= 复用已发表的 M05-C 扫参总体（25 条，24 可判 + 1 点名跳过），逐框比 `masked` vs `nomask` vs `culled` 四假设投票。**6 道闸全绿**（`V2` 与已发表读数一致到 0、`worst_abs_diff = 0.0`），核心发现 = 遮罩在泪部 1 + 眉部 6 共 **7 个框里逐像素完全相同**（M11-⑥，与 M13 的"6/8 眉框零差异"同源），投票列 `masked < culled` 在整框与决定性点集上 **24/24 + 24/24** 成立。唯一"nomask 更近"的 `PARAM_EYELID_L` 差 −0.005507 MAD、建立在 17 个像素上、最大单通道台阶 5（远低于"变了"的 16 台阶）。跳过那条 = `PARAM_MOUTH_OPEN_Y_MANUAL`（index 31，Unity 侧 `changed_px = 0` 达不到 `min_px = 30`）。现役目录 = `evidence/m11c-mask-vote-20260925-fixed1/`（其 `-20260925` 前身被取代、字节留档、不被发表件引用）。
- 产品侧新增第三份入库运行时代码 `Assets/FengChuanXiangMoc3/Runtime/CubismMouthParameterPin.cs`（律⑤ 执行件，`PinExecutionOrder = 900`：物理 800 之后、渲染 10000 之前）—— 它由 乙 落码、丙 的打分读它的效果，登记账 = M11-①（9 组里 2 组在"钉"动手前"泵"已改过嘴，按冻结配对会给相反结论）。
- 表格闸清单 `m11_table_check_manifest.txt` 本轮往后追加两行（`-fixed3` 与 `-fixed1` 两份 丙 README）：往后的腿只往后追加自己新发表的表格件，不改上面已被发表件逐条引用的行（M10c 那四件冻结，理由见该清单头注）。追加式审计 `m11_append_only_audit.py` 的 FILES 现含 `issues/16`、`map.md`、`tools/README.md`、`issues/15`、M10c 证据 README、本仓记录、`issues/05`（两份 丙 证据 README 未提交前不入审计：untracked ⇒ `git show HEAD:` 取不到）。
- **2026-09-26 M14（观感复核，零 Unity）**：`m14_visual_prep.py` = 确定性合成器（输入只读：run 5 抓图与 M10c K8 帧来自本机 `tmp/`、M04 场景图与 M09 表情帧来自 `evidence/`；输出 25 件 = strip 18 / diff 5 / gif 2 落 `evidence/m14-motion-visual-20260926/`，标签纯 ASCII，逐件 sha256 写 `m14_manifest.json`，总入图 6.35 MB ≤ 8 MB 票内额度）。识图腿两份胶水脚本 **`tmp/m14_vision.py`（包装器：UTF-8 提示词文件 → subprocess → 答案文件，中文绝不进命令行）与 `tmp/m14_vision_run.py`（批跑器：20 job、外层超时 660 s > 包装器内层 600 s、答案合格性闸 = `exit=0` 且 stdout 有正文、失败单次重试、12 s 间隔）在 `tmp/`、不进 git** —— 教训 M14-①：嵌套子进程超时必须外层 > 内层、"剩余预算"绝不允许当单次超时传；两脚本可由证据 README §7 的描述逐字重建。表格闸清单本腿往后追加一行（M14 证据 README）；4 份识图提示词（`vision/prompts/`）开票即冻结、随证据入 git。
- **2026-09-26 M15（更正腿，零 Unity）**：`m15_fix_v43.py` = V4_3 表情页重出器（用户断点审核抓出中性格错用 m09b **A 段未泵锚** `feng_A_editorSync_neutral.png`（MD5 `fd3ddca0…` = M09 G1 unpumped ⇒ 遮罩未生效、齿/舌同画 = M05a-① 同型）；该器落盘前**逐字节断言**两锚 MD5（未泵 `fd3ddca0…` / 已泵门槛锚 `ae25133b…`），任一不符即拒写；只新增 `pkg/V4_3_expression_page-fixed-20260926.png`（66,750 B / sha256 `4adb71bb…`），**不覆写任何已发表件、manifest 一字不动**；差异账 = `CORRECTED-20260926.md`，登记 issues/05 M14-③）。**新律：断点交付件只准用已泵/遮罩生效锚**；m09b 目录 A 段未泵件仅作 G1 锚比对用途，禁止进交付件。
- **2026-09-26 M16（动作切换连续链逐帧实拍：一次 Unity + 两条零 Unity 腿）**：`m16_probe_compile_check.py` = 起跑前离线编译闸（Unity 自带 Roslyn `csc` + `-define:UNITY_EDITOR;UNITY_6000;…`，阳性对照 = 产物 ≥ 100,000 B 且 13 个必需符号全 present，否则 `SELF_CHECK=BROKEN`；本轮 `PROBE_COMPILE: PASS`、dll 480,256 B）· `m16_score.py` = H4 磁盘侧独立复算器（只读 PNG 原件重算 1,097 个 `drawn_px` 与 1,096 对 `changed_px`、逐张 md5、全部分布列，交界对照按每行自己声明的 `prev_file` 跟；S0 尺子回声 = 从探针源码正则读 5 个常量、漂移即红；仪器阳性对照四件；本轮 `VERDICT: MATCH`、per-frame=0 / columns=0）· `m16_pkg.py` = V5 交付件装配器（Pillow；六条交界 GIF = 25 源帧 @480px/60 ms + 全链概览 GIF @320px + 交界条页/部件页/保持尾页三页；`gif_audit()` 逐件重开 GIF 断言 `总时长 == 源帧数 × 每帧毫秒` 并把 `frames_in_gif_file` / `frames_merged_as_identical` 写进 manifest = **M16-① 的修法**；超额只许改抽帧步长与边长 ⇒ 本轮概览 5→7（220 帧 → 159 帧，两个数都由 `pick_count()` 按七段计划窗口真实数出，不是按比例反推 = **M16-③** 的修法）+ median-cut 128 色两条写进 `adaptations`；十件 7,783,084 B = 7.42 MB ≤ 8 MB，重跑逐件 sha256 不变）。识图两份胶水 = `tmp/m16_vision_prep.py`（25 格交界帧 / 159 格概览帧拼条页）与 `tmp/m16_vision_run.py`（外层 660 s > 内层 600 s + 答案合格性闸 + 失败单次重试 + 12 s 间隔），按 M14 先例留 `tmp/` 不进 git、可由 `evidence/m16-motion-chain-20260926/README-20260926.md` §6 描述逐字重建。表格闸清单 `m11_table_check_manifest.txt` 本票往后追加一行 `scratch|evidence/m16-motion-chain-20260926/README-20260926.md`（只往后追加自己新发表的表格件，上面已被引用行不动）。
- 输出编码沿用本票约定：`.txt` 写手 `encoding="ascii", errors="backslashreplace"`，`.json` 原样 UTF-8；诊断脚本自己写 UTF-8 文件再 Read，中文绝不进命令行参数。

## M17 腿工具登记（2026-09-26）

起因 = 用户 M16 断点原话"通过，接下来按你刚刚处理动作切换的方法，一并把眼睛，嘴，表情的切换一并做了，断点设置在工作完成之后" ⇒ M16 的逐帧连续实拍方法搬到**表情轴**（票面 `issues/19-m17-eye-mouth-expression-chain.md`，判据 K0..K6 在任何 Unity 进程之前冻结）。

- **M17-0（票面笔，本笔）已入库**：`m17_publish_inputs.py` = **M17 的输入发布器 / 冻结器**。跑法 `python tools/m17_publish_inputs.py`（无参数；输出两份 = `../tmp/m17_unity_manifest.json` 与 git 副本 `evidence/m17-inputs-20260926/m17_inputs.json`，两份**逐字节同**）。2026-09-26 本机实测：**P0..P16 共 50 道闸全绿**、`checks=[]`、`checks_failed=[]`、`verdict=FROZEN`、124,523 B、md5 `d4485649f01ad27b94eea763dfffcace`。它冻结的东西：七窗表达式链（含每窗参数表与 group/box 归属）、24 只框（读自 `m05_region_score.json`，不手填）、七条链 112 张烘焙参考帧的**逐张 md5**（= M17-② 的还账）、`place()` 配准三元组、168 帧抓取计划与命名、正弦权重预测序列、K0..K6 判据原文、以及"三条阈值只回声不再乘 2"的 `limits_frozen_echo`。**它拒绝在四种情况下发布**（都是实打实的 `gate()` 失败形状）：任何 `.exp3.json` 与 model3/m09 两处索引不一致、声明参数落在 M05 的 35 条死参数里、嘴框与 `m11 framing.mouth_boxes` 不逐矩形相同、零参数链的 16 帧不字节相同。踩过的两个坑（都随本笔入库）：① `carrier` 里没有 `bound_drawables` 这个键 —— bound 是**运行期普查**，在册期望在 `tolerances.r7`，P14 首轮因此 `KeyError`；② 烘焙帧根路径少了一层 `_bake`（`凤川祥/out/...` ⇒ 正确是 `凤川祥/_bake/out/...`），首轮表现为"参考件不存在"。
- **本票后续三件在收口笔入库**（此刻尚未存在，不预先登记读数）：`m17_probe_compile_check.py`（起跑前离线 csc 闸，阳性对照沿用 M16 配方）、`m17_score.py`（K6：从 182 张 PNG 独立复算整帧 / 24 框 / 并集 / 参考光栅化计数，含逐张 md5 抽验与**错配阳性对照**）、`m17_pkg.py`（V6 装配：七条逐窗面部 GIF + 概览 GIF + 交界条页 + Unity vs 参考并排页 + 组内计数曲线页；`gif_audit()` 与 `pick_count()` 两条 M16 律原样继承）。
- **M17-③ 的还账笔（本笔）**：`m11_append_only_audit.py` 的 FILES 补进 `issues/17`、`issues/18` 与 M14/M16 两份证据 README（此前"下一腿再加"的约定在两票上没兑现 ⇒ 那两票的台账行从未被审计看过）。`issues/19` 与本票证据 README 此刻 untracked，`head_lines()` 取不到 blob 会 `SystemExit`，故由 M17 收口腿加。表格闸清单 `m11_table_check_manifest.txt` 本笔往后追加一行 `scratch|issues/19-m17-eye-mouth-expression-chain.md`（只追加，上面已被引用行不动）。
- **M17 收口笔（2026-09-26，本笔）：上面"本票后续三件在收口笔入库"那条预告的三件已全部入库并实跑过**（前一条与再上面那条 M17-③ 还账笔一字未改，读数以下面为准；本条是追加，不是改写。另注：M17-③ 那条里说 `issues/19` "此刻 untracked" 是**该笔写作时**的状态 —— 它随后随票面笔 `f7ca8d2` 入库，故本笔的审计清单只需补它一件）：
  - `m17_probe_compile_check.py` = 起跑前离线编译闸（Unity 自带 Roslyn `csc` + `-define:UNITY_EDITOR;UNITY_6000;…`，阳性对照沿用 M16 配方 = 产物字节门槛 + 必需符号逐个 present）。本机实测 **`PROBE_COMPILE: PASS`**、csc 0 条 error、dll **564,736 B**、**14 个必需符号全 present**（含 `RunM17` / `M17Prepare` / `M17WindowFrame` / `M17Gates` / `CountChanged3` / `m17_region_pass`）、`M17_PROBE_COMPILE_EXIT: 0`。无参数，读探针源码自行编译。
  - `m17_score.py` = **K6 磁盘侧独立复算器**（跑法 `python tools/m17_score.py --run-dir <绝对路径> [--report <mask_probe_m17.json>]`；`--run-dir` 必填且按 `os.path.abspath` 相对 **cwd** 解析 ⇒ 从 `.scratch/moc3-showcase` 里跑时必须传**能落到仓根 `tmp/` 的路径**，否则复算器找不到原件）。它只读 PNG 原件重算 **168 个 md5 + 168 个 `drawn_px` + 167 对帧对 × 4 类计数（整帧 / 24 框 / 声明框并集 / 参考侧光栅化）+ 501 个组内数 + 112 张参考帧哈希**，另含 S7 表达式数组顺序普查（= M17-0-5 的磁盘证据）与**错配阳性对照**（本轮 **2,211 px > 0** ⇒ 尺子不是恒绿）。本机实测 **`VERDICT: MATCH` / `MISMATCHES: 0`**，输出 `tmp/m17_score.json` / `.txt`。
  - `m17_pkg.py` = V6 装配器（跑法 `python tools/m17_pkg.py --run-dir <已发表轮的绝对路径> --out <证据目录> --wrong-run-dir <第二轮绝对路径>`；三个路径同样按 cwd 解析，**`--wrong-run-dir` 目录不存在时第 13 件认错人对照页会被跳过 ⇒ 只出 12 件**，这是本轮踩过一次的坑，登记在此防重踩）。产出 **13 件 = 3,516,937 B（3.35 MB）≤ 8 MB 额度、`adaptations=[]`（零裁剪）**：七条逐窗面部 GIF（24 帧全帧 @260px、面部裁剪并集框 `[441,140,580,254]`）+ 全链概览 GIF（步长 3 / 240px / 100ms，抽样数由 `pick_count()` 数出来）+ 六处交界条页 + Unity vs 参考并排页 + 组内计数曲线页 + 认错人对照页；`gif_audit()` 逐件断言"源帧数 / 文件内帧记录数 / 总时长"三件（M16-① 的律），逐件 sha256 写 `m17_manifest.json`。**收口自查抓出的注记账**（M17 自查①）= 首版 `gif_frame_count_note` 断言方向反了（实测七条逐窗 GIF 全 24/24、0 并帧，只有概览并掉 19 帧），换成实测三段后**重跑十三件 sha256 逐件相同**（`sha256sum -c` 13/13 OK = 合成器确定性第三次独立复证）。
  - 识图两份胶水 = `tmp/m17_vision_prep.py`（七条逐窗条页 + 概览 + 每窗末帧 vs 参考末帧并排页）与 `tmp/m17_vision_run.py`（外层 660 s > 内层 600 s + 答案合格性闸 + 失败单次重试 + 12 s 间隔；退出码靠任务通知读 = M16-② 的律），按 M14/M16 先例**留 `tmp/` 不进 git**，可由 `evidence/m17-eye-mouth-expression-20260926/README-20260926.md` §6 与 `vision/vision_inputs.json` 逐字重建。
  - 本笔另追加：表格闸清单 `m11_table_check_manifest.txt` += `scratch|evidence/m17-eye-mouth-expression-20260926/README-20260926.md`（表格闸读工作树文件，untracked 也能进）；追加式审计 `m11_append_only_audit.py` 的 FILES **本笔只 += `issues/19-m17-eye-mouth-expression-chain.md`**（它随票面笔 `f7ca8d2` 已 tracked ⇒ `git show HEAD:` 取得到）—— 上面那份 M17 证据 README **本笔仍不加**：它随本笔才入库，`head_lines()` 对取不到 blob 的路径直接 SystemExit，与 M11-丙 那条"untracked ⇒ 不入审计"的理由同源，**由 M17 回执笔加**（回执笔跑在收口笔之后，那时 HEAD 已有 blob）。探针 `Mode.M17` 在既有 `Assets/Editor/FengMoc3MaskProbe.cs` 内，**无新 `Assets/` 文件 ⇒ 无补交 `.meta`**。
  - **M17 自查③（本笔的 append-only 纪律违规，登记不静默）**：往两处**长表尾**追加本票新行时，两次把**已发表行整行**当 Edit 锚点顶掉 ⇒ 追加式审计抓到 `!! NON-APPEND HEAD 120`（`记录_实时moc3线.md` §3 的 **M17-0 开票行**，1,548 字符）与 `!! NON-APPEND HEAD 265`（本文件上面那条 **M17-③ 还账笔 bullet**，347 字符）。处置 = 逐字节恢复（记录侧由 `tmp/fix_rec120.py` 从 `git show HEAD:` 取原文插回，本文件侧直接写回 HEAD 文本），新内容一律改成**追加** ⇒ 修前 `tools/README.md replace=1 / non-append-edits=1`、记录 `replace=5 / non-append-edits=1`；修后两文件 `non-append-edits=0`、`TOTAL_DELETED_HEAD_LINES=0`、表格闸 `TOTAL_BAD=0`；记录 §3 由 248 行 → **249 行**。**律 = 往"一行一条"的长表尾追加行时，Edit 锚点必须落在表体之外的下一段（`---` / 标题行），不得用任一已发表行当锚点**。全文见 `issues/05` 的 M17 收口节 自查③ 行与 `evidence/m17-eye-mouth-expression-20260926/README-20260926.md` §7 第 6 条。
  - **M17 回执笔（2026-09-26，本笔）= 纯台账笔：不改任何已发表读数、不动判据、不新增证据件**（本笔 5 个文件全是台账与闸清单）：① 把 M17 证据 README 追加进 `m11_append_only_audit.py` 的 FILES（兑现上面那句“由 M17 回执笔加”；它已随收口笔 `c335838` 入库 ⇒ `head_lines()` 取得到 blob，且本笔对它一字未动 ⇒ 审计对它 0 差）；② 把收口笔哈希 `c335838` 回填到六处挂点（字符串落点共 **7** 处 = 下面六项里 §9 那条 M17 行占首尾两处；口径 = 按**出现次数**数（`str.count`）⇒ 记录 4 + map 2 + issues/05 1 = 7；若按**行数**数则是 3 + 2 + 1 = 6 行）（记录 §1 表末行 / §3 执行腿行 / §9 M17 条首尾各一处、`map.md` 的 M17 台账行与 Decisions 第 13 项、`issues/05` 的 M17 节末行）。③ **本笔自己的一处同类违规（自查③ 的第二次，已就地修正）**：第一次回填 `issues/05` 第 424 行时把哈希句插在**原句中间**，而该行只有 144 字符、短于审计的 200 字符前缀门槛 ⇒ 短行的任何中间插入都不可能满足"前缀保留 + 变长"，审计立刻报 `!! NON-APPEND HEAD 424`；处置 = 用 `tmp/m17_fix_424.py` 按 `git show HEAD:` 整行还原后改为**行尾追加**（实测 = 审计的 `keep-prefix HEAD 424 … 144 chars, work 538 chars`，即 144 → 538 字符）。**律补一句：短于 200 字符的已发表行只能尾部追加。** 两闸在本笔最后一次编辑之后、提交之前复跑。**本回执笔自身的哈希由下一票（M18）第一笔回填**（“一笔装不下自己”）。
  - **M17 交付更正笔（2026-09-26，本笔）= 交付格式更正 + 一道新闸，零读数变更、零判据变更**：断点报告里 V6 十三件的 15 个链接目标被写成 CommonMark 尖括号包 Windows 反斜杠路径 ⇒ 渲染器交出去的目标不存在（用户侧 = "验收视图文件不存在"），而文件本身 13/13 在库（`git ls-files` 命中 13、逐件 `os.path.isfile` 为真、字节数与 `m17_manifest.json` 相符）⇒ **归因 = 链接写法错，不是证据缺失**。已改回历票验证过可渲染的写法（`file:///` + 正斜杠 + 空格转 `%20`），并入库新闸 `tools/m17_delivery_link_gate.py`（跑法 `python tools/m17_delivery_link_gate.py --links <目标清单> [--report <文件>]`；四条判据 = 必须 `file:///` 前缀 / 不得含尖括号 / 不得含裸空格与反斜杠 / 百分号解码后必须是仓内存在的非空文件；报告 `tmp/m17_link_gate.txt` 末行 `TOTAL_BAD`，非 0 时 exit 1）。两份清单同笔入库 = 正例 `tools/m17_delivery_links.txt`（本票 15 件真目标 ⇒ 实测 **TOTAL_BAD=0 / exit 0**）与反例 `tools/m17_delivery_links_negative.txt`（五种真实错法 ⇒ 实测 **TOTAL_BAD=5 / exit 1**，证明这道闸抓得住本笔踩的错）。**律 = 发断点报告之前必须先跑这道闸，`TOTAL_BAD` 非 0 不得交付。** **另记本笔自己的纪律账（自查③ 的同类第三次，已就地修正）**：往 `issues/05` 末尾追加"交付更正笔"那段时，锚点取的是已发表第 424 行的**行尾串**却只到倒数第二个字符，而该行真正的最后一个字符是收尾括号"】"⇒ 括号被顶进新段尾部、第 424 行短了一个字符 ⇒ 审计当场报 `!! NON-APPEND HEAD 424`（同一行、同一类错第三次）；处置 = 把"】"放回 424 行原位。**精确律 = 用"行尾串"当 Edit 锚点做追加时，锚点必须一直取到该行真正的最后一个字符（含收尾括号），取半行就会把尾巴顶到新段。** 本笔动 5 个文件 = `issues/05`、本文件、闸本体、两份清单；三个新 `tools/` 文件随本笔才入库 ⇒ 按 M17 律③（untracked 不入审计），`m11_append_only_audit.py` 的 FILES 由 **M18 第一笔**登记。两闸在本笔最后一次编辑之后、提交之前复跑。**本笔哈希由 M18 第一笔回填，与 `a629f4e` 一并（同一笔可还多笔债）。**
- 输出编码沿用本票约定：`.txt` 写手 `encoding="ascii", errors="backslashreplace"`，`.json` 原样 UTF-8；诊断脚本自己写 UTF-8 文件再 Read，中文绝不进命令行参数。


- **M17 交付机制笔（2026-09-26，本笔）= 换交付通道 + 把链接闸升级到"零转义" + 新增发图前强制预检**（不改任何已发表读数、不动判据、不新增证据件；证据 `pkg/` 十三件一字未动，本笔只动 `tools/` 与台账）：
  - **为什么还要再改一次**：上一笔（交付更正笔 `3ff330d`）把链接写成 `file:///` + `%20` 并保留了裸圆括号 `(2)`，用户第二次仍报"验收视图文件不存在"。实测根因 = 仓库根 `C:/Users/Administrator/My project (2)` 自带空格与括号 ⇒ **仓内任何绝对链接都必须转义**，而转义写法由渲染器决定，这层不确定性不可消除；同时上一笔声称的"历票验证过可渲染"经 transcript 复查为**零证据**（两份记录里 `![` 与 `](file:///` 命中均为 0），属把假设写成结论。详见 `issues/05` 的"M17 交付机制笔"节。
  - **`tools/m17_publish_acceptance_view.py`（新）**：把 `--run-dir` 的 `m17_manifest.json` 逐件按 sha256 校验后复制到**无需转义的镜像目录**（仓库外），复制件再哈希一次对 manifest；生成自包含 `acceptance_view.html`（13 图 base64 内联）并**从镜像路径生成报告链接表**（`--links-out`，杜绝手写目标）；写完 HTML 立刻做 **ROUNDTRIP**：正则取出每段 base64、解码、sha256，与 manifest 的图片集合逐一相等，缺/多/解不开都算红。本笔实跑 = `ARTIFACTS manifest=13 verified=13 mismatched=0` / `ROUNDTRIP ok=13/13` / `LINKS emitted=16 rejected=0` / `MIRROR_TOTAL_BYTES 8,268,921` / `RESULT OK`。命令：`python tools/m17_publish_acceptance_view.py --map-root "C:/Users/Administrator/My project (2)/.scratch/moc3-showcase" --run-dir evidence/m17-eye-mouth-expression-20260926 --dest "C:/Users/Administrator/moc3_evidence/m17-20260926" --links-out tools/m17_delivery_links.txt`
  - **`tools/m17_delivery_link_gate.py`（升级）**：判据从"`file:///` + 无空格 + 无尖括号 + 在仓库内"改成"`file:///` + **零转义字符**（`< > " 空格 \ ( ) %` 一律禁止）+ 百分号解码后是存在非空文件 + 位于 `--allow-root` 交付根之下"。红绿对拍实跑：**上一笔发给用户的那份 15 条表在新闸下 = `TOTAL_BAD=15` / exit 1**（旧闸曾判 0，这就是补闸前后的差别）；镜像 16 条 = `TOTAL_BAD=0` / exit 0；负例 fixture 重写为 6 条不同成因 = `TOTAL_BAD=6` / exit 1。
  - **`tools/delivery_preflight.py`（新，发图前唯一入口）**：按序跑 ① 表格闸 ② 追加式审计 ③ 当前链接表 `TOTAL_BAD=0` ④ **牙齿测试**（负例 fixture 红数必须等于其非注释行数，否则 `!! GATE HAS NO TEETH`）⑤ 镜像文件数/字节数清点，全绿才打印 `DELIVERY_READY 1`。本笔实跑 = `[table_gate] exit=0 TOTAL_BAD=0` / `[append_audit] exit=0 TOTAL_DELETED_HEAD_LINES=0` / `LINK_GATE dests=16 ok=16 bad=0` / `TEETH expected_bad=6 got=6 ok_on_bad=0` / `MIRROR files=16 present=16 total_bytes=8268921` / `DELIVERY_READY 1`（报告 `tmp/delivery_preflight.txt`）。命令：`python tools/delivery_preflight.py --links tools/m17_delivery_links.txt --negative tools/m17_delivery_links_negative.txt --allow-root "C:/Users/Administrator/moc3_evidence"`
  - **律（本笔新加，下一票直接生效）**：① **发证据图之前必须 `DELIVERY_READY 1`**，不为 1 不得交付；② 交付通道 = 镜像目录里的**无转义路径**，报告里的链接目标只能由 `m17_publish_acceptance_view.py` 生成，不许手写；③ 每次交付同时给出**一条完全不依赖链接解析的通道**（自包含 HTML 的纯文本路径），链接失效时用户仍有可用面；④ 台账里写"某写法已验证"必须附可重跑的测量，否则标为假设；⑤ 牙齿测试属于闸的一部分——只验"该绿的绿"而不验"该红的红"，闸会在无人察觉时变成假绿。
  - **本笔另加纪律改法**：两处台账追加改走 `tmp/m17_append_ledger.py`（EOF 追加写手），不再用已发表行的行尾串当 Edit 锚点（自查③ 同类失手三次的共同根因）。
  - **在册账**：`tools/m17_publish_acceptance_view.py`、`tools/delivery_preflight.py` 两个新 `tools/` 文件本笔随笔入库，但**不得进追加式审计 FILES**（`head_lines()` 对无 HEAD blob 的路径直接 SystemExit），由 **M18 第一笔**登记；本笔自身的哈希亦由 **M18 第一笔**与 `a629f4e`、`3ff330d` 一并回填（"一笔装不下自己"）。

- **M18 第一笔（2026-09-27，本笔）= 表情塌缩诊断件 + 泛化交付件 + 两处闸清单登记，零读数变更、零判据变更**（用户裁决原文与三条效力见 `issues/05` 的 M18 节与本文件同日的 M18 段；本笔不动 M17 任何已发表读数）：
  - **`tools/m18_expression_collapse.py`（新，本票唯一生成器）**：零 Unity，只读 M09-B 已发表的 **218 张落定帧** + 216 对 v2↔v3 表达式 + `tools/m17_publish_inputs.py` 的 `EXPRESSIONS` 七格，跑六腿（A 绑定普查 / B 源侧交叉核对 / C 逐帧像素聚类 / D 抽象互异七名 / E M09-B 档位对账 / F 演示栏修法）与八道闸，输出 `evidence/m18-expression-collapse-20260927/`（报告 + CSV + `m18_inputs_digest.json` + `m18_expression_set.json` + 三张概览图 + `m18_manifest.json`）。跑法 = 从映射目录 `python tools/m18_expression_collapse.py`；**退出码 = 红闸数，本票恒为 1**（G-A 有意保留为红 ⇒ "exit 1"在这条线上不是失败信号，而是发表口径的一部分）。尺子全部沿用已发表口径：可见地板 **30 px**、弱档边 **300 px**、changed px = 四通道任一 > 16 级；筛子 = 128×128 RGBA、>1 级的格子 ≤ 32 判为近重复（筛子形状与入选数写在报告里，M18 流程账③）。
  - **`tools/publish_acceptance_view.py`（新，泛化交付通道）**：把 `m17_publish_acceptance_view.py` 的五条规则（无转义目录 / 逐件 sha256 复校 / base64 自包含 HTML / 内联字节回读对 manifest / 链接表由工具生成）参数化（`--manifest` `--companion` `--title` `--lead` `--allow-root`）。**为什么不直接给 M17 那件加开关**：那件自本笔起进追加式审计 FILES（见下条），改它的硬编码 `m17_manifest.json` / `README-20260926.md` / 标题就是对已发表工具做非追加改动，正是那道闸要抓的形状 ⇒ **律 = 已受追加式审计冻结的工具需要新行为时，另立泛化件并在两处登记理由，不改冻结件。**
  - **两处闸清单登记（本笔兑现 M17 留下的三笔在册账）**：① `tools/m11_append_only_audit.py` 的 FILES 自本笔 += 五个早已入库的 M17 交付件 = `tools/m17_delivery_link_gate.py`、`tools/m17_delivery_links.txt`、`tools/m17_delivery_links_negative.txt`、`tools/m17_publish_acceptance_view.py`、`tools/delivery_preflight.py`（兑现 `tools/README.md` 第 **274**、**285** 行与 `issues/05` 第 **426**、**441** 行的"由 M18 第一笔登记"）⇒ **本笔对这五个文件一字未动**，审计对它们应为 0 差；② 表格闸清单 `tools/m11_table_check_manifest.txt` += `scratch|issues/20-m18-expression-collapse.md` 与 `scratch|evidence/m18-expression-collapse-20260927/README-20260927.md`（表格闸读工作树文件，untracked 也能进）。**本笔不入 FILES 的** = `tools/m18_expression_collapse.py`、`tools/publish_acceptance_view.py`、`tools/m18_delivery_links.txt`（由 publisher 生成的本票正例清单，随本笔入库）、`issues/20`、M18 证据 README —— 它们随本笔才入库，`head_lines()` 对无 HEAD blob 的路径直接 SystemExit（M17 律③同源）⇒ **由 M18 回执笔或 M19 第一笔登记**。**随之生效的一条新律**：清单入库即冻结 ⇒ **将来重跑 publisher 不得回写 `tools/m18_delivery_links.txt`，要新行为就另立新文件（`tools/mNN_delivery_links.txt` 形状）**，与"冻结件需要新行为时另立泛化件"是同一条纪律的两面。
  - **交付机制实跑（本票断点件）**：`python tools/publish_acceptance_view.py --map-root "<repo>/.scratch/moc3-showcase" --run-dir evidence/m18-expression-collapse-20260927 --dest "C:/Users/Administrator/moc3_evidence/m18-20260927" --links-out tools/m18_delivery_links.txt --manifest m18_manifest.json --companion README-20260927.md --allow-root "C:/Users/Administrator/moc3_evidence"` ⇒ 镜像 = 7 件产物 + 2 件 companion + 1 张自包含 HTML，随后 `python tools/delivery_preflight.py --links tools/m18_delivery_links.txt --negative tools/m17_delivery_links_negative.txt --allow-root "C:/Users/Administrator/moc3_evidence"` 必须末行 **`DELIVERY_READY 1`** 才许发图（M17 律①，本票原样执行）。
  - **哈希账**：M17 回执笔 = **`a629f4e`**、交付更正笔 = **`3ff330d`**、交付机制笔 = **`74930a5`**，三笔由本笔按**行号引用**兑现（不动已发表行）。**本笔自身的哈希由下一票（M19）第一笔回填**（"一笔装不下自己"）。



- **M19 收口笔（2026-09-27，本笔）= 修复栏实拍四件工具 + 交付链接清单登记，零判据变更**（票面 = `issues/21`，判据 W0..W7 开票即冻结；唯一数值出口 = `evidence/m19-expression-chain-live-20260927/README-20260927.md`）：
  - **`tools/m19_publish_inputs.py`（新）**：发布修复后七格栏的冻结输入（闸 P0..P16）——从盘上实 `start_fcx.txt`（改后冻结靶子 md5 `b916bd9f…`）重解析七名并与票面窗序对拍；输出 `evidence/m19-inputs-20260927/`（manifest + 逐窗输入），其 md5 即探针 W0 的内钉值。
  - **`tools/m19_probe_compile_check.py`（新）**：零 Unity 编译预检（盘侧 Roslyn），Unity 腿之前跑，防一次 batchmode 烧在编译错上（M17 先例件的本票后继）。
  - **`tools/m19_score.py`（新）**：零 Unity 磁盘侧复算器——只读 PNG 字节 + 逐帧 JSON，两把尺（整帧 `changed_px(>3)` 与 `changed_px(>16)`）判 W0..W6（含主闸 W3 的 21 对、W5 交界读数、控制组与 S7 序普查），两尺各带故意错配自检证明尺非常绿；退出码 0 = `VERDICT: MATCH` 且 W0..W6 全绿。**`--run-dir` 必须绝对路径**（相对路径会落到 `.scratch/tmp`，本票实跑踩过）。
  - **`tools/m19_pkg.py`（新，冻结件 `m17_pkg.py` 的通用后继）**：装配 V7 交付包（落定页 / 21 对矩阵页 / 六条交界 GIF / 全链 GIF + 复算两件）到 `evidence/m19-expression-chain-live-20260927/`，产 `m19_manifest.json`（发布器 schema：`artifacts[{name,bytes,sha256}]`，name 相对运行目录）；**尺的符号直接从 `m19_score` import，不复制第二把尺**；锚点列 = 本件从 PNG 字节现算的新读数、登记于 manifest 并注明出处；`verdict != MATCH` 或有 mismatch 行即拒绝出包；每条 GIF 的源帧/记录/合并/时长走 `gif_audit` 在册（M16-①③）。按"被审计冻结的闸要新行为用通用后继工具、不改原件"律，`m17_pkg.py` 一字未动。
  - **`tools/m19_delivery_links.txt`（新，发布器生成、不许手写）**：14 条 `file:///C:/Users/Administrator/moc3_evidence/m19-20260927/` 镜像链接；**随本笔入库即冻结** = 已入库链接清单不复写，M20 交付另立 `m20_delivery_links.txt`。
  - **清单登记切分（按律③）**：`issues/21` 随本笔进 `m11_append_only_audit.py` FILES（开票笔 `b7ae2a4` 起 tracked ⇒ 审计对它应为纯 INSERTED）；上列四件 py + 本链接清单 + M19 证据 README 随本笔才入库（`head_lines()` 取不到 blob）⇒ **由回执笔登记**。探针 `Mode.M19` 在 `Assets/Editor/FengMoc3MaskProbe.cs`（M17 常量与行为字节不变，`m17_score.py` 正弦闸原样保绿）。
  - **哈希账**：开票笔 = **`b7ae2a4`**，由本笔在 `issues/21` §7 与 `map.md` M19 收口段按**行号引用**兑现（不动已发表行）。**本笔（收口笔）自身的哈希由 M19 回执笔回填**（"一笔装不下自己"）。


- **M19 回执笔（2026-09-27，纯台账腿，零读数变更）**：
  - **哈希账**：收口笔 = **`d382f55`**，由本笔在 `issues/21` §8、`map.md` M19 回执段、本仓记录 M19 回执段与 `issues/05` M19 回执节按**行号引用**兑现（不动已发表行）。
  - **清单登记闭环**：`m11_append_only_audit.py` 的 FILES 自本笔 += 随收口笔入库的六件 = `tools/m19_publish_inputs.py`、`tools/m19_probe_compile_check.py`、`tools/m19_score.py`、`tools/m19_pkg.py`、`tools/m19_delivery_links.txt`、`evidence/m19-expression-chain-live-20260927/README-20260927.md`（本笔对六件一字未动 ⇒ 审计 0 差；"取不到 HEAD blob 由下一笔登记"的历轮约定至此全部清偿）。`tools/m19_delivery_links.txt` 自入库起冻结 = 已入库链接清单不复写，M20 交付另立 `m20_delivery_links.txt`。
  - **两闸（末次编辑后跑）**：`m11_append_only_audit` `TOTAL_DELETED_HEAD_LINES=0`；`m10c_table_integrity_check --files-from m11_table_check_manifest` `TOTAL_BAD=0`。
  - **本笔（回执笔）自身的哈希由 M20 第一笔回填**（"一笔装不下自己"）。**断点** = 用户审 `C:/Users/Administrator/moc3_evidence/m19-20260927/`；**下一票编号 = M20**。


## M20 开票账（2026-09-27，本节只登记票面与还账；工具账由收口笔兑现）

- **哈希账兑现**：M19 回执笔 = **`9b69fed`**（本文件第 310 行挂点"本笔（回执笔）自身的哈希由 M20 第一笔回填"由本节兑现，不动已发表行）。
- 本票 = `issues/22`（零 Unity 打包文献票）。计划新工具 = `tools/m20_pack_model.py`（模型包装配 + G-P1/G-P2 包内校验）、`tools/m20_verify_workflow.py`（G-P3 工作流引用闭合校验器）、`tools/m20_delivery_links.txt`（发布器生成、新立不复用 M19 冻结件）——本笔时刻均 untracked，按律（`head_lines()` 取不到 HEAD blob 的不得进追加式审计 FILES）由**收口笔/回执笔**分批登记；本节对已入库件一字未动。门槛 B 仍 0/5；本票交付措辞 = 资源交付 / 验收视图，不是达标。



## M20 收口账（2026-09-27，工具账兑现；零 Unity）

- 本票四件工具随收口笔入库：`tools/m20_pack_model.py`（模型包装配 + G-P1 闭包/G-P2 零漂移/G-P4 完好性，报告 `m20_pack_report.txt`）；`tools/m20_verify_workflow.py`（G-P3 引用闭合校验器，注记格式 `〔出口: 路径｜锚:"..."〕`，CRLF→LF 归一后逐条字面匹配，06 按设计除外）；`tools/m20_pack_workflow.py`（工作流包装配：结构闸七件无夹带 + G-P3 报告新鲜度前置检查（mtime 比对）+ MANIFEST mtime 钉死（确定性双跑）+ G-P4）；`tools/m20_publish_delivery.py`（G-P5 发布器：两 zip 与报告 sha 对账、镜像 README、自包含 HTML 兜底、链接表经链接闸生成 `tools/m20_delivery_links.txt`）。四件与链接清单本笔提交前 untracked ⇒ FILES 登记由 M20 回执笔兑现；`issues/22` 已随本笔进 FILES。
- 复现链（实际跑过的参数）：`python tools/m20_pack_model.py` → `python tools/m20_verify_workflow.py`（改内容后必须重跑再装配）→ `python tools/m20_pack_workflow.py`（可连跑两次对 sha）→ `python tools/m20_publish_delivery.py` → `python tools/delivery_preflight.py --links tools/m20_delivery_links.txt --negative tools/m17_delivery_links_negative.txt --allow-root "C:/Users/Administrator/moc3_delivery"` ⇒ `DELIVERY_READY 1`。门槛 B 仍 0/5。



## M20 回执账（2026-09-27，FILES 登记兑现；零 Unity）

- 兑现收口账节内“FILES 登记由 M20 回执笔兑现”一句：`tools/m20_pack_model.py`、`tools/m20_verify_workflow.py`、`tools/m20_pack_workflow.py`、`tools/m20_publish_delivery.py`、`tools/m20_delivery_links.txt` 五件已随收口笔 `df436d1` 入库（HEAD 取得到 blob），自本笔起进 `m11_append_only_audit.py` FILES 受追加式约束；本笔对五件与全部已入库工具一字未动。`m20_delivery_links.txt` 自入库即冻结（律：已入库链接清单不复写，复跑写新文件名）。门槛 B 仍 0/5。



## M21 开票账（2026-09-30，工具账候收口笔；零 Unity）

- 本票计划新件 = `Assets/Editor/FengMoc3Psd2LiveProbe.cs`（RunM21：逐样本导入 tml/ds、G-Q1 计数、1024² 中性 + 18 参数端点普查抓图，采集件不作判据）与 `tools/m21_score.py`（G-Q4 磁盘侧独立复算器，m16/m17/m19 先例形状）；零 Unity 编译预检 = Unity 自带 Roslyn（`m05c_unity_csharp_check.sh` 的后继件，参数化到探针新文件名）。本笔时刻均 untracked ⇒ FILES 登记由收口笔/回执笔分批兑现（历轮同因）。`issues/23` 随本笔起 tracked、由收口笔进 FILES。门槛 B 仍 0/5。



## M21 收口账（2026-09-30，工具账兑现）

- 本票实跑入库四件：`tools/m21_probe_compile_check.py`（Unity 自带 Roslyn 磁盘编译预检 + 阳性对照 symbols 4/4，m05c 形状的通用后继，不改 m05c 一字节）；`Assets/Editor/FengMoc3Psd2LiveProbe.cs`（采集件：2 样本 × 37 抓图 + 逐参数两端点普查，退出码不作判据）；`tools/m21_score.py`（G-Q1..Q4 磁盘复算器：cdi3/model3 对数、背景差分 census 并列输出、72 张重算对账）；`tools/m21_pkg.py`(装配器：changed=0 端点图与中性锚字节并锚去重 18→14，manifest 由程序生成、逐件 sha 回读)。发布复用通用件 `publish_acceptance_view.py`（参数化调用，未改件），链接清单新立 `tools/m21_delivery_links.txt`（自入库即冻结）。四件工具与链接清单、证据 README 由 M21 回执笔进 FILES；`issues/23` 随本笔已进。复现链全文 = `evidence/m21-psd2live-unity-20260930/README-20260930.md` §8。门槛 B 仍 0/5。



## M21 回执账（2026-09-30，FILES 登记兑现）

- 兑现收口账内"由 M21 回执笔进 FILES"：`tools/m21_probe_compile_check.py`、`tools/m21_score.py`、`tools/m21_pkg.py`、`tools/m21_delivery_links.txt`、`evidence/m21-psd2live-unity-20260930/README-20260930.md` 五件自本笔起受追加式审计约束（随 `d68be78` 入库、HEAD 取得到 blob；本笔对其一字未动 ⇒ 审计 0 差）。链接清单冻结律照旧（复跑写新文件名）。门槛 B 仍 0/5。



## M22 开票账（2026-09-30，工具账候收口笔）

- 计划新工装（全部后继件，不改冻结件）：`tools/m22_extract_normalize.py`（包装 extract_legacy_rig_v2/normalize 到爱音路径）；`tools/m22_run_legacy_import.ps1`（gradlew :cli:run 命令行冻结）；`tools/m22_e2e3_patch.py`（m12 E2/E3 泛化到任意 baked 件）；`tools/m22_convert_sidecars.py`（exp→exp3、mtn→motion3 转换器 + physics 翻译后继，模板 = 凤川祥现役件）；`tools/m22_wire_model3.py`（wired 迁移 + fade 撞名普查）；打分/交付 = m21_score/m20 打包形状后继。本笔时刻均 untracked ⇒ 收口/回执笔分批进 FILES；`issues/24` 随开票笔入库、由收口笔进 FILES。门槛 B 仍 0/5。



## M22 工具账（2026-10-01，收口笔追加）

爱音这趟新写 13 件，全部"后继件不改冻结件"，一次跑完即入库：

| 件 | 一句话职责 / 判据形状 |
|---|---|
| `m22_measure_distortions.py` | 三失真断言器（`csmHasMocConsistency` 走 native；混合标志在 v3 盘上**量不到**，注册为 `not_measurable_on_v3_disk`，改由 `m12_raster2.py` 的 `blend_counts` 出证） |
| `m22_e2_groupfix.py` | E2 通用后继件：填 draw-order 三件套 + 元数据槽 + 尾部重定位；**阳性对照 = 凤川祥案逐字节复现**，压力扫全参数×3 + 随机 240 |
| `m22_fade_collision_census.py` | 撞名普查双键版（`--key base\|fullpath`）；`--self-check` 必须同时复现凤川祥**两组在册值**（128/198/71/#42 与 0/0/397）才允许发表新模型数字 |
| `m22_optional_prep.py`、`m22_range_manifest.py` | 运动派生可选参数区间；授权核被本机浏览器占死时**诚实替代品**的清单器（证据串写明"不是授权 runtime 轨迹"） |
| `m22_assemble_package.py` | 成品包装配 + wired 闭包计数（顶层键形状对齐凤川祥现役件：`Version/FileReferences/FileFormats`，Groups 不迁） |
| `m22_publish_unity_inputs.py` | Unity 输入清单：216 张表情预期落地值、8 段动作峰帧值、物理计数、全部对照**现场数**；摘要**双发**（字节 + 文本，CRLF 教训） |
| `m22_stage_assets.py` | 落件 + 闭包复核；内容不同**拒覆盖**；`--prune` 清 Unity 生成物（中止的导入留半截） |
| `m22_probe_compile_check.py` | 通用 C# 编译闸后继件（`--source/--symbols/--self-check`，`-define:UNITY_EDITOR` + 产物字节与符号阳性对照） |
| `m22_score.py` | 独立复算器：只读 PNG 字节与 JSON，重算全部发表数并逐闸打分；**负例已过**（拿表情腿失效那次跑，必须红） |
| `m22_run_anim_preflight.py` | 不改冻结闸、只换路径常量原样调用 M10 的 `.anim` 事件普查；凤川祥侧必须复现六个在册数字 + 载体 md5 |
| `m22_pkg.py` | 证据装配 + **byte-duplicate 合并**（并入中性者打印张数）+ 生成 manifest |
| `m22_make_zip.py` | M20 三判据同款（G-P1 闭包 / G-P2 逐件零漂移 / G-P4 完好）+ **mtime 钉死确定性** + 零转义仓外根 |

探针一份：`Assets/Editor/AienMoc3ParityProbe.cs`（A 导入 / B 加载计数 / C 按 M10b 那把尺读 fade 表 /
D 216 张表情 / E 动作静帧 / F 真 PlayMode 播放+物理 / G 凤川祥**同尺只读对照**），
每腿写完即 `WriteReport`，单段抛异常只记 `play_threw`/`leg_failures` 不带走整链。



## M22 回执笔工具账追加（2026-10-01）

- `tools/m22_play_picks.py`：把 fade 表的 `CubismFadeMotionObjects` 槽位与 `MotionInstanceIds`
  （十六进制小端 int32 数组）逐槽对齐，输出 397 段的**可播性三分**（clean_bound / unbound_hardcut /
  shadowed）与补测抽样计划 `tmp/m22_play_plan.json`。判据形状 = 三分必须**恰好覆盖 397 且无剩余**；
  它同时是**可推翻的预测器**（shadowed 应抛、clean 应播），实测结果记在证据 README 的补测段。
  只读，不写 `Assets/**`。
- 探针 `AienMoc3ParityProbe` 的 F 腿改为优先吃这份计划（无则回退主清单，并把所吃文件的 md5
  钉进报告 `F_pick_plan_md5`）；`tools/m22_score.py` 增一条披露闸
  `playability_partition_predicted_correctly`（不动冻结条款）。


## M22 还账笔工具账追加（2026-10-01）

- `tools/m22_score.py`：**入库补记** —— `playability_partition_predicted_correctly` 那 22 行源码
  随本笔才入库（它产出的 `m22_score_report.txt` 已随 `c7a5d50` 入库；本笔让报告与生成它的
  源码重新同批）。冻结条款一条未动，对 HEAD 为纯插入。
- `evidence/m22-aien-unity-20261001/pkg/`：随本笔入册三条删除（M21-③ 并帧：
  `aien_F_anon_angry01.png`、`aien_F_anon_live_event_253_ur_gacha01.png`、
  `aien_F_soyo_kandou01.png` 与幸存者字节相同 ⇒ 并入一张），pkg 恢复 **24 png**，与
  `m22_manifest.json` 的 31 件一致；镜像侧三张孤儿副本同步删除，链接闸复跑仍绿。
- `tools/m22_play_picks.py` 自本笔起进 `m11_append_only_audit.py` FILES（随 `c7a5d50` 入库，
  本笔对它一字未动）；`Assets/Editor/AienMoc3ParityProbe.cs` 按 M21 先例不入 FILES。
- 哈希账：回执笔 `c7a5d50` 由本笔按行号引用兑现。门槛 B 仍 0/5。


## M23 开票工装计划（2026-10-01，全部为后继件，不改冻结件）

- `tools/m23_fade_list_census.py`：模型目录下 **全部** `*.fadeMotionList.asset`（模型级 + 目录级）逐张普查
  （槽位 / 空槽 / 去重 id / 重复 id / 悬挂 guid），`--self-check` 必须同时复现爱音 497/100/402/95、
  凤川祥 397/0/397/0 与两侧 7 张表逐张槽位全等。`m22_play_picks.py` 的通用后继，不改它一字。
- `tools/m23_flatten_preview.py`：G-F2 压平算术 + 8 段 shadowed 抽样计划 `tmp/m23_play_plan.json`（跨 ≥5 成员）。
- `Assets/Editor/AienFadeFixProbe.cs`（`RunM23`）：净室复制 + 单次导入 + 盘上读表 + 双实例真 PlayMode；
  配套 `tools/m23_probe_compile_check.py`（m22 形状，参数化）。
- `tools/m23_score.py`、`tools/m23_delivery_links.txt`：零 Unity 复算与交付面（兑 M22 形状）。
- 本笔时刻均 untracked ⇒ FILES 登记由收口笔/回执笔分批兑现（历轮同因）。门槛 B 仍 0/5。


## M23 收口工具账（2026-10-01，五件新件 + 一次进程）

- `tools/m23_fade_list_census.py` 模型目录下全部 fade 表逐张普查 + `--self-check`（复现爱音 497/100/402/95
  与凤川祥 397/0/397/0、两侧 7 张表逐张槽位全等）；`m22_play_picks.py` 的通用后继。
- `tools/m23_flatten_preview.py` G-F2 压平算术 + `--teeth` 四组合成表牙齿 + 乙腿名单。
- `tools/m23_stage_cleanroom.py` 净室装配（623 件逐字节复制、导入前产物计数断言、闭包断言，幂等）。
- `tools/m23_score.py` 硬盘侧独立复算 + G-F0..G-F6 打分；`tools/m23_pkg.py` 证据装配；
  `tools/m23_write_readme.py` 从字节生成 README；`tools/m23_make_zip.py` 成品包 v2（Z1..Z4 四条不变量）。
- 运行时修复件 = `Assets/FengChuanXiangMoc3/Runtime/CubismFadeMotionListCompactor.cs`（随 v2 包交付）；
  探针 = `Assets/Editor/AienMoc3ParityProbe.cs` 的 `RunM23`（扩件，复用 M22 实测跑通的 PlayMode 与像紆尺）。
- 本笔入库五件此刻 HEAD 无 blob ⇒ FILES 登记由回执笔兑现；链接清单 `tools/m23_delivery_links.txt`
  自入库即冻结（复跑写新文件名）。门槛 B 仍 0/5。


## M23 回执与 V2 审核通过（2026-10-01）

用户裁决原文：「先把这笔补掉，再开4包，V2我审核通过」。V2 审核断点已通过，回执后推进高松灯、绘里、立希、长崎素世四包；清理仍单独候保留/删除清单。

收口提交 `700dbd7` 已核实。回填挂点为 `map.md:725`、`issues/05-defect-register.md:652`、`记录_实时moc3线.md:432`；登记义务挂点为 `issues/25-m23-fade-hole-fix.md:165`、`tools/README.md:430`。此前引用的 map:647、issues/05:613、记录:404 属 M22 还账挂点，不作为 M23 收口挂点。

FILES 实际新增七项：四个 Python 工具（m23_score、m23_pkg、m23_write_readme、m23_make_zip）、m23_delivery_links.txt、M23 证据 README 共六件，以及遗漏的 issues/25 票面。七项均已有 HEAD blob；六件交付件本笔不改字节，票面只追加本节。旧文“五件工具 + 链接 + README”的计数以本节为准。

证据边界补充：397 段的 267/130/0 是磁盘绑定分类，实播为旧实例压平后的八个修复样本及八个回归样本，不能等同 397 段全部实播或全部混合验证。净室一次导入直接证明本次新目录模型级表为 397/0；未做同一新目录受控重复导入实验，故“反复导入是唯一原因”尚未得到因果验证。当前 YAML 的 fileID:0 只证明当前空槽，不独立证明其生成历史。修复组物理极差范围为 0.271834–0.592688；0.202–0.593 是含回归组的口径。上述说明不改变冻结判据或已发表读数。

继续沿用全新目录、首次导入后检查模型级表的操作规则。门槛 B 仍 0/5，K2 维持红，论文不动；本回执不启动 Unity。回执自身哈希由下一票记录。


## M24 四包输入阶段（2026-10-01）

用户已审核爱音 V2 通过并授权「先把这笔补掉，再开4包」。M23 回执提交 `76d0108` 由本节回填（挂点为各台账 M23 回执节末句）；新票 `issues/26-m24-four-pack-pipeline.md`。

盘点实测为三 RAR、一 ZIP，纠正旧文“两件 RAR”。系统 bsdtar 3.8.4 成功解包；高松灯四套、绘里一套、立希一套、素世四套分件加嵌套原件一套，共十一套模型入口，声明动作引用 512、表情引用 324（不是去重文件数）。绘里 physics.json 引用缺失，独立副本按包内同名 physics.json0 原始字节接通，修正映射与摘要保留。原归档未改，多模型不冒充已装配为单角色。

已完成输入副本、逐引用复制、moc 提取与归一，独立复算 `MISMATCHES 0`；机器读数入 `evidence/m24-four-pack-intake-20261001/`。工装为 m24_intake.py、m24_prepare_sources.py、m24_extract_rigs.py、m24_verify_sources.py；新票与四件工具本笔才入库，后续登记 FILES。当前尚未导出 moc3、尚未启动 Unity、尚无四包成品或验收网页。下一阶段为逐模型导出与边车转换；Unity 腿仍按跑前报备和真人放行边界。门槛 B 仍 0/5、K2 红、论文不动，清理未执行。

## M24 转换阶段工装（2026-10-01，本笔新增 15 件）

`m24_export_rig.py`（Umamo legacy-import 单套封装，JDK 21，`--attempt` 独立输出目录）、
`m24_uvflip.py`（冻结 `m12_e3_uvflip` 的报告目录后继包装，不改原件）、
`m24_verify_geometry.py`（逐套 ID/范围/UV 字节的三阶段核验 raw|groupfix|uvfix）、
`m24_sidecars.py`（动作+表情转换，未知 ID 排除并逐件登记，尾逗号内存适配器）、
`m24_sidecars_test.py`（3 项：真实绘里样本、四种尾逗号形态、非法字段仍拒）、
`m24_physics.py`（冻结 m08 翻译器的 11 套批量前置校验与复算，辅助测量保持 null）、
`m24_assemble.py`（`candidate-01` 装配与引用闭包、组序/表情名序对账）、
`m24_uv_convention_census.py`、`m24_uv_vertex_census.py`（两把被否的覆盖率尺，留档）、
`m24_uv_ink_consumption.py`（墨迹消耗率判据）、
`m24_publish_disk_evidence.py`（逐件复算 + 判据断言 + 作废件改名；`-O` 直接拒跑）、
`m24_raster_evidence.py`（离线默认姿势光栅面板与输入哈希清单）、
`m24_evidence_page.py`（证据页 `page.html`）、
`m24_pkg.py`（交付 manifest 装配，逐件复算哈希并复核墨迹判据与 summary 一致）、
`m24_write_disk_readme.py`（证据 README 生成器，表格数值全部取自 summary 与墨迹普查）。

两处工具级更正：`m24_verify_geometry` 首版把"核心回读"当成文件字节比对，比对自身镜像恒真；
`m24_publish_disk_evidence` 首版只信 `uvfix_verify.json` 的 `size_delta`（该字段按定义恒 0），现改测两件实际长度。
冻结的 `m22_measure_distortions.py` 不动，其 `cov_*` 标注为灰度、退出码不可信。


### 立希 Unity 实测与 fix4 导出新增工具（2026-10-02）

- `tools/stage_taki_unity_validation.py`：把批准包内 model/ 解到唯一新目录并生成 43 段动作计划，逐文件写后回读比对，目录已存在即拒绝。
- `tools/taki_unity_evidence.py`：按清单把运行截图内联成自包含证据页，落盘后重新解析 HTML、逐图解码并复算 SHA-256。
- `tools/taki_unity_page_check.py`：headless Edge 打开交付页，断言 DOM 图数、截图非空与尺寸，落 browser-check.json。
- `tools/taki_full_score.py`：零 Unity 复算完整播放报告，独立解码 87 张 PNG、算背景差分、核对身份顺序与时长/帧数，写 validation.json。
- `tools/taki_make_zip3.py`：带 Unity 实测断言（P6：包内 model 文件等于 Unity 加载字节）的后继打包器，已被 fix4 取代。
- `tools/taki_make_zip4.py`：修正 README 清单行数口径，并新增包内回读牙齿：声明行数须等于实际行数、入口路径须出现。
- `tools/taki_fix3_replace.py`、`tools/taki_fix4_replace.py`：先备份并复算原源件哈希再替换根目录 立希.rar，替换后从该文件回读 ZIP 与逐条目哈希。
- `Assets/Editor/TakiFullPlaybackProbe.cs`：立希完整时长播放探针（真 PlayMode、逐段结束回调与参数区间、增量落盘、首段失败即停并保留已测读数）。
- `Assets/Editor/TakiMoc3UnityProbe.cs`：第一趟启动诊断件，仍含爱音净室阶段与压平组件，只作诊断现场，不作验收判据。


## moc2 → moc3 工作流 V2（2026-10-05，流萤猫案例后继）

- 主线规范：`../moc2-to-moc3-workflow-v2.md`。按 G0 源盘点与身份冻结、G1 可信源与多入口归属、G2 源语义与目标侧车、G3 单角色合并与版本适配、G4 runtime 冻结、G5 Unity 净室、G6 真 PlayMode、G7 确定性打包回读、G8 网页与交付检查执行；每阶段都有前置条件、动作、完成条件和证据出口。
- 案例与失败路径：`../moc2-to-moc3-pinhao-lessons-20261005.md`。记录连字符 MTN ID、未知曲线、目标 clamp、reference lowering、配件定位、UV/顺序/遮罩、Unity staging、控制器接线、ZIP 回读和证据页检查的真实故障与边界。
- 复现入口：先读取两份 V2 文档和本轮输入清单，再按阶段选择现役工具；没有一个已验证的跨角色一键编排器。Pinhao 工具中的输入、输出、日期、数量和本机依赖先核对，不能直接重跑以覆盖历史产物。
- 关键案例结果仅作出口样例：源审计 66 参数/18 parts，未知动作曲线 5,874；侧车虚拟参数 91、缺失 Part 5；Unity `664/232/5/37`（动作/表情/姿势/PNG），runtime 904 个文件；交付入口引用 903、ZIP 907 条目、manifest 906 行。〔出口：`tmp/pinhao-source-audit-20261003-v3/audit.json`、`tmp/pinhao-sidecars-20261005-v2/audit.json`、`tmp/pinhao-unity-project-20261005-v4/tmp/pinhao-unity-20261005-123928-122-03a7e0e8c8c34a358225975f77ab0ae2/report.json`、`delivery/pinhao-unity-20261005-v1/delivery-report.json`〕
- 证据边界：Unity passed 不等于源/目标动态等价；physics on/off 只证明 Unity 因果响应；同下肢可见边界不证明源缺少鞋或小腿几何；像素、顶点、条目、引用和字节数不能混用。旧 M20 工作流与已交付 ZIP 保持冻结，新行为需新后继工具和新输出目录。
