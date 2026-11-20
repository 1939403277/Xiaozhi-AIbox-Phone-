# Bug 修复日志

日期：2026-11-19
项目：`3.5LCD_xiaozhi`
板型：`woyou-aibox-lcd35`（ESP32-S3-DevKitC-1 + 正点原子 ATK-MD0350，NT35310 / 8080-16bit / 480×320 + XPT2046）
SDK：ESP-IDF v6.0.2

## 一、启动崩溃：NT35310 初始化命令参数超出 i80 驱动缓冲区

**现象**：开机调用栈停在 `panel_nt35310_init()` → `esp_lcd_panel_io_tx_param()` → i80 面板 IO；构造板级单例时整机起不来。

**根因**：

- ESP-IDF 的 i80 面板 IO 只用一块 32 字节的内部 `format_buffer`（`LCD_I80_IO_FORMAT_BUF_SIZE`）暂存命令参数。
- 本项目配置为 `bus_width = 16`、`lcd_param_bits = 8`，驱动会把每个参数字节展开成 2 个总线字节。
- 于是单次参数发送的实际上限只有 **16 字节**：超过 16 字节会写穿 32 字节缓冲区造成堆损坏；超过 32 字节则直接触发驱动里的断言。
- 厂商初始化表（`nt35310_init_cmds`）中 `0xB0`、`0xB3`、`0xB7` 等已超过 16 字节，`0xC3` 达到 48 字节、`0xE0~0xE8` 达到 36/32 字节，正好命中崩溃路径。
- 不能把长命令拆成多次 `tx_param`：重复写同一条命令会让屏内参数索引归零，后半段参数会覆盖寄存器开头。

**修复**（`main/boards/woyou-aibox-lcd35/nt35310.c`）：

- 新增 `nt35310_tx_param()`：
  - 参数 ≤16 字节：仍走 `esp_lcd_panel_io_tx_param()`。
  - 参数 >16 字节：按「低字节放参数、高字节补 0」展开成 16 位总线字（与驱动自身的展开结果逐字节一致），改用 `esp_lcd_panel_io_tx_color()` 发送。该路径把调用方缓冲区直接交给 DMA，不受 32 字节限制。
  - 使用静态 64 字节对齐的内部 RAM 缓冲区；长度补齐到 4 字节满足 DMA 对齐；发送后 `vTaskDelay(1)` 等待 DMA 读完再复用缓冲区。
  - 奇数字节参数（如 `0xCF` 的 17 字节）会多补一个全 0 总线字，对该寄存器无副作用。
- `panel_nt35310_init()` 中的厂商初始化表循环改用此函数；`MADCTL`、`COLMOD`、`CASET`、`RASET` 等短命令行为不变。
- `lcd_param_bits = 8` 的配置无需修改：厂商表中的 `(值, 0x00)` 成对写正是 16 位 8080 总线上 `LCD_WR_DATA()` 的原始字节流。

**验证**：全量编译通过；设备能继续走到 LVGL 界面，说明初始化阶段已不再崩溃。

## 二、状态灯按 1 颗驱动，实际是 6 颗 WS2812

**现象**：GPIO48 上串联 6 颗 WS2812，但板级代码只按 1 颗初始化，状态效果只驱动第一颗。

**根因**：`GetLed()` 里把 `CircularStrip` 的 `max_leds` 写死为 1；`config.h` 也只有 GPIO 定义，没有灯珠数量。

**修复**：

- `main/boards/woyou-aibox-lcd35/config.h`：
  - 变体 4 新增 `LCD35_STATUS_LED_COUNT 6`（v1.0 的 GPIO48、v1.1 的 GPIO38 都适用）。
  - 文件末尾增加缺省定义：其他变体未定义时按 1 颗处理。
- `main/boards/woyou-aibox-lcd35/woyou_aibox_lcd35.cc`：`GetLed()` 改用 `LCD35_STATUS_LED_COUNT`。

`CircularStrip` 的常亮/呼吸/闪烁/滚动效果本来就遍历 `max_leds_`，无需改 LED 实现。若某版本板载灯珠更少，多发送的像素会被灯链尾部丢弃。

## 三、整屏彩虹噪点花屏

