# 流萤猫导出失败路径与复用边界

更新：2026-10-05。配套入口：[moc2 → moc3 工作流 V2](moc2-to-moc3-workflow-v2.md)。这里保存案例、故障分支和工具能力边界；新角色的数量、坐标和环境从自己的源清单计算。

## 已确认故障与前置闸门

| 触发 / 症状 | 定位依据与处理 | 新版闸门 / 使用边界 |
| --- | --- | --- |
| 三个入口被当作三个独立角色，或按文件夹名猜附件 | 源归档与唯一 MOC 分组、drawable 和父 deformer 追踪，见 `tools/pinhao_source_audit_v2.py`、`tools/pinhao_ornament_trace_v1.py` | G0/G1：入口、唯一 rig 与装配实例分开记；配件必须有运行时和几何映射 |
| MTN 中含连字符的合法参数漏解析 | 实际 parser 和回归测试见 `../../Tools/Live2DReconstruction/src/live2d_reconstruction/motion.py`、`../../Tools/Live2DReconstruction/tests/test_motion.py` | G0：逐曲线身份核对；`PARAM_ARM_R_01_001-1` 是案例，不限制其他合法 ID |
| 未知参数被删除、猜别名，或把未绑定曲线当已生效 | 官方 Core 语义实验见 `tools/pinhao_source_runtime_probe_v3.mjs`；侧车保留见 `tools/pinhao_sidecars_v1.py` | G2：虚拟参数与无效 Part 按源实测语义区分；原名与源记录保留，不自动宣称视觉效果 |
| Unity 参数超出声明范围，源运行时却能播放 | `tools/pinhao_sidecars_v1.py` 的 `clamp_motion_for_target` 和 `samples_clamped`；回归见 `tools/test_pinhao_sidecars_v1.py` | G2：源记录原样保留，目标播放单独 clamp，全量回读。通用 `to_motion3` 不隐式改变其他消费者语义 |
| reference bake 后乘法与双面标志不符 | 实际编译入口 `../../Tools/Umamo/app/cli/src/jvmMain/kotlin/org/umamo/cli/LegacyImportCommand.kt` 按 reference 版本 lowering；回归 `../../Tools/Umamo/app/cli/src/jvmTest/kotlin/org/umamo/cli/LegacyImportBlendTest.kt` | G3：读取 reference 实际版本并核对 legacy/extended blend 表；不要编辑同名非编译副本 |
| 头饰显示但位置不对 | `tools/pinhao_ornament_trace_v1.py`、`tools/pinhao_merge_rig_v1.py` 与 `tools/pinhao_geometry_verify_v1.py` 的独立变换检查 | G1/G3：可见与位置正确分开；warp-local 修正需父级变换和逆变换证据，案例偏移不迁移 |
| 几何报告通过，预览仍错层、白块或 UV 错位 | 渲染层需按全局 order、mask、blend、纹理 alpha 逐项检查；独立 UV 字节读取见 `tools/pinhao_geometry_verify_v1.py` | G3：先可信渲染与数据分层归因；E2/E3 按实测缺陷执行，不固定对所有文件翻转 |
| Gradle 无法启动、改错 Kotlin 文件 | 实际编译源码是 `Tools/Umamo/app/cli/...`；本案用完整 JDK 21 | G3/G5：核对 JAVA_HOME、实际 build target 与依赖，Cubism 附带 JRE 不等于可构建 JDK |
| staging 编译失败，尚未进入 PlayMode | 主探针存在未定义处理函数的孤立事件订阅；`tools/pinhao_unity_stage_v4.py` 在 staging 副本移除并断言 | G5：先验证生成后的副本编译；该修正只适用于案例订阅，不能泛化为删除所有事件 |
| Unity 启动索引异常，或启动后过早退出 | v4 staging 使用有效 Search.settings 关闭 startup indexing；异步探针由自身结束 | G5/G6：记录环境 workaround；图形验证保留 graphics，异步流程不加提前退出的 `-quit` |
| 控制器静默、姿势图相同或参数被还原 | `../../Assets/Editor/PinhaoCompletePlaybackProbe.cs` 的 `NewModel` 核对模型绑定，补组件后 Refresh；参数 store 与执行序明确 | G5/G6：证明控制器已接入模型；记录探针添加/禁用组件；原始 prefab 自动接线另测 |
| 动作资源存在但 fade 表无法匹配 | 探针 `Inventory` 按实例 ID 的实际首个匹配槽检查；历史普查见 `tools/m23_fade_list_census.py` | G5：先查 clip/事件/全部 fade 表的空槽与重复 ID，再开始动作腿；清理缓存不是因果证据 |
| Windows 临时 ZIP 被另一句柄锁定 | `tools/pinhao_unity_delivery_v1.py` 的 `zip_bytes`、`verify_zip` 使用 TemporaryDirectory 内路径 | G7：关闭写句柄后再读取；双次构建与最终磁盘回读分别取证 |
| 浏览器实际已加载，内置截图却无可见视口 | 当时内置截图报告 `NATIVE_BROWSER_VIEWPORT_UNAVAILABLE`；独立 headless Edge 产生新 PNG 后执行项目识图 | G8：检查实际截图文件与尺寸；先普通 headless，兼容参数按失败证据选用，保留独立 profile |

