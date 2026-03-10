# 使用 Open-AutoGLM 进行 APP 性能测试方案

## 1. 项目能力分析（为什么可用于性能测试）

Open-AutoGLM 的 Phone Agent 本质是“视觉感知 + 决策 + 设备控制”的自动化执行框架，可稳定复现真实用户在手机上的操作路径，适合做端到端性能测试。

- **跨设备控制层**：已支持 Android（ADB）、HarmonyOS（HDC）和 iOS（XCTest/WDA），可覆盖多终端回归场景。
- **可编排执行层**：`PhoneAgent.run(task)` 支持基于自然语言任务自动执行；`step()` 可按步驱动，适合性能采样点插桩。
- **环境可检验**：CLI 内置系统检查（工具安装、设备连接、输入法/WDA 可用性），降低压测前环境不一致导致的噪声。
- **模型调用可观测**：部署检查脚本能够输出 token 使用量，可用作“模型推理成本”与“端上操作耗时”的分离指标。

> 结论：该项目不直接提供 FPS/CPU/内存分析器，但非常适合作为“稳定自动化流量发生器 + 场景执行器”，结合 adb/hdc/xctrace 等系统工具可构建完整性能测试体系。

## 2. 测试目标与指标体系

建议将指标拆成 3 层，便于定位瓶颈：

### A. 用户体验层（E2E）

- 任务总耗时（Task Duration）
- 关键路径耗时（如“启动 App → 首屏可交互”）
- 操作成功率（任务成功完成比例）
- 重试次数 / 卡死率 / 超时率

### B. 设备资源层（端上）

- CPU 占用（App 进程、系统总占用）
- 内存（PSS/RSS、峰值内存）
- 帧率与卡顿（Jank、掉帧数）
- 电量消耗、温度、流量（可选）

### C. 智能体与模型层

- 单步推理耗时（模型响应时间）
- 每任务 step 数
- Token 消耗（prompt/completion/total）
- 感知失败率（截图异常、元素误判导致的无效动作）

## 3. 典型测试场景设计

按“真实业务链路”设计场景，建议每个场景至少 30 次重复：

1. **冷启动场景**：从桌面打开目标 App，进入首页并完成一次搜索。
2. **热启动场景**：后台恢复目标 App，打开详情页。
3. **复杂交互场景**：搜索 → 筛选 → 详情 → 返回列表（多页面跳转）。
4. **弱网场景**：限速/高延迟网络下重复关键路径。
5. **长时稳定性场景**：持续 30~60 分钟执行循环任务，观察内存与成功率漂移。

## 4. 测试实施架构

推荐“四段式”流水线：

1. **驱动层（Open-AutoGLM）**：负责跨平台自动化操作与场景执行。
2. **采集层（系统工具）**：
   - Android：`adb shell dumpsys meminfo`、`top`、`gfxinfo`、`batterystats`
   - HarmonyOS：`hdc shell` 下对应性能命令
   - iOS：`xctrace` / Instruments / WDA 状态接口
3. **聚合层（日志/指标）**：统一写入 CSV/JSON（按 run_id、step_id、timestamp 对齐）。
4. **分析层（报告）**：输出 P50/P90/P95、成功率、趋势图与回归结论。

## 5. 基于本项目的落地步骤

### 步骤 1：准备并校验环境

- 安装依赖并连接设备。
- 先执行项目自带检查脚本，确保模型可用：

```bash
python scripts/check_deployment_cn.py --base-url http://localhost:8000/v1 --model autoglm-phone-9b
```

### 步骤 2：定义标准化任务模板

将任务语句固定，避免提示词漂移影响对比，例如：

- `打开抖音并搜索“露营”`
- `打开电商 App，搜索“蓝牙耳机”，进入第一个商品详情页`

每条任务保持：
- 同一设备型号
- 同一系统版本
- 同一网络条件
- 同一模型参数（temperature/top_p 等）

### 步骤 3：执行前后采样

在每次任务前后采集资源数据：

- 前采样：启动前 3~5 秒 CPU/内存基线
- 任务中：每秒采样一次（或关键步骤采样）
- 后采样：结束后持续 5~10 秒，观察资源回落

### 步骤 4：记录智能体过程指标

建议记录：
- run_id、task_name、start_ts、end_ts、duration_ms
- step_count、finish_reason、success
- model_latency_ms（每 step）
- token_usage（若模型服务返回 usage）

