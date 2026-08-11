# ZMK 行为（Behavior）说明手册

> 本手册以 ZitaoTech KEYPOINT 的 `config/zitaotech_keypoint.keymap` 为基础，系统讲解 ZMK 键盘固件中的各种**行为（behavior）**是什么、怎么用，以及本项目里都用了哪些。
>
> 适合第一次接触 ZMK 的同学阅读。

---

## 〇、什么是"行为"（Behavior）？

在 ZMK 里，一个按键并不直接等于"某个字母"，而是等于**一个行为**。行为决定按下/松开这个键时会发生什么。

在键位文件里，行为用 `&名称` 引用，后面跟参数（称为 *binding cells*）：

```
&kp A             ; 行为 kp（按键），参数 A
&mo 1             ; 行为 mo（瞬时层），参数 1（层号）
&mt LSHFT ESC     ; 行为 mt（修饰键+按键），参数 LSHFT 和 ESC
```

`config/zitaotech_keypoint.keymap` 顶部就包含了很多头文件，把常见行为的"名称"引入进来：

```c
#include <behaviors.dtsi>            // 内置行为：&kp &mt &mo &to &trans &none ...
#include <dt-bindings/zmk/keys.h>    // 按键码：&kp A、&kp LSHFT、&kp N1 ...
#include <dt-bindings/zmk/bt.h>      // 蓝牙：&bt BT_NXT / BT_CLR ...
#include <dt-bindings/zmk/outputs.h> // 输出：&out OUT_TOG ...
#include <dt-bindings/zmk/pointing.h>// 指针：&mmv &mvm ...
#include <dt-bindings/zmk/mouse.h>   // 鼠标：&mkp &msc ...
#include <dt-bindings/zmk/backlight.h>// 背光：&bl BL_DEC ...
#include <dt-bindings/zmk/ext_power.h>// 外设电源：&ext_power ...
#include <dt-bindings/zmk/rgb.h>     // RGB：&rgb_ug ...
```

行为分为两大类：

| 类型 | 说明 | 例子 |
|---|---|---|
| **内置行为** | ZMK 自带，直接 `&名称` 使用 | `&kp`、`&mt`、`&mo`、`&bt`、`&mkp` |
| **自定义行为** | 在本文件 `behaviors { }` 块里定义，然后像内置行为一样用 | `&td0`、`&hm`、`&M_kp`、`&BLE_encoder` |

---

## 一、内置行为

### 1.1 按键类 `&kp`
发送一个按键（可组合修饰键一起用）。