## 交付审查发现的证据缺口

这些条目限制结论范围，不改写历史报告，也不将本次文档更新描述为重新验证模型。

| 现有表述 / 检查 | 当前能支撑的事实 | 后继要求 |
| --- | --- | --- |
| README/HTML 写“源不存在鞋或下方小腿几何” | 源与 Unity 默认图在相同下肢位置结束。探针 `BuildCamera` 使用全部 Core drawable 的 world bounds 加边距，但这仍不证明源 drawable 缺失 | G1：补源 drawable 几何、opacity、纹理、遮罩、装配与取景检查后再下因果结论；当前写“可见边界一致，具体成因未定” |
| README 将 `pixel_changed` 数量叫作“changed pixels or vertices” | 生成器只统计 `pixel_changed` 布尔字段 | 像素变化、顶点变化及二者并集分别计算和命名；211 是案例像素响应条目数 |
| “Source candidate status: not provided … package status remains passed” | Unity 报告有自己的 passed 状态；缺失源候选状态仍是缺失 | G4/G7：候选、运行测试和包核验各有字段，不借用另一个阶段的状态填空 |
| 负例拒绝数全对，所以越界检查已证明 | 本案的 root 外负例指向不存在的文件；旧 gate 会先因文件不存在拒绝 | G8：用实际存在的 root 外文件，逐项核对拒绝原因；旧 3/3 结果不升级为原因覆盖 |
| 内存构建副本 verify 后即发布 | 工具验证临时 ZIP，然后写最终 ZIP；本次另做最终磁盘 ZIP 独立回读补足 | G7：后继工具把最终磁盘回读作为发布流程的必需阶段 |
| 双次压缩通过等于下一模型也能直接复用 | 现工具固定入口、路径、数量和日期，允许覆盖自己的交付目录；构造器压缩级别也需核实逐条 ZipInfo 的实际应用 | 后继打包器参数化、dry-run、拒覆盖、明确逐条压缩设置；本次仅定义要求，没有声称这些功能已实现 |
| 浏览器 OCR 把 pose 读成 3/3 | 原始报告与 DOM 为 5/5 | 核验原始结构化来源；识图用于画面检查，不替代数值出口 |
| 目录 `stat().st_size` 当 mirror 字节 | 目录大小不是目录下全部文件字节和 | 镜像字节由必需文件逐件求和，并独立比对文件集合与哈希 |
| 净室一次导入正确，因此旧缓存的唯一原因已确定 | 本轮新工程派生资源正确，只证明该受控路径可用 | 将重复导入/缓存影响作为待实验证实的原因；保留净室操作边界 |
| 全部动作 passed，因此所有动作视觉保真、切换也通过 | 探针逐动作检查完整播放、回调、范围与有限值；图像只取计划代表样本 | 全量播放、视觉采样、源目标时序等价、动作切换分别记覆盖；新增切换需求另做双段测试 |
| Physics on/off 有响应，因此还原源物理 | 本案证明 Unity 内 baseline/on/off 因果响应 | 源轨迹幅度、相位、偏置和释放行为未比较时，源物理等价保持未验证 |