### 步骤 5：统计与判定

建议门禁（可按业务调整）：

- 成功率 ≥ 95%
- P95 任务耗时不高于基线 + 10%
- 峰值内存不高于基线 + 15%
- 卡顿指标（如 Jank）不高于基线 + 10%

## 6. 建议的数据结构（示例）

```json
{
  "run_id": "2026-03-10-android-mi13-001",
  "task": "打开电商App并搜索蓝牙耳机",
  "success": true,
  "duration_ms": 18420,
  "step_count": 11,
  "model": {
    "avg_latency_ms": 920,
    "prompt_tokens": 5420,
    "completion_tokens": 880
  },
  "device": {
    "cpu_avg": 41.2,
    "mem_pss_mb_peak": 612,
    "jank_count": 17
  }
}
```

## 7. 风险与优化建议

- **模型波动**：尽量固定模型版本与参数；评估时使用 P95 而非单次结果。
- **环境噪声**：测试机关闭自动更新/通知；全程飞行模式+指定 WiFi（若测弱网则统一网络仿真策略）。
- **场景漂移**：App UI 改版可能影响智能体动作路径；每周回放一次“基准场景集”。
- **可解释性不足**：保留 step 级日志（动作、截图、耗时），便于回归定位。

## 8. 推荐执行节奏

- **日常构建**：冒烟性能（3~5 个关键场景，快速门禁）
- **每晚构建**：全量回归（20+ 场景，多机型）
- **版本发布前**：长稳 + 弱网 + 低电量专项

---

如果你愿意，我可以下一步基于你当前目标 APP（如抖音/小红书/淘宝）直接给出一份可执行的“场景清单 + 指标阈值 + 采集命令模板（Android/HarmonyOS/iOS）”。

## 9. 理想汽车 APP 可执行性能测试清单（Android/HarmonyOS/iOS）

> 适用范围：理想汽车 APP 的「启动、社区/内容浏览、车控页面、充电地图、账号相关」核心路径。

### 9.1 测试前参数约定（统一口径）

- 重复次数：每个场景 **30 次**（冒烟可先 10 次）
- 统计口径：重点看 **P50/P95**，同时看失败率
- 设备条件：
  - 前台仅保留理想汽车 APP
  - 关闭系统自动更新、消息推送弹窗
  - 屏幕亮度固定（如 60%）
  - 电量 > 50%，温度稳定
- 网络条件：
  - 基线：WiFi 稳定网络
  - 弱网：带宽限制 + 人为引入时延（如果实验室支持）

### 9.2 场景清单（可直接用于 Phone Agent）

| 场景ID | 场景名称 | Phone Agent 任务示例（自然语言） | 成功判定 |
|---|---|---|---|
| LXP-01 | 冷启动到首页可交互 | `从桌面打开理想汽车APP，等待首页加载完成并可点击任意底部Tab` | 30s 内进入首页且可完成一次点击 |
| LXP-02 | 热启动恢复 | `将理想汽车APP切到后台5秒后恢复到前台，确认页面可继续操作` | 10s 内恢复且无白屏/卡死 |
| LXP-03 | 社区浏览链路 | `打开理想汽车APP社区，进入第一条内容，停留3秒后返回列表` | 可进入详情并返回成功 |
| LXP-04 | 充电地图查询 | `打开理想汽车APP并进入充电地图，搜索“北京西二旗”并打开首个结果` | 地图与搜索结果正常展示 |
| LXP-05 | 车控页加载（有车主账号） | `打开理想汽车APP进入车控页面，执行一次空调开关模拟操作（不提交危险操作）` | 车控页可加载，操作指令有明确反馈 |
| LXP-06 | 长链路稳定性 | `循环执行：首页->社区->返回->充电地图->返回，共循环20次` | 成功率 ≥ 阈值，无明显内存爬升 |

> 说明：LXP-05 涉及敏感操作，建议仅做“可点击到确认页/模拟操作”，避免真实车辆控制风险。

### 9.3 指标阈值（建议初始门禁）

#### A. 用户体验（E2E）

- LXP-01 冷启动：首页可交互 **P95 ≤ 8s**
- LXP-02 热启动：恢复可交互 **P95 ≤ 3s**
- LXP-03/LXP-04 页面跳转：单次跳转 **P95 ≤ 4s**
- 场景成功率：**≥ 95%**
- 超时率：**≤ 3%**

