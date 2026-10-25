# 根因分析：刷新交易后锁定交易消失

## 摘要

| 项 | 内容 |
| --- | --- |
| 现象 | 刷新交易后未解锁交易消失，需关闭并重开界面才恢复 |
| 涉及模组 | Trade Cycling `1.21.1-1.0.23` + Visible Traders `1.3.2` |
| 平台 | Minecraft 1.21.1 / Fabric |
| 根因 | 跨模组反射桥接的**方法名不匹配**，导致集成静默失效 |
| 性质 | 上游版本错配，非逻辑缺陷 |

---

## 1. 证据链

### 1.1 运行日志

`logs/latest.log` 启动阶段输出：

```
Failed to initialize visible traders integration
java.lang.NoSuchMethodException: net.minecraft.class_1646.visibleTraders$forceTradeGeneration()
    at de.maxhenkel.tradecycling.compatibility.VisibleTraders.getRegenerateMethod(VisibleTraders.java:55)
    at de.maxhenkel.tradecycling.compatibility.VisibleTraders.init(VisibleTraders.java:49)
    at de.maxhenkel.tradecycling.TradeCyclingMod.init(TradeCyclingMod.java:23)

Failed to initialize visible traders integration
java.lang.NoSuchMethodException: net.minecraft.class_3222.visibleTraders$wrapAndSendMerchantOffers(
    net.minecraft.class_1915, int, net.minecraft.class_1916, int, int, boolean, boolean)
    at de.maxhenkel.tradecycling.compatibility.VisibleTraders.getSendMerchantOffersMethod(VisibleTraders.java:66)
```

### 1.2 反射桥接代码（修复前）

```java
private static Method getRegenerateMethod() {
    Method forceTradeGeneration = Villager.class.getDeclaredMethod("visibleTraders$forceTradeGeneration");
    ...
}

private static Method getSendMerchantOffersMethod() {
    Method sendMerchantOffers = ServerPlayer.class.getDeclaredMethod(
        "visibleTraders$wrapAndSendMerchantOffers",
        Merchant.class, int.class, MerchantOffers.class, int.class, int.class, boolean.class, boolean.class);
    ...
}
```

`init()` 捕获异常后，两个 `Method` 字段保持 `null`：

```java
public static void init(TradeCyclingMod mod) {
    if (!mod.isModLoaded("visibletraders")) return;
    regenerateTrades = getRegenerateMethod();       // → null
    sendMerchantOffers = getSendMerchantOffersMethod(); // → null
}
```

### 1.3 Visible Traders 1.3.2 的真实 API

对 `VisibleTraders-1.3.2.jar` 执行 `javap`（`net.ramixin.visibletraders.ducks.VillagerDuck`）：

```
public interface VillagerDuck {
    void visibleTraders$setLockedTradeData(LockedTradeData);
    Optional<LockedTradeData> visibleTraders$getLockedTradeData();
    void visibleTrades$regenerateTrades();          // ← 改名
    MerchantOffers visibleTraders$getCombinedOffers();   // ← 新增
    int visibleTraders$getShiftedLevel();                // ← 新增
    void visibleTraders$updateTrades();
}
```

对照关系：

| 修复前期望 | 1.3.2 实际 |
| --- | --- |
| `visibleTraders$forceTradeGeneration()` | `visibleTrades$regenerateTrades()` |
| `visibleTraders$wrapAndSendMerchantOffers(...)`（位于 `ServerPlayer`） | 不存在；改为 `getCombinedOffers()` + `getShiftedLevel()` |

---

## 2. 为什么表现为「必须重开界面」

### 2.1 刷新流程（服务端）

`TradeCyclingMod#onCycleTrades`：

```java
villager.setOffers(null);
villager.getOffers();                            // 重掷普通交易
villagerAccessor.invokeUpdateSpecialPrices(player);
villager.setTradingPlayer(player);
VisibleTraders.forceTradeGeneration(villager);   // ← 空操作（Method == null）
sendOffers(player, container.containerId, villager); // ← 退化为普通列表
container.slotsChanged(merchantAccessor.getTradeContainer());
```