```c
&kp A            // 字母 A
&kp LSHFT        // 左 Shift
&kp N1           // 数字 1
&kp C_MUTE       // 静音
&kp PG_DN        // Page Down
&kp LBKT         // [ 左方括号
&kp NUBS         // \ 反斜杠（非美式键盘位）
&kp GRAVE        // ` 反引号
```

本项目大量使用，例如默认层所有字母键、RAISE 层的 F 键等。

### 1.2 复合按键 `&mt`（Mod-Tap，修饰键+按键）
**一个键两种作用：按住 = 修饰键（Hold），轻点 = 按键（Tap）。**

```
&mt <Hold 行为> <Tap 行为>
```

最常见的用法：
```c
&mt LSHFT ESC    // 按住=Shift，单击=Esc
&mt LCTRL Z      // 按住=Ctrl，单击=Z
```

本项目默认层第一个键：
```c
&mt ESC TAB      // 按住=Esc，单击=Tab
```
（注意本项目把"按住"设成了 Esc，"单击"设成 Tab，和常见习惯相反。）

**相关参数**（在自定义 hold-tap 行为里配置）：
- `tapping-term-ms`：判定"按住还是单击"的时间阈值，超过即视为按住。
- `flavor`：判定倾向，见本文 2.1。

### 1.3 层类行为

| 行为 | 语法 | 作用 | 本项目使用 |
|---|---|---|---|
| `&mo` | `&mo <层号>` | **瞬时层**：按住激活，松开即回 | `&mo 1`(LOWER)、`&mo 2`(RAISE) |
| `&to` | `&to <层号>` | **切换层**：跳到该层并**常驻**，直到再次 `&to` | `&to 3`（LOWER 层切入 MOUSE） |
| `&tog` | `&tog <层号>` | **开关层**：在该层与默认层间来回切换（可作用于基础层） | 未用 |
| `&trans` | `&trans` | **透传**：本键不做任何事，把按键"穿透"给更低的层 | LOWER/RAISE 层大量使用 |
| `&none` | `&none` | **空行为**：彻底屏蔽，什么都不发生 | RESERVE 层、RAISE 层占位 |

**`&trans`（透传）很重要**：例如 LOWER 层的 `A` 键位写的是 `&trans`，意思是"这一层不接管 A，按 A 时直接落到 QWERTY 层的 A"。这样在 LOWER 层里字母键依然能打字。

### 1.4 系统与硬件类

| 行为 | 语法 | 作用 | 本项目使用 |
|---|---|---|---|
| `&bootloader` | `&bootloader` | 进入刷机模式（重启为 UF2 引导） | LOWER 层"Boot"键（宏 `&BOT`） |
| `&sys_reset` | `&sys_reset` | 软复位（重启，不进刷机） | 未用 |
| `&bt` | `&bt BT_CLR` | 清除当前蓝牙配置的配对 | LOWER 层左拇指区 |
| `&bt` | `&bt BT_NXT`/`BT_PRV` | 切换下一个/上一个蓝牙配置 | 自定义 `&BLE_encoder` |
| `&out` | `&out OUT_TOG` | 在 USB 与蓝牙之间切换输出 | LOWER 层右拇指区 |

### 1.5 鼠标类行为

| 行为 | 语法 | 作用 | 本项目使用 |
|---|---|---|---|
| `&mkp` | `&mkp LCLK` | 按住时按住某个鼠标键（点击） | 中央实体键 = 左/右键 |
| `&msc` | `&msc SCRL_UP` | 滚动鼠标滚轮 | 下排小键 = 上下滚动 |
| `&mmv` | `&mmv 500 0` | 按向量移动鼠标 | 未用 |
| `&mvm` | `&mvm UP` | 沿方向移动鼠标 | 未用 |
| `&mss` | `&mss SCRL_R` | 沿方向滚动 | 未用 |

**鼠标按键参数**：
```
LCLK / MB1   → 左键
RCLK / MB2   → 右键
MCLK / MB3   → 中键
```
本项目里 `&mkp LCLK`、`&mkp RCLK`、`&mkp MCLK`、`&mkp MB1/MB2/MB3` 都出现过。

### 1.6 灯光类行为

| 行为 | 语法 | 作用 | 本项目使用 |
|---|---|---|---|
| `&bl` | `&bl BL_DEC` / `&bl BL_INC` | 背光亮度减/增 | LOWER 层中央小键 |
| `&bl` | `&bl BL_TOG` / `&bl BL_SET 5` | 背光开关 / 设定亮度 | 未用 |
| `&rgb_ug` | `&rgb_ug RGB_TOG` | RGB 开关、亮度、色调等 | 未用（配置标注 unreal） |

### 1.7 传感器（编码器）行为
`&inc_dec_kp` 专门用于**旋转编码器**：正向转 = 一个按键，反向转 = 另一个按键。

```c
&inc_dec_kp C_VOL_UP C_VOL_DN   // 正向=音量+，反向=音量-
&inc_dec_kp LEFT RIGHT          // 正向=←，反向=→
&inc_dec_kp UP DOWN             // 正向=↑，反向=↓
```

本项目默认层的两个编码器：
```c
sensor-bindings = <&inc_dec_kp C_VOL_UP C_VOL_DN &inc_dec_kp LEFT RIGHT>;
//                 左编码器=音量                 右编码器=左右方向键
```
LOWER 层：
```c
sensor-bindings = <&BLE_encoder &inc_dec_kp UP DOWN>;
//                 左编码器=切蓝牙配置（自定义）   右编码器=上下
```

> 编码器绑定写在每个层的 `sensor-bindings` 里，而不是 `bindings` 里。没写的层（如 RAISE）会沿用默认层的编码器功能。

---

## 二、自定义行为（本项目 `behaviors { }` 块）

项目里一共定义了 6 个自定义行为，是理解这套键位的核心。

### 2.1 `hm` —— HomeRow 修饰键（hold-tap 模板）

```c
hm: homerow_mods {
    compatible = "zmk,behavior-hold-tap";   // 行为类型：按住=一个行为，单击=另一个行为
    #binding-cells = <2>;                   // 使用时要带 2 个参数
    tapping-term-ms = <500>;                // 超过 500ms 判定为"按住"
    bindings = <&kp &kp>;                   // <按住行为, 单击行为>，这里都设为按键
};
```

- 用法示例：`&hm LSHFT A` = **按住=Shift，单击=A**；`&hm LALT S` = 按住=Alt，单击=S。
- 这是典型"行内（Home Row）修饰键"方案：把 Shift/Ctrl/Alt 放到中间排，减少手指移动。
- ⚠️ **本项目目前定义了它但没有使用**——默认层字母键仍是普通 `&kp`。想启用可自行把字母键替换成 `&hm LSHFT A` 这类写法。

### 2.2 `LMouse_sc` —— 轻点左键（无长按功能）

```c
LMouse_sc: behavior_LMouse_Scroll {
    compatible = "zmk,behavior-hold-tap";
    #binding-cells = <2>;
    flavor = "tap-preferred";       // 倾向单击：轻点立即响应，长按才触发 hold
    tapping-term-ms = <200>;        // 200ms 阈值
    bindings = <&none &mkp>;        // <按住=nothing, 单击=鼠标键>
};
```

- 用法示例：`&LMouse_sc 0 MB1` → **单击=鼠标左键**，按住则**什么也不做**（`&none`）。
- 因为绑定里第一个参数给 `&none`，所以它"没有长按功能"，纯当一个左键使用。
- 本项目中位于 **MOUSE 层右拇指位**，保证手不离开指点设备也能单击。

> `flavor = "tap-preferred"`（单击优先）意味着：按键按下瞬间先当作单击执行，只有按住超过 200ms 才判定为 hold。适合"绝大多数时候只想单击"的键。

### 2.3 `M_kp` —— 鼠标键 + 字母 二合一

```c
M_kp: behavior_mouse_keypress {
    compatible = "zmk,behavior-hold-tap";
    #binding-cells = <2>;
    flavor = "tap-preferred";
    tapping-term-ms = <200>;
    bindings = <&mkp &kp>;          // <按住=鼠标键, 单击=普通键>
};
```

- 用法示例：`&M_kp MB3 U` → **单击=U，按住=鼠标中键**；`&M_kp MB1 J` → 单击=J，按住=左键。
- 这是 MOUSE 层的灵魂：**J/U/M 三个键既能打字、又能当鼠标按键**。
  - `&M_kp MB3 U`（J 键上方，右排）
  - `&M_kp MB1 J`
  - `&M_kp MB2 M`（M=右键）

### 2.4 `td0` —— 连击（Tap-Dance）：单击左键 / 双击清空蓝牙

```c
td0: tap_dance_0 {
    compatible = "zmk,behavior-tap-dance";
    #binding-cells = <0>;               // 不需要参数，直接 &td0 用
    tapping-term-ms = <400>;            // 400ms 内算"连续点击"
    bindings = <&mkp LCLK>, <&bt BT_CLR>;  // 第1击=左键；第2击=清空蓝牙配对
};
```

- **Tap-Dance**：按 1 次、2 次……触发不同的绑定。
- ⚠️ 本项目**定义了但没有引用它**（可作为双击清空配对的候选快捷键，需自己加上）。

### 2.5 `tdRMCLK` —— 单击右键 / 双击中键

```c
tdRMCLK: tap_dance_R_LMouse {
    compatible = "zmk,behavior-tap-dance";
    #binding-cells = <0>;
    tapping-term-ms = <400>;
    bindings = <&mkp RCLK>, <&mkp MCLK>;   // 第1击=右键；第2击=中键
};
```

- 使用位置：**MOUSE 层右拇指**。单击出右键菜单，快速双击滚屏中键，体验类似"笔记本触摸板两指点击"。

### 2.6 `BLE_encoder` —— 编码器切蓝牙配置

```c
BLE_encoder: BLE_encoder {
    compatible = "zmk,behavior-sensor-rotate";   // 传感器旋转行为
    #sensor-binding-cells = <0>;
    bindings = <&bt BT_NXT>, <&bt BT_PRV>;       // <顺时针, 逆时针>
};
```

- 给编码器用的**自定义旋转行为**：顺时针 = 下一个蓝牙配置，逆时针 = 上一个。
- 绑定在 LOWER 层的左编码器上。

---

## 三、宏（宏定义）的作用

```c
#define LOWER 1
#define RAISE 2
#define MOUSE 3
#define BOT bootloader
```

- 只是**给层号和常用键起别名**，让键位文件更好读、更好改。
- `&mo 1` 就是 `&mo LOWER`；`&BOT` 就是 `&bootloader`。
- 改层号/换行为时，只需改这里一处，不必全文件搜索。

---

## 四、行为与"输入处理器"（Input Processor）

除了按键行为，配置里还有一层**输入处理器**，作用在触摸板/小红点上（见 `left_bbtrackpad_keypoint.overlay`）：

```c
&bbtrackpad_listener {
    input-processors = <&zip_temp_layer 3 600>,  // ① 有输入时临时激活层3(MOUSE) 600ms
                       <&ip_behaviors0>, ...      // ② 把触摸板的物理按键事件转成方向键
};
```

| 处理器 | 作用 |
|---|---|
| `zmk,input-processor-temp-layer` | 收到输入事件时**临时激活某层一段时间**（这里是触摸板一动 → MOUSE 层生效 600ms） |
| `zmk,input-processor-behaviors` | 把底层输入事件（如 `INPUT_BTN_0`）**映射成某个行为/按键**（这里把触摸板按键映射成 `←→↓↑`） |

> 理解：行为处理"键按下去做什么"，输入处理器处理"指点设备（触摸板/小红点）的输入如何影响键盘"。两者配合实现"一碰触摸板，附近按键立刻变鼠标键"。

---

## 五、行为速查表（本项目）

| 写法 | 行为 | 出现位置 |
|---|---|---|
| `&kp X` | 普通按键 | 各层字母、数字、符号、F 键 |
| `&mt ESC TAB` | 按住=Esc / 单击=Tab | QWERTY 左上角 |
| `&mo 1` / `&mo 2` | LOWER / RAISE 瞬时层 | QWERTY 拇指区 |
| `&to 3` | 切到 MOUSE 层（常驻） | LOWER 右拇指 |
| `&trans` | 透传 | LOWER、RAISE |
| `&none` | 空 | RESERVE、RAISE |
| `&mkp LCLK/RCLK/...` | 鼠标左右中键 | 中央实体键、MOUSE 层 |
| `&msc SCRL_UP/DOWN` | 鼠标滚动 | 下排小键、MOUSE 层 |
| `&M_kp MB3 U` 等 | 单击=字母 / 按住=鼠标键 | MOUSE 层 J/U/M |
| `&LMouse_sc 0 MB1` | 单击=左键 | MOUSE 层右拇指 |
| `&tdRMCLK` | 单击=右键 / 双击=中键 | MOUSE 层右拇指 |
| `&bl BL_DEC/INC` | 背光减/增 | LOWER 层中央 |
| `&inc_dec_kp ...` | 编码器→按键 | 各层 sensor-bindings |
| `&BLE_encoder` | 编码器→切换蓝牙 | LOWER 层左编码器 |
| `&BOT`(`&bootloader`) | 进入刷机模式 | LOWER 层左/右拇指 |
| `&bt BT_CLR` | 清除配对 | LOWER 层左拇指 |
| `&out OUT_TOG` | USB/蓝牙切换 | LOWER 层右拇指 |
| `&td0` / `&hm` | 已定义但未使用 | （预留） |

---

## 六、常见问题

- **`&mt`、`&hm`、`&M_kp`、`&LMouse_sc` 都是"按住+单击"，有什么区别？**
  它们本质都是 **hold-tap** 行为，区别只是"按住"和"单击"各绑定什么：
  - `&mt`：按住=修饰键，单击=按键（内置，只能修饰键做 hold）
  - `&hm`：自定义，hold 和 tap 都设为按键（`&kp &kp`），可给任意键当修饰键
  - `&M_kp`：hold=鼠标键，tap=按键
  - `&LMouse_sc`：hold=无，tap=鼠标键

- **为什么 MOUSE 层一碰触摸板就激活？**
  是 `zip_temp_layer` 输入处理器在起作用（激活 600ms），不是键位问题。

- **`#binding-cells` 是什么？**
  表示"用这个行为时需要传几个参数"。比如 `&hm` 声明了 2 个，所以必须写 `&hm LSHFT A` 两个参数；`&td0` 声明 0 个，直接 `&td0` 即可。

- **为什么 RAISE 层很多 `&none`？**
  `&none` 是显式"屏蔽"，防止该位置误触发。想给这些位置加功能时，把 `&none` 换成实际行为即可。

---

*本手册基于 `config/zitaotech_keypoint.keymap` 编写；ZMK 版本对应 `west.yml` 中的 v0.3.0 分支。*
