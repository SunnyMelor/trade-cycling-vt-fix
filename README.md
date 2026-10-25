# Trade Cycling × Visible Traders 兼容性修复（补丁）

**Minecraft 1.21.1 / Fabric** — 修复「刷新交易后未解锁交易消失」的问题

> ⚠️ **本仓库不含 Trade Cycling 的源代码。**
> 上游 `Trade Cycling` 为 **All Rights Reserved**，本仓库仅提供**补丁文件**，需自行从官方仓库获取源码后应用。
> 版权归原作者 **Max Henkel** 所有。

---

## 1. 上游项目

| 模组 | 作者 | 源仓库 | 许可证 |
| --- | --- | --- | --- |
| **Trade Cycling** | Max Henkel | https://github.com/henkelmax/trade-cycling | All Rights Reserved |
| **Visible Traders** | Ramixin | https://github.com/Ramixin/VisibleTraders | MIT |

- Trade Cycling 下载：https://modrinth.com/mod/trade-cycling
- Visible Traders 下载：https://modrinth.com/mod/visible-traders

补丁基线：

- `henkelmax/trade-cycling` 分支 **`1.21.1`**，版本 `1.21.1-1.0.23`
- `Visible Traders` **`1.3.2`**

---

## 2. 问题现象

同时安装两个模组后：

1. 打开村民交易界面，未解锁交易可正常预览
2. 点击「刷新交易」后，**未解锁交易全部消失**
3. 必须关闭界面再重开，未解锁交易才重新出现

## 3. 根因

Trade Cycling 通过反射桥接调用 Visible Traders 的内部方法，但调用的是**已被重命名的方法**：

| Trade Cycling 期望 | Visible Traders 1.3.2 实际 |
| --- | --- |
| `visibleTraders$forceTradeGeneration()` | `visibleTrades$regenerateTrades()` |
| `visibleTraders$wrapAndSendMerchantOffers(...)` | `visibleTraders$getCombinedOffers()` + `visibleTraders$getShiftedLevel()` |

运行时日志：

```
Failed to initialize visible traders integration
java.lang.NoSuchMethodException: net.minecraft.class_1646.visibleTraders$forceTradeGeneration()
    at de.maxhenkel.tradecycling.compatibility.VisibleTraders.getRegenerateMethod(VisibleTraders.java:55)

java.lang.NoSuchMethodException: net.minecraft.class_3222.visibleTraders$wrapAndSendMerchantOffers(...)
    at de.maxhenkel.tradecycling.compatibility.VisibleTraders.getSendMerchantOffersMethod(VisibleTraders.java:66)
```

`init()` 捕获异常后把两个 `Method` 字段置为 `null`，导致刷新时：

- 「重新生成锁定交易」变成**空操作**
- 「带上锁定交易发送」失败，**退化为只发送普通交易**

客户端因此只更新了已解锁交易，Visible Traders 的合并列表仍是旧快照，
直到重新打开界面（触发 `openTradingScreen` 钩子）才刷新。

详细分析见 [docs/root-cause.md](./docs/root-cause.md)。

---

## 4. 补丁内容

`patches/0001-visibletraders-compat.patch` — 12 个文件，+172 / −96

| 文件 | 改动 |
| --- | --- |
| `compatibility/VisibleTraders.java` | 按名称列表探测新旧方法名；发送时使用合并列表与位移等级；保留旧版回退 |
| `TradeCyclingMod.java` | `sendOffers` 统一走桥接 |
| `gui/CycleTradesButton.java` | 悬浮提示显示当前快捷键 |
| `assets/trade_cycling/lang/*.json` | 新增提示文案条目（9 种语言） |

核心逻辑：

```java
regenerateTrades = findVillagerMethod("visibleTrades$regenerateTrades",
                                      "visibleTraders$forceTradeGeneration");
getCombinedOffers = findVillagerMethod("visibleTraders$getCombinedOffers");
getShiftedLevel  = findVillagerMethod("visibleTraders$getShiftedLevel");
```

这样刷新时下发的包与「打开界面」时完全一致，锁定交易立即随包到达客户端。

---

## 5. 使用方法

```bash
git clone https://github.com/henkelmax/trade-cycling.git
cd trade-cycling
git checkout 1.21.1
git apply /path/to/0001-visibletraders-compat.patch
```

然后构建（需 **JDK 21**）：

```bash
./gradlew :fabric:build
```

产物：

```
fabric/build/libs/trade-cycling-fabric-1.21.1-1.0.23.jar
```

> 若 `git apply` 因上游更新而失败，可改用 `git apply --3way` 或
> `patch -p1 < 0001-visibletraders-compat.patch` 手动处理冲突。

---

## 6. 已构建制品

`dist/trade-cycling-fabric-1.21.1-1.0.23.jar`

| 项 | 值 |
| --- | --- |
| 大小 | 522,250 B |
| SHA-256 | `5EB197AD6C0EF5052E3842EA41DD493C4E11FE5641DFA1BFF8A257EA038807E1` |

已验证：从上游源码应用本补丁后重新构建，产物与该文件**逐字节一致**。

直接安装：将 jar 与 Fabric API 放入 `mods/`（单人游戏即可；多人游戏服务端也需放置）。

---

## 7. 运行环境

| 项 | 要求 |
| --- | --- |
| Minecraft | 1.21.1 |
| Fabric Loader | ≥ 0.15.11 |
| Fabric API | ≥ 0.102.0+1.21.1 |
| Visible Traders（可选） | 1.3.2 |

### 验证

在隔离 Fabric 服务端（Loader 0.19.5 + Fabric API 0.116.17）做 A/B 对照：

| 运行 | 结果 |
| --- | --- |
| 原版 `1.21.1-1.0.23` | 复现两条 `NoSuchMethodException` |
| 应用补丁后 | 反射初始化成功，无报错 |

---

## 8. 快捷键

交易界面内按 `C` 刷新交易（可在「选项 → 控制 → 按键绑定 → 物品栏」修改）。

> 若 `options.txt` 中该项为 `key.keyboard.unknown`，请在按键绑定界面重新设置。

---

## 9. 版权与免责

- **Trade Cycling** © **Max Henkel** — **All Rights Reserved**
  本仓库**不包含**其源代码，仅提供针对该版本的补丁与构建产物，供个人使用。
- **Visible Traders** © **Ramixin** — **MIT**
  https://github.com/Ramixin/VisibleTraders
- **Minecraft** © Mojang Studios / Microsoft
- **Fabric** © FabricMC

本仓库与上述作者**无隶属关系**，未获官方背书。
若原作者认为本仓库存在不当之处，请提 Issue，将立即处理。

**请勿将本补丁用于商业分发。** 如需公开传播修改版，请先取得原作者授权。

---

## 10. 说明

本兼容性修复由 **DeepSeek** 辅助完成：根因定位、代码修改、构建与 A/B 验证。