现有交付措辞的强断言位置：`tools/pinhao_unity_delivery_v1.py` 的 `make_readme`、`make_evidence_page`。旧版交付保持冻结；未来重发应生成新版本并重跑包、页和镜像受影响检查。

## 现役工具怎么复用

下列路径相对仓库根目录；运行前读取当前脚本接口并检查输入输出，不能按文件名推断通用性。

| 环节 | 当前工具 / 来源 | 能力边界 |
| --- | --- | --- |
| 源审计 | `.scratch/moc3-showcase/tools/pinhao_source_audit_v2.py` | 当前对象与输出路径绑定；新角色需适配并独立核对源集合 |
| 源语义 | `.scratch/moc3-showcase/tools/pinhao_source_runtime_probe_v3.mjs` | 源 Core 行为实验；不替代完整绘制、物理或 Unity 验证 |
| 部件与单角色 | `pinhao_ornament_trace_v1.py`、`pinhao_merge_rig_v1.py` | 脚本位于同一 tools 目录；案例纹理页、drawable 和配件变换专用 |
| 几何核验 | `pinhao_geometry_verify_v1.py` | 提供 `--moc` 和 `--out`，内部仍限制案例画布和源参考路径；不是任意角色验证器 |
| 边车转换 | `pinhao_sidecars_v1.py` | 有 `--source`、`--rig`、`--runtime-evidence`、`--output`、`--verify`；当前语义来自本案，未知 ID 处理依赖证据 |
| 格式转换 | `Tools/Umamo/app/cli/.../LegacyImportCommand.kt` | `legacy-import` 接受 rig/out、texture、manifest、reference；真实接口见源码，不搬同名副本 |
| runtime 组装 | `pinhao_runtime_assemble_v3.py` | 已验证案例的闭包与哈希；路径及输出仍硬编码 |
| Unity staging | `pinhao_unity_stage_v4.py` | 拷贝 SDK 与资源并逐件核哈希；依赖 sibling worktree 的 Search.settings/InputSystem 和本机 Newtonsoft PackageCache |
| Unity 探针 | `Assets/Editor/PinhaoCompletePlaybackProbe.cs` | 添加 Motion/Update/Store 等组件并禁用自动驱动；验证探针配置后的运行载体，主文件需 staging 修正孤立订阅 |
| 模型 ZIP / 页面 | `pinhao_unity_delivery_v1.py` | 确定性双构建、临时包回读；没有 dry-run，输出可覆盖，数量/入口/标题硬编码 |
| 历史交付预检 | `delivery_preflight.py`、`m17_delivery_link_gate.py` | 冻结项目台账和旧链接格式检查；固定写 `tmp/delivery_preflight.txt` 与 `tmp/preflight_negative_gate.txt`，正例复杂路径和拒绝原因覆盖另核验 |

上表未提供新角色的一键命令。沿用工具时先生成本轮配置/后继适配器、检查接口，再运行本轮输入；不要直接重跑会覆盖历史证据的案例命令。

## 可安全复算的测试入口

以下命令在仓库根目录执行，仅验证对应代码边界；测试通过不提升模型外观、Unity 或交付状态。数量由当次结果报告，不固定案例通过数。

```sh
python -m pytest .scratch/moc3-showcase/tools/test_pinhao_sidecars_v1.py
python -m pytest Tools/Live2DReconstruction/tests/test_motion.py
```

`test_motion.py` 使用 pytest 顶层测试函数；不要用 unittest discovery 运行它，因为该入口会报告 0 个测试而不构成验证。Kotlin 回归使用真实 Umamo build target 与完整 JDK；当前环境的构建命令从 Gradle 配置核对。

## 本版落实范围

本次落实可复用操作规范、失败分支和证据边界，以及工具索引入口。自动编排、跨角色参数化、通用 dry-run、发布事务、精细链接原因校验和原始 prefab 自动接线验证属于后继实现要求，当前工具尚未统一提供。此文不改变旧 M20 ZIP、流萤猫交付 ZIP、模型字节、SDK、论文或用户验收状态。
