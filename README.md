# Anka Mosaico WebSocket Gamepad

基于 **ESP-Mosaico / ESP32-S31** 的 Wi-Fi 游戏手柄，为 Anka 的 `anka-outpost-strike` 提供实体按键、九宫格触屏、体感观察和震动反馈。

打开应用后沿用 Mosaico 系统已保存的 Wi-Fi，扫描游戏配对码，使用 **WebSocket + JSON** 绑定当前玩家。无需再次用手机配置网络，也无需输入服务器地址。

- 仓库名称：`anka-ws-gamepad`
- 设备应用名称：`anka_handpad`
- 当前固件版本：`0.4.1`
- 固件框架：原生 ESP-IDF，保留官方 Vibe / Recovery ABI 2

## 功能

- **左实体键开火、右实体键瞄准**，对应按键灯跟随状态。
- 中心圆起手拖动控制前后左右移动；短点中心跳跃，其他格短点执行相应功能。
- 顶排横滑超过半格后激活转身；之后每格对应180°、1/3格对应60°，按住期间可反向滑动，停手停止转动。
- 移动轨迹、转身门槛提示，以及中心按钮的红绿连接状态。
- BMI270 体感：按实际握持姿势校准，发送完整相对四元数姿态。
- 摄像头彩色预览、二维码识别、绑定进度反馈。
- 射击、受击等游戏事件触发马达与状态灯反馈。
- 断线释放输入，保留本次应用会话的绑定和校准，持续尝试重连。

### 与蓝牙版的区别

| 项目 | 本仓库：WebSocket 版 | BLE GATT 版 |
|---|---|---|
| 设备应用 | `anka_handpad` | `anka_gatt_gamepad` |
| 接入方式 | 系统 Wi-Fi + 扫描游戏二维码 | 浏览器选择蓝牙设备，无需扫码 |
| 输入链路 | 设备 → Anka Device 服务 → 游戏网页 | 设备 → 本机浏览器 |

一个玩家只绑定一台设备。两种固件分别作为 Vibe 应用保存，切换前先解除游戏中的原设备绑定。
本仓库不实现标准 USB / BLE HID 手柄，不直接兼容其他游戏的 Gamepad API。

## 使用流程

1. 在 Mosaico 系统设置中配置一次 Wi-Fi，然后打开 **`anka_handpad`**。
2. 游戏点击「手柄设备接入 → 扫码接入」（旧版界面直接显示二维码）。
3. 翻开摄像头对准配对码。应用显示识别和连接反馈，绑定成功后进入校准页。
4. 按实际游戏姿势握好设备，同时按住屏幕 A+B 或实体双键约800ms，提示松手后全部松开，确定握姿。
5. 进入纯图标手柄页。中心跳跃按钮**绿框表示连接、红框表示断开**。

校准不要求必须立起或完全静止，允许自然手抖；松手时的实际姿势作为参考。六轴 IMU 没有磁北参考，仍存在长期航向漂移的物理边界。

绑定后不响应屏幕边缘重扫或退出手势，避免滑动误触；普通断线自动重连，不重复校准。应用退出或重启后重新扫码。
AI 键单击仅在未绑定时返回 Vibe。明确解绑、玩家授权撤销或凭证失效，需要结束本次应用会话后重新绑定。

### 网络与服务器地址

设备复用官方系统网络配置，只读获取 Wi-Fi 凭证，不创建配网热点。系统尚未联网时，先返回 Vibe 设置网络。

二维码提供服务器地址和一次性 `bindKey` / `token`。ESP 不能访问电脑的 `localhost` / `127.0.0.1`，局域网调试时应在 **Anka 服务端**配置设备可访问的地址，例如：

```dotenv
ANKA_PUBLIC_URL=http://192.168.0.102:5173
```