#### B. 资源指标（端上）

- CPU（App 进程平均）：**≤ 45%**（按机型可调整）
- 峰值内存（PSS）：较基线版本 **增幅 ≤ 15%**
- Jank（Android/Harmony）：较基线版本 **增幅 ≤ 10%**
- 长链路（LXP-06）20 轮后内存回落：末轮 PSS 不高于首轮 **+20%**

#### C. 智能体指标

- 每任务 step 数漂移：相对基线 **≤ +20%**
- 模型单步响应时延 P95：相对基线 **≤ +15%**
- 感知失败（截图/识别导致无效动作）占比：**≤ 5%**

### 9.4 采集命令模板

> 先替换变量：
> - Android 包名：`PKG=com.lixiang.auto`（若不一致请先探测）
> - iOS Bundle ID：`BUNDLE_ID=<your.bundle.id>`

#### 9.4.1 Android（ADB）

**1）包名探测（若不确定）**

```bash
adb shell pm list packages | grep -i lixiang
```

**2）启动耗时（冷启动）**

```bash
PKG=com.lixiang.auto
ACT=$(adb shell cmd package resolve-activity --brief $PKG | tail -n 1)
adb shell am force-stop $PKG
adb shell am start -W -n $ACT
```

**3）CPU/内存按秒采样（任务执行期间）**

```bash
PKG=com.lixiang.auto
for i in $(seq 1 60); do
  TS=$(date +%s)
  adb shell top -b -n 1 | grep $PKG | head -n 1 | sed "s/^/$TS,CPU,/"
  adb shell dumpsys meminfo $PKG | grep -E "TOTAL PSS|TOTAL RSS" | sed "s/^/$TS,MEM,/"
  sleep 1
done
```

**4）帧率/卡顿（gfxinfo）**

```bash
PKG=com.lixiang.auto
adb shell dumpsys gfxinfo $PKG reset
# 执行你的场景后
adb shell dumpsys gfxinfo $PKG > gfxinfo_${PKG}.txt
```

**5）电量（可选）**

```bash
adb shell dumpsys batterystats --reset
# 执行场景后
adb shell dumpsys batterystats > batterystats_${PKG}.txt
```

#### 9.4.2 HarmonyOS（HDC）

> HarmonyOS 各版本性能命令存在差异，以下提供通用模板，按设备实际命令做微调。

**1）应用进程确认**

```bash
hdc shell "ps -A | grep -i lixiang"
```

**2）CPU/内存采样模板**

```bash
for i in $(seq 1 60); do
  TS=$(date +%s)
  hdc shell "top -n 1 | grep -i lixiang | head -n 1" | sed "s/^/$TS,CPU,/"
  hdc shell "hidumper -s memory" | head -n 20 | sed "s/^/$TS,MEM,/"
  sleep 1
done
```

**3）图形/渲染相关（按系统支持）**

```bash
hdc shell "hidumper -s RenderService" > render_service.txt
```

#### 9.4.3 iOS（XCTest/WDA + xctrace）

**1）确认设备与 Bundle ID**

```bash
xcrun xctrace list devices
# Bundle ID 可通过 Xcode 或 ideviceinstaller 查询
```

**2）使用 xctrace 录制性能（Time Profiler 示例）**

```bash
BUNDLE_ID=<your.bundle.id>
xcrun xctrace record \
  --device "<Your iPhone Name>" \
  --template "Time Profiler" \
  --output lixiang_time_profile.trace \
  --time-limit 60s \
  --launch -- $BUNDLE_ID
```

**3）内存模板录制（Allocations 示例）**

```bash
BUNDLE_ID=<your.bundle.id>
xcrun xctrace record \
  --device "<Your iPhone Name>" \
  --template "Allocations" \
  --output lixiang_alloc.trace \
  --time-limit 60s \
  --launch -- $BUNDLE_ID
```

### 9.5 一次完整执行建议（最小可用）

1. 先跑 LXP-01、LXP-02、LXP-03（每个 10 次）做冒烟基线。
2. Android/Harmony 先采 CPU + 内存；iOS 先采 Time Profiler + Allocations。
3. 将每次 run 输出统一到 `run_id` 粒度（CSV/JSON）。
4. 对比上一个稳定版本，输出 P50/P95 与失败率变化。
5. 若阈值越线，回放 step 日志 + 截图定位具体页面。
