# Android 虚拟定位（Mock Location）实践笔记

> 面向**自己的设备**，记录 Android 11–15（MIUI / HyperOS 为主）下把 GPS 位置伪装到指定坐标的方法、
> 反检测思路、以及常见坑。root 可选：不 root 有官方模拟位置通道，root 后能做到"更干净"。
>
> 用途请自律：签到、测试、开发调试、隐私保护都合理；伪造位置骗取考勤/外卖/打车补贴等属于违规，与本仓库无关。

---

## 0. 三条路线怎么选

| 路线 | 需要 root | 干净程度 | 适用 |
|---|---|---|---|
| A. 开发者选项「选择模拟位置应用」 | 否 | 低，App 能直接读到 `isMockProvider` 标记 | 快速测试、开发调试 |
| B. root + 系统级注入（Fake Location 的 root 模式 / LSPosed 模块） | 是 | 中，可清掉部分标记 | 日常使用、签到类 |
| C. root + Shamiko/DenyList 隐藏 + 模块级反检测 | 是 | 高 | 目标 App 检测严格时 |

判断逻辑：**先看目标 App 是否检查 `Location.isFromMockProvider()`**。检查 → A 必挂；不检查 → A 就够。

---

## 1. 路线 A：不 root（官方通道）

```
设置 → 关于手机 → 连点「版本号」7 次 → 开发者选项
开发者选项 → 选择模拟位置信息应用 → 选 Fake Location / Mock GPS
```

然后在该 App 里设定坐标并启动。命令行等价检查：

```bash
# 看当前是否允许模拟位置（设置里选中的应用会写进 secure 表）
adb shell settings get secure mock_location
# 看 AppOps 是否放行
adb shell appops get <包名> android:mock_location
```

> 注意：Android 6 之后**必须**在开发者选项里指定 App，直接调 `LocationManager.setTestProviderLocation` 会抛
> `SecurityException`；只有被选中的应用才有权限。

路线 A 的破绽：

- `Location.isFromMockProvider()` / `location.isMock` 会返回 true；
- `Settings.Secure.ALLOW_MOCK_LOCATION` 状态可被读取；
- 部分 App 直接遍历已安装的模拟位置应用名单。

---

## 2. 路线 B：root 后系统级注入

### 2.1 Fake Location 的 root 模式

多数「Fake Location」类应用在检测到 root 后，会提供：

- **Root 模式 / 系统模式**：把自己的定位服务注册成系统 Provider，绕过 `isFromMockProvider`；
- **反检测（隐藏模拟位置）**：清除 mock 标志，配合 root 权限直接改 `Location` 对象。

步骤：

```
① 设备已 root（Magisk，见 xiaomi-root-guide 仓库）
② Magisk → 超级用户 → 允许 Fake Location
③ App 内：设置 → 运行模式 → Root 模式
④ 设定坐标 → 启动 → 用地图 App 交叉验证
```

### 2.2 LSPosed 模块方式（更可控）

```
① Magisk 装 LSPosed(Zygisk) → 重启
② 安装定位伪装类模块（作用域勾选目标 App）
③ 模块内设置坐标 / 轨迹
④ 目标 App 强制停止后重开
```

优点：模块级 hook，能同时改 `Location`、`GnssStatus`、`Sensor` 等；
缺点：与检测方的对抗是持续的，App 一更新就可能失效。

---

## 3. 路线 C：反检测组合拳

目标 App 常见检测项与应对：

| 检测项 | 读取方式 | 应对 |
|---|---|---|
| `isFromMockProvider()` | `Location#isFromMockProvider` | 模块 hook 返回 false / root 模式注册系统 Provider |
| 模拟位置应用是否被选中 | `Settings.Secure` / `AppOps` | 用 root 模式（不经过开发者选项通道），并清 AppOps 记录 |
| 开发者选项是否开启 | `Settings.Global.DEVELOPMENT_SETTINGS_ENABLED` | 隐藏或改回 0（注意副作用） |
| 已安装应用清单含"Fake/Mock/Location" | `PackageManager` | 用模块隐藏应用名 / 改包名版本 |
| root 痕迹 | `su` 二进制、`magisk` 进程、挂载项 | Magisk DenyList + Shamiko，把目标 App 加入排除 |
| 传感器与 GPS 不一致 | 加速度/陀螺仪/基站/WiFi 指纹 | 选"静止模式"，或让轨迹有合理速度；必要时配合基站伪装 |

Magisk 侧隐藏（与 root 仓库联动）：

```
Magisk → 设置 → 遵守排除列表（打开）→ 配置排除列表 → 勾选目标 App
（装了 Shamiko 时，排除列表即"隐藏名单"）
```

---

## 4. 坐标与轨迹的细节

- **坐标系**：App 里填的通常是 WGS-84；国内地图显示是 GCJ-02，两者差 50–500 m。要"看起来对"，就得按目标显示系反算。
- **高度/精度**：假的 `altitude`、`accuracy` 给 5–15 m 更像真机；精度恒等于 0 很可疑。
- **速度**：瞬移会被轨迹分析抓；跨城要用"路线模拟"，给合理时速。
- **时间戳**：`Location#getTime` 必须是当前时间，写死过去的时间戳会被判异常。
- **基站/WiFi 定位**：只改 GPS 而基站没变，混合定位 App 会把你拉回真实位置；需要同时伪装 cell/wifi（多数 App 做不到，这就是它检测的点）。

---

## 5. 常用 adb 命令

```bash
# 允许某应用模拟位置（也可在开发者选项里点选）
adb shell appops set <包名> android:mock_location allow

# 查看当前模拟位置状态
adb shell dumpsys location | grep -i -E "mock|gps" | head -20

# 强制停止并清缓存后重开目标 App
adb shell am force-stop <目标包名>

# 查看目标 App 是否在 Magisk 排除列表生效（需 root）
adb shell su -c "magisk --denylist ls" 2>/dev/null
```

---

## 6. 坑位速查

| 现象 | 原因 | 处理 |
|---|---|---|
| 设了模拟位置但地图不动 | 开发者选项没选 App / 高精度定位抢占 | 选应用 + 关闭 WiFi/蓝牙扫描辅助 |
| 一启动就跳回真实位置 | 基站/WiFi 混合定位覆盖 | 开飞行模式只留 GPS，或同时伪装 cell |
| 目标 App 提示"检测到模拟位置" | `isFromMockProvider` 检查 | 走 root 模式 / LSPosed 模块 |
| root 模式无效 | Magisk 未授权该 App | Magisk → 超级用户 → 允许 |
| 模块生效但 App 闪退 | hook 与 App 版本不兼容 | 换模块版本或改用 App 自带 root 模式 |
| 重启后失效 | 模拟 Provider 未持久化 | App 内开"开机自启"，或用模块常驻 |

---

## 7. 合规提醒

虚拟定位技术本身是中性的（开发调试、隐私保护、游戏测试都在用）。
但把它用于**考勤代打卡、虚假行程骗取补贴、外卖/网约车刷单**等，属于违约甚至欺诈，
后果由使用者承担。本仓库只做技术记录。