以上 IP 和端口仅为示例；按实际网络与部署修改。生产使用可信 HTTPS / WSS。
当前 `sdkconfig.defaults` 启用了 `CONFIG_ANKA_ALLOW_INSECURE_LOCAL`，允许受信任局域网内的 HTTP / WS 调试；发布生产包时可在 `menuconfig` 关闭。
设备端没有独立 host 配置项，配对和重连凭证只保存在本次应用的内存中。

## 开发目录

```text
anka-ws-gamepad/
├── main/                    # 输入、UI、摄像头、Wi-Fi、JSON/WS 和体感
├── tests/                   # 手势、扫码预处理、校准与姿态主机测试
├── tools/                   # 固定依赖准备、构建、打包与校验
├── licenses/                # 第三方许可声明
├── .mosaico.json            # 官方 CLI 的默认应用配置
├── CMakeLists.txt
├── sdkconfig.defaults
├── partitions.csv
├── dependencies.json        # SDK / BSP / utils / quirc 固定版本
└── dependencies.lock        # ESP-IDF Component Manager 锁文件
```

固件源码和依赖准备脚本位于本仓库，不依赖 Anka 的 `doc/mosaico` 资料缓存才能构建。游戏服务与网页接收端由配套 Anka / `anka-outpost-strike` 仓库提供。

## 准备依赖与构建

以下命令均在 **本仓库根目录**执行。准备 Python 3.11+、Git 和可下载依赖的网络；主机测试还需要 C 编译器。
当前命令使用 Bash，适合 macOS / Linux；Windows 可使用相应开发环境另行验证。

```sh
python3 tools/bootstrap.py --idf --tools --archive-submodules
./tools/idf.sh build
python3 tools/package_firmware.py
```

如已配置 GitHub SSH 别名，可使用：

```sh
python3 tools/bootstrap.py --idf --tools --archive-submodules \
  --github-host github_vitcou
```

`--github-host` 或 `ANKA_GITHUB_HOST` 影响 SDK 的 Git 下载；第三方归档仍通过 GitHub HTTPS 下载。
工具链保存在 `.sdk/`，BSP / utils / quirc 保存在 `.deps/`；`idf.sh` 仅为当前命令激活这些路径。

| 依赖 | 固定版本 / 提交 |
|---|---|
| ESP-IDF | `7b9cc1ac79f865983f59bb8ff3ff43eb74ff1dbe` |
| ESP-Mosaico BSP | `5f88b91e1be055d1d3674364123b2779a1758377` |
| Mosaico Utils / Iris / Vibe | `be151ae047014e79a9c7ae0092dd203844499003` |
| quirc | `927d680904dc95fdff4cd9d022eb374b438ff8f2` |
| CMake / Ninja | `3.31.10` / `1.13.0` |

以 [dependencies.json](dependencies.json) 和 [dependencies.lock](dependencies.lock) 为准，不跟随上游主分支自动升级。
构建会校验官方 SC101IOT PCLK 与 DVP frame-tail 补丁，缺失时拒绝继续，避免生成摄像头异常的固件。
当前 UI 使用原生矢量图标，**不需要执行 `tools/prepare_font.py`**；字体工具为历史工具，不参与当前构建。

可选配置：

```sh
./tools/idf.sh menuconfig
```

### 从旧目录迁移

旧 `build/` 和部分 Python 虚拟环境可能保存原绝对路径，迁移后不能直接保证可复用。
先将旧 `build/` 改名或移走，重新准备依赖并构建；环境仍引用旧位置时重新创建相应本地工具环境。
`build-*` / `build_*` 迁移备份与工具缓存均已忽略，不提交到源码仓库。

## 安装到 Mosaico

构建产物：

```text
build/anka_handpad-system-update.irisfw
```

设备需已有兼容的 **Vibe Mode / Recovery ABI 2**，并通过 USB 连接。安装前在 Vibe 的 **Develop → Save Current Build** 保存需要保留的原应用。

```sh
python3 mosaico.py iris list --details
python3 mosaico.py iris system-update \
  --bundle build/anka_handpad-system-update.irisfw
python3 mosaico.py iris device-status
```