**现象**：屏幕能显示 LVGL 画面的结构（表情圆、背景色块），但颜色变成高频彩虹噪点，看起来像随机内存数据。

**根因**：

- `SpiLcdDisplay` 是 SPI 屏与 i80 屏共用的显示层，其中 esp_lvgl_port 的 `.swap_bytes` 被固定为 1。
- SPI 屏逐字节发送，屏期望高字节先到，因此需要交换；而 8080/i80 16 位并口由 DMA 把 RGB565 缓冲区按 16 位小端直接送到 D15..D0，一个像素的字节顺序本来就是正确的。
- 再交换一次等于把 `0xF800`（红）变成 `0x00F8`，画面结构保留但颜色全部错乱，表现为整屏彩虹噪点。

**修复**：

- `main/display/lcd_display.h` / `main/display/lcd_display.cc`：`SpiLcdDisplay` 构造函数新增 `bool swap_bytes = true` 参数，`.swap_bytes` 改用该参数；默认值保持所有现有 SPI 屏行为不变。
- `main/boards/woyou-aibox-lcd35/woyou_aibox_lcd35.cc`：本板构造 `SpiLcdDisplay` 时显式传 `false`。

**验证**：用户上机确认显示恢复正常。该项为跨板型公共显示层改动，其他板型因默认值保持原行为，但仍建议后续回归一台 SPI 屏板。

## 四、排查但非代码问题：构建在 wpa_auth.c 处中止

**现象**：一次构建输出 `Building C object esp-idf/wpa_supplicant/.../wpa_auth.c.obj` 后接 `ninja: build stopped: subcommand failed`，无 `error:` 具体信息。

**处理**：使用同一 ESP-IDF v6.0.2 环境完整重编，2229 个编译步骤全部通过，`wpa_auth.c` 及其所在组件无报错；随后的增量构建同样通过。判定为偶发的构建进程被外部因素打断（并发构建、内存不足或杀毒软件拦截），非源码缺陷，未做代码改动。

**附带说明**：当前 CLI 环境没有设置 `ESP_ROM_ELF_DIR`，CMake 会警告 `Error while generating esp_rom gdbinit`。只影响 GDB 初始化脚本，不影响固件编译与运行。

## 验证记录

| 项目 | 状态 |
|---|---|
| ESP-IDF v6.0.2 全量构建（2229 步） | 通过 |
| `swap_bytes` 修改后的增量构建 | 通过 |
| LCD 显示恢复（用户上机确认） | 通过 |
| 6 颗灯珠全亮效果上机确认 | 待验证 |
| 触摸 / 音频 / 长时间稳定性 | 待验证 |
| 其他 SPI 屏板型回归（公共显示层改动） | 待验证 |

---

## 五、2026-11-20 迭代：触摸手势 / USB Host / 行情页 / 设置持久化

本轮为功能迭代（非缺陷修复），记录变更点与遗留项。

### 5.1 手机式手势导航

**原状**：`Xpt2046Touch` 的滑动回调无参数，板级回调只能"滑一下就打开菜单"；菜单内无手势。

**改动**：

- `xpt2046.h/.cc`：`OnSwipe(std::function<void(int dx, int dy)>)`，位移单位改为**屏幕像素**。
  一次按压只在松手沿产生一个动作：位移 ≥60px 且主轴 ≥ 次轴 2 倍 → 滑动；≤40px 且按住 ≤800ms → 点击；两者互斥。
- `main_menu.h/.cc`：新增 `ShowPage(page)` / `HandleSwipe(dx, dy)` / `IsHome()`，
  功能页标题追加 `(n/9)` 作为左右滑动的反馈。
  - 未激活时上划 → 唤出菜单
  - 已激活：上划 → 首页（首页再上划 = 退出菜单）；左划 → 下一页；右划 → 上一页
- `woyou_aibox_lcd35.cc`：`OnSwipe` 改为 `Schedule` 回主任务调用 `MainMenu::HandleSwipe`。

### 5.2 触摸失效排查（未定论，已加取证手段）

**原状**：`ReadRaw()` 只要有一帧超界（`TOUCH_RAW_MIN~MAX`）就判为松手，电阻屏抖一帧会让 LVGL
收到 `RELEASED` 而丢掉点击；且没有任何上电自检日志。

