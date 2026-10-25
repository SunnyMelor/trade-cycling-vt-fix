# Release v1.0.23-vtfix

## Trade Cycling × Visible Traders 兼容性修复补丁

修复「刷新交易后未解锁交易消失、必须重开界面才能预览」的问题。

- **目标版本**：Trade Cycling `1.21.1-1.0.23`（分支 `1.21.1`）
- **Minecraft**：1.21.1
- **加载器**：Fabric Loader ≥ 0.15.11
- **依赖**：Fabric API ≥ 0.102.0+1.21.1
- **可选**：Visible Traders 1.3.2

### 文件

| 文件 | 说明 |
| --- | --- |
| `0001-visibletraders-compat.patch` | 源码补丁（12 文件，+172 / −96） |
| `trade-cycling-fabric-1.21.1-1.0.23.jar` | 应用补丁后的构建产物 |

产物校验：

| 项 | 值 |
| --- | --- |
| 大小 | 522,250 B |
| SHA-256 | `5EB197AD6C0EF5052E3842EA41DD493C4E11FE5641DFA1BFF8A257EA038807E1` |

### 使用

```bash
git clone https://github.com/henkelmax/trade-cycling.git
cd trade-cycling && git checkout 1.21.1
git apply 0001-visibletraders-compat.patch
./gradlew :fabric:build   # 需 JDK 21
```

或直接使用现成的 `trade-cycling-fabric-1.21.1-1.0.23.jar`。

### 修复内容

1. **反射桥接适配 Visible Traders 1.3.2**
   - `visibleTraders$forceTradeGeneration` → `visibleTrades$regenerateTrades`
   - 改用 `visibleTraders$getCombinedOffers` / `visibleTraders$getShiftedLevel`
     下发「已解锁 + 未解锁」完整列表
   - 保留旧方法名回退，兼容更早的 VT 版本
2. **刷新后锁定交易立即保留** — 无需关闭/重开界面
3. **新增**：刷新按钮悬浮提示显示当前快捷键（如 `刷新交易（C）`）

### 快捷键

交易界面内按 `C` 刷新（可在「选项 → 控制 → 按键绑定 → 物品栏」修改）。

> 若 `options.txt` 中该项为 `key.keyboard.unknown`，请在按键绑定界面重新设置。

### 验证

在隔离 Fabric 服务端（Loader 0.19.5 + Fabric API 0.116.17）做 A/B 对照：

| 运行 | 结果 |
| --- | --- |
| 原版 `1.21.1-1.0.23` | 复现两条 `NoSuchMethodException` |
| 应用本补丁 | 反射初始化成功，无报错 |

### 版权提醒

本补丁针对 **All Rights Reserved** 的 Trade Cycling。
本仓库**不包含**其源代码，仅提供补丁与构建产物，供个人使用。
版权归原作者 **Max Henkel** 所有：https://github.com/henkelmax/trade-cycling
详见 [THIRD-PARTY-NOTICES.md](../THIRD-PARTY-NOTICES.md)。