多设备时加 `--device-id <实际设备ID>`。如设备仍由原工程的 Gateway 管理，可使用该工程的 `mosaico.py` 安装本包，避免多个项目同时占用 USB。

`system-update` 是官方命令名称；本包包含分区表和主应用，替换当前主应用槽，不包含 bootloader、Vibe 或用户 Wi-Fi / NVS 镜像。
安装后在 **Develop → Save Current Build** 保存为 **`anka_handpad`**，即可从 **My Ideas / Manage** 打开。蓝牙版的名称是 `anka_gatt_gamepad`。

预编译包可作为 GitHub Release 附件发布；安装已有包无需重新编译，但主机仍需准备官方 Iris CLI 及其运行依赖。

## 协议与实现

配对入口形如：

```text
https://<host>/v1/device/join?bindKey=<key>&token=<token>
```

扫描后转换为对应 WS / WSS 地址，不额外要求 `hello`。设备鉴权独立于 User 登录和 Share 邀请；手柄控制已有玩家，不创建新的玩家席位。

- 连续 JSON 快照包含 `move`、`buttons`、`turn`、`turnTotal` 和 `pose`。
- `turnTotal` 是未取模的累计角度，避免快滑跨180°或合并快照时丢失圈数。
- `pose` 是相对握姿的完整四元数，取最新状态，不按消息逐包累加。
- 单次功能使用 `action`，由游戏判断是否可执行。
- 连接恢复先发送中立快照，清除移动和保持按键；游戏端保留150ms输入超时释放机制。
- 受击等事件通过反向 `feedback` 返回设备执行限幅震动和灯光反馈。

主要实现：`controller_link.c` / `controller_url.c` 负责 WS；`controller_wifi.c` 复用系统网络；`gesture.c` 处理移动与转身；`controller_motion.c` / `orientation.c` 负责体感；`controller_scan.c` / `controller_decode.c` 负责相机和扫码。

配套 **Anka 仓库**中的完整接口文档为 `docs/DEVICES.md`，研究和实施记录位于 `doc/mosaico/`。

## 验证与调试

无需设备即可运行现有纯 C 校准与姿态检查：

```sh
printf 'void calibration_tests(void); void orientation_tests(void); int main(void) { calibration_tests(); orientation_tests(); return 0; }\n' > /tmp/anka-ws-motion-test.c
cc -std=c11 -O2 -Wall -Wextra -I main \
  /tmp/anka-ws-motion-test.c tests/calibration.c tests/orientation.c \
  main/imu_calibration.c main/orientation.c -lm -o /tmp/anka-ws-motion-test
/tmp/anka-ws-motion-test
```

`tools/check_host.py` 还覆盖手势、URL、相机预处理及 quirc，但其二维码生成步骤保留了旧 Anka 目录布局假设。
独立仓库迁移后，需先把该脚本的二维码编码器工作目录调整到已安装 `qrcode` 依赖的游戏目录，再运行完整检查。

官方 Iris USB 提供非敏感诊断、屏幕读取和触点注入。USB 注入不能替代实机手指、实体双键、镜头光学扫码或实际震动体验；这些需单独验收。
不要将 Wi-Fi 密码、配对 URL、Device 凭证或 Gateway 状态文件提交到 Git。

## 第三方资料与许可

- [ESP-Mosaico 入门指南](https://mosaico.ai/zh-hans/guide)
- [ESP-Mosaico Vibe](https://github.com/esp-mosaico/esp-mosaico-vibe)
- [ESP-Mosaico BSP](https://github.com/esp-mosaico/esp-mosaico-bsp)
- [ESP-IDF](https://github.com/espressif/esp-idf)
- [quirc 许可](licenses/quirc-ISC.txt)
- [历史字体资源许可](licenses/NotoSansSC-OFL.txt)

第三方依赖遵循各自许可证；源码缓存保留其许可文件，分发固件时保留相应声明与通知。