**改动**（`xpt2046.cc`）：

- 松手去抖：连续 `kReleaseStableFrames = 2` 帧读不到有效值才真正判松手，按下沿保持。
- 上电自检：`Init()` 末尾读一次 X/Y 并打印 `self-test raw x=.. y=..`。
  判读：恒为 0/4095 → MISO/CS/命令字问题；落在量程内且随手指变化 → 正常。
- `LCD35_TOUCH_DEBUG` 由"每帧打印"改为"启动自检 + 每 500ms 一条"，避免刷日志。

**遗留**：触摸的最终根因（量程 / `TOUCH_AD_SHIFT` / 接线）必须拿到实机 raw 日志才能定性，
当前未上机验证。

### 5.3 USB Host 与 U 盘

- `config.h`：`LCD35_MOUNT_USB 1`；变体 4 的 `#error` 改为仅校验 `LCD35_KEEP_UART0`。
- `config.json` 增加板级 `sdkconfig_append`（不污染共享 `sdkconfig.defaults`）：
  `CONFIG_USJ_ENABLE_USB_SERIAL_JTAG=n`、`CONFIG_ESP_CONSOLE_SECONDARY_NONE=y`、
  `CONFIG_USB_HOST_CONTROL_TRANSFER_MAX_SIZE=512`、`CONFIG_LV_USE_KEYBOARD=y`。
  注：IDF 6.0.2 已无 `CONFIG_USB_HOST_ENABLED` 符号，Host 栈随 `usb_host_msc` 依赖自动参与构建，
  因此该项写了也不会报错，但不起开关作用。
- 烧录/日志改走 UART0（COM9），原生 USB 仅作 Host。
- 验证：`usb_disk.cc.obj` 与 `espressif__usb_host_msc` 均已进入构建产物。

### 5.4 主菜单第 4 项：B 站热点 → 股票行情

- 新增 `market_client.h/.cc`：`esp_http_client` 同步拉取 + 字符串解析（不引入 cJSON 解析大对象）。
  - A 股：`http://qt.gtimg.cn/q=%s`，自动补 `sh/sz/bj` 前缀；只取 ASCII 名称，规避 GBK 乱码。
  - 加密：`https://api.binance.com/api/v3/ticker/24hr?symbol=%s`，缺省补 `USDT`。
  - URL 模板与超时都在 `config.h`（`MARKET_STOCK_URL_FMT` / `MARKET_CRYPTO_URL_FMT`）。
- `main_menu.cc`：`BuildMarket()` 内含输入框、预设代码、结果区与 LVGL 软键盘（`lv_keyboard`）。
  查询放独立任务（6.1KB 栈），结果回写前用 `lv_obj_is_valid()` 检查控件，避免切页后写野指针。
- `main/CMakeLists.txt`：`PRIV_REQUIRES` 增加 `esp_http_client`。
- 首次编译触发两个问题并已修：`-Werror=frame-larger-than=1536`（缓冲移到堆）、
  `lv_keyboard` 未启用（加 `CONFIG_LV_USE_KEYBOARD=y`）。

### 5.5 设置项持久化

- 音量 / 语音唤醒 / 闹钟开关写入 NVS（namespace `lcd35`，键 `vol`/`wake`/`alarm`/`alarm_h`/`alarm_m`），
  `MainMenu::Init()` 时恢复，进设置页时把音量重新应用到 codec。
- U 盘行改为显示 `剩余 / 总容量 MB`（`esp_vfs_fat_info`），未插入时提示 FAT32/exFAT。

### 5.6 本轮验证

| 项目 | 状态 |
|---|---|
| `scripts/build.py woyou-aibox-lcd35 --name woyou-aibox-lcd35` 全量构建 | 通过（`Project build complete`） |
| U 盘驱动编入固件（`usb_disk.cc.obj` + `usb_host_msc`） | 通过 |
| sdkconfig 生效（USJ 关闭、次级 console 关闭、LVGL keyboard 打开） | 通过 |
| 触摸 raw 自检日志 | 待上机 |
| 手势 / 菜单切页 / 行情查询 / NVS 掉电保持 | 待上机 |
| U 盘实际挂载与 MP3/相册播放 | 待上机 |