由于 `sendMerchantOffers == null`，`sendTrades` 直接返回 `false`，退回到：

```java
player.sendMerchantOffers(containerId, villager.getOffers(), ...);
```

即**只发送已解锁的普通交易**。

### 2.2 客户端展示（Visible Traders）

VT 1.3.2 的客户端合并逻辑位于 `MerchantScreenMixin`：在渲染、点击、滚动时把 `MerchantMenu#getOffers` 替换为「已解锁 + 未解锁」的合并列表。

而合并列表的来源只有一处 —— 服务端在 `Merchant.openTradingScreen` 时通过 `@WrapOperation` 把 `sendMerchantOffers` 的入参换成 `getCombinedOffers()`：

```java
@WrapOperation(method = {"openTradingScreen"},
    at = @At(value = "INVOKE", target = "...Player;sendMerchantOffers(...)"))
private void sendLockedOffersWithNormalOffersOnScreenOpen(...) {
    VillagerDuck duck = VillagerDuck.of(villager);
    original.call(player, syncId, duck.visibleTraders$getCombinedOffers(),
                  duck.visibleTraders$getShiftedLevel(), k, bl, bl2);
}
```

**这个钩子只在「打开界面」时触发。**

因此：

1. 刷新时走的是 Trade Cycling 自己的 `sendMerchantOffers`，不经过 VT 的包装 → 客户端只更新了普通交易；
2. 客户端缓存的合并列表仍是刷新前快照（甚至为空）；
3. VT 1.3.2 内部**没有**「收到新交易包后重新请求锁定列表」的网络通道；
4. 直到关闭并重开界面，`openTradingScreen` 钩子再次执行，合并列表才被重新计算并下发。

这与用户描述完全一致。

---

## 3. 修复策略

选择在 **Trade Cycling 侧**修复，理由：

- 不需要修改 Visible Traders（MIT，不动第三方制品）；
- 与上游 `1.21.1` 分支的既有设计意图一致（本就通过反射桥接）；
- 可同时兼容新旧两代 VT API。

### 3.1 方法名探测（兼容多版本）

```java
@Nullable
private static Method findVillagerMethod(String... names) {
    for (Method method : Villager.class.getDeclaredMethods()) {
        if (method.getParameterCount() > 0) continue;
        for (String name : names) {
            if (method.getName().equals(name)) {
                method.setAccessible(true);
                return method;
            }
        }
    }
    return null;
}

regenerateTrades = findVillagerMethod("visibleTrades$regenerateTrades",
                                      "visibleTraders$forceTradeGeneration");
getCombinedOffers = findVillagerMethod("visibleTraders$getCombinedOffers");
getShiftedLevel  = findVillagerMethod("visibleTraders$getShiftedLevel");
```

### 3.2 发送合并列表

```java
public static void sendTrades(ServerPlayer player, Villager villager, int containerId) {
    if (wrapAndSendMerchantOffers != null && sendWithLegacyMethod(player, villager, containerId)) {
        return; // 旧版 VT 自行包装
    }
    player.sendMerchantOffers(containerId, getOffers(villager), getLevel(villager),
                              villager.getVillagerXp(), villager.showProgressBar(), villager.canRestock());
}
```

`getOffers` → `visibleTraders$getCombinedOffers()`（已解锁 + 未解锁）
`getLevel` → `visibleTraders$getShiftedLevel()`（VT 用高位编码未解锁数量）

这样刷新时下发的包与「打开界面」时完全一致，锁定交易立即随包到达客户端。

---

## 4. A/B 验证

隔离 Fabric 服务端，同一组模组，仅替换 Trade Cycling 制品：

| 运行 | 关键日志 |
| --- | --- |
| 原版 jar | 两条 `NoSuchMethodException`，集成失效 |
| 修复后 jar | 无任何 `NoSuchMethod` / `Could not find` 报错，恢复正常 |

`KeyboardMixin` 注入目标亦已通过 mappings 校验：
`method_1466` = `KeyboardHandler.keyPress`。
