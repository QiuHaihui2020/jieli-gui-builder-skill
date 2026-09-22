# 代码里怎么驱动 UI 控件

json 决定控件长什么样、在哪；运行时让它们动起来是 C 代码的事。这份文档讲两者怎么衔接。

官方接口文档：https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/extras/generated-interfaces.html

## 目录

- [先分清接口来自哪里](#先分清接口来自哪里)
- [控件在代码里叫什么](#控件在代码里叫什么)
- [页面生命周期回调](#页面生命周期回调)
- [线程模型：绝不能在别的线程碰控件](#线程模型绝不能在别的线程碰控件)
- [消息总线：业务侧怎么把数据送进 UI](#消息总线业务侧怎么把数据送进-ui)
- [按钮点击怎么接](#按钮点击怎么接)
- [自定义绘制](#自定义绘制)
  - [⚠ 绘制描述符的字段名和 style API 不一样](#-绘制描述符的字段名和-style-api-不一样)
- [运行时创建和替换控件的陷阱](#运行时创建和替换控件的陷阱)
- [全局状态开关：某个模式下功能失效](#全局状态开关某个模式下功能失效)
- [同一界面服务多个模式](#同一界面服务多个模式)
  - [同一条消息在不同音源下含义不同](#同一条消息在不同音源下含义不同)
- [送进 UI 的文本必须是 UTF-8](#送进-ui-的文本必须是-utf-8)
- [SDK 里「有实现但没人调用」的功能](#sdk-里有实现但没人调用的功能)
- [歌词类多行文本：别用 lv_lyrics](#歌词类多行文本别用-lv_lyrics)
- [无法外移的改动怎么保住](#无法外移的改动怎么保住)
- [批量改代码时的锚点陷阱](#批量改代码时的锚点陷阱)
- [排查清单](#排查清单)

## 先分清接口来自哪里

代码里会碰到两类接口，混淆了会找错文档：

| 来源 | 典型接口 | 在哪查 |
|---|---|---|
| GUI Builder 生成 | `setup_scr_<页面名>()`、`ui_get_scr()`、`res_event_<事件名>()`、`gui_scr_action_cb()`、`gui_msg_send()` | 官方「工具生成API接口」页 |
| SDK 运行时 | `lvgl_rpc_post_func()`、`lvgl_module_msg_get_ptr()` 等 | SDK 源码，官方 GUI Builder 文档里没有 |

官方文档只覆盖前者。**线程安全、异步投递这些官方一个字都没提**，但实际工程里绕不开，见下文。

## 控件在代码里叫什么

json 里的控件 `imgbtn_3`，在页面 `music_player` 下，生成代码里就是结构体成员：

```c
lv_ui_music_player *ui_scr = ui_get_scr_ptr(&guider_ui, GUI_SCREEN_MUSIC_PLAYER);
lv_obj_t *btn = ui_scr->music_player_imgbtn_3;      // <页面名>_<控件名>
```

取页面有两个容易混的接口，别用错：

| 接口 | 返回 | 用途 |
|---|---|---|
| `void *ui_get_scr_ptr(lv_ui *ui, int32_t scr_id)` | 该页面的**控件结构体**（`lv_ui_<页面名> *`） | 要访问具体控件时用这个 |
| `gui_scr_t *ui_get_scr(int32_t scr_id)` | 页面对象本身 | 页面级操作 |

访问控件用前者，声明在生成的 `gui_guider.h` 里。

命名规则是 `<页面名>_<控件名>`，改了 json 里的 `name` 这个成员名就跟着变，引用它的代码会编译不过——**改控件名前先 grep 一遍 `custom/` 目录**。

拿到指针后先判活，页面可能已经被切走销毁：

```c
if (!ui_scr || ui_scr->music_player_del) {
    return;
}
if (!lv_obj_is_valid(ui_scr->music_player_slider_1)) {
    return;
}
```

`<页面名>_del` 这个标志和 `lv_obj_is_valid()` 要一起用，前者判页面、后者判具体控件。

## 页面生命周期回调

页面被切入/切出时，生成代码会调这个弱函数（声明见官方「工具生成API接口」）：

```c
void gui_scr_action_cb(gui_screen_id_t id, gui_screen_action_t action);
```

在 `custom/` 里按页面实现，是**页面级初始化和清理的标准位置**：

```c
static int gui_src_action_music_player(int action)
{
    switch (action) {
    case GUI_SCREEN_ACTION_LOAD:
        /* 切进来时主动拉一次数据，否则界面要等下一次推送才有内容 */
        if (bt_get_total_connect_dev() > 0) {
            bt_cmd_prepare(USER_CTRL_AVCTP_OPID_GET_PLAY_TIME, 0, NULL);
            bt_cmd_prepare(USER_CTRL_BIP_GET_IMAGE, 0, NULL);
        }
        break;
    case GUI_SCREEN_ACTION_UNLOAD:
        /* 切走时停掉动画、释放这个页面占的资源 */
        logo_stop(NULL);
        music_player_song_info_lyrics_clean();
        break;
    }
}
```

两个实用要点：

**LOAD 里要主动拉数据。** UI 是被动接收推送的，如果数据源是事件驱动（蓝牙事件、周期上报），用户切进页面的那一刻界面可能还是空的或者是上一首的内容。在 LOAD 里主动请求一次能消除这个空窗。注意这类主动拉取往往带条件（上例只在蓝牙已连接时拉），**换成别的数据源时这个条件不会自动成立**——本地播放、网络播放都得各自补。

**UNLOAD 里必须释放。** 动画、定时器、解码器、临时 buffer 都要在这里停掉，否则切页面后它们还在后台跑，轻则耗电重则访问已销毁的控件崩溃。

## 线程模型：绝不能在别的线程碰控件

LVGL 不是线程安全的。蓝牙回调、音频回调、定时器回调都跑在各自的任务里，**在那里直接调 `lv_*` 函数会随机崩溃或花屏**，而且现象往往延迟出现，很难定位。

正确做法是把操作抛到 LVGL 线程：

```c
lvgl_rpc_post_func(ui_music_process_handler, 1, cur_time);   // 1 = 参数个数
lvgl_rpc_post_func(ui_music_album_pic_handler, 0);           // 0 = 无参数
```

判断标准很简单：**这个函数会不会被 LVGL 之外的线程调用**。会，就得走 `lvgl_rpc_post_func`。

## 消息总线：业务侧怎么把数据送进 UI

这类工程通常有一个字符串消息总线，业务模块不直接碰控件，而是发消息：

```c
bt_music_post_msg_to_ui("music_cur_time %4", time_ms);
```

函数名可能带 `bt_` 前缀，但**它往往是整个 UI 的消息总入口，不只服务蓝牙**。判断依据是看有多少模块在调它——音频后处理、本地播放、蓝牙都可能往里发。别被名字误导。

接收侧在 `custom.c` 里按字符串分发：

```c
if (strstr(msg, "music_tol_time")) {
    ui_music_tol_time_handler(va_arg(argptr, int));
} else if (strstr(msg, "music_cur_time")) {
    ui_music_cur_time_handler(va_arg(argptr, int));
}
```

用这套机制时注意三点：

**一、`strstr` 是子串匹配，消息名有包含关系会串味。** `"music_time"` 会被 `"music_tol_time"` 的分支抢先命中，取决于 if-else 的顺序。加新消息时挑一个不与现有消息互为子串的名字。

**二、参数类型必须和 `va_arg` 对齐。** 发送方写 `int`、接收方 `va_arg(argptr, char *)`，编译器一点提示都不会给，运行时直接跑飞。

**三、总入口常有守卫。** 比如：

```c
void bt_music_post_msg_to_ui(const char *msg, ...)
{
    if (!storage_device_ready()) {       // SD 卡没就绪就丢弃所有消息
        return;
    }
```

一旦守卫不成立，**所有** UI 更新都会静默失效，不只是和存储相关的那些。UI 整体不动时，先查这里。

### 数据不更新的排查顺序

界面某个数据不动，按这个顺序找，通常两三步就能定位：

1. **有没有人发这条消息**——grep 消息名，注意搜索范围要覆盖 `sdk/audio`、`sdk/apps/common` 这些容易漏的目录，别只搜业务目录
2. **发的单位对不对**——时间类尤其容易错，接收端可能按毫秒解析（`cur_time / 1000 / 60`）而发送端给的是秒
3. **依赖的前置数据有没有先发**——比如进度条百分比要用总时长做分母，总时长没发到，进度条就一直不动
4. **守卫有没有挡住**

真实例子：某播放器蓝牙播放时进度正常、SD 卡播放时不动。原因是时间数据只有蓝牙的 AVRCP 事件回调在发，本地播放模式没有等价事件源，得自己起个周期定时器上报。而同一个界面的频谱却两种模式都正常——因为频谱由音频后处理节点直接回调上报，和播放源无关。**同一个界面上的不同数据，来源可能完全不同，要分别追。**

## 模型绑定（bindMsg）

json 里控件带 `bindMsg` 时，它的内容由模型驱动，不要用 `lv_label_set_text()` 之类直接写——会被模型的下一次刷新覆盖。正确做法是更新模型：

```c
gui_msg_status_t gui_msg_send(int32_t msg_id, void *value, int32_t len);
```

工程里常见的是再包一层（比如 `lvgl_module_msg_get_ptr()` 取缓冲、填好内容再 `lvgl_module_msg_send_ptr()` 发出，发送成功后缓冲由内部释放，不用自己 free）。

看到某个控件的 json 里有 `bindMsg`，就说明它的内容归模型管，改 json 里的 `text` 只影响设计期预览。

## 按钮点击怎么接

在设计器里给控件绑定事件后，生成代码会产出弱符号空实现：

```c
GUI_WEAK void res_event_next_song(lv_event_t *e) { }
```

在 `custom/` 下写同名函数即可覆盖：

```c
void res_event_next_song(lv_event_t *e)
{
    app_send_message(APP_MSG_BT_MUSIC_NEXT, 0);
}
```

事件回调跑在 LVGL 线程，所以**回调里读写控件是安全的**，反过来往业务侧发消息要用业务自己的消息机制，别在这里做耗时操作——会卡住整个界面刷新。

## 自定义绘制

频谱、波形这类控件库没有的东西，靠在某个容器上挂绘制事件回调实现：

```c
lv_obj_add_event_cb(obj, my_draw_cb, LV_EVENT_DRAW_POST_BEGIN, NULL);

static void my_draw_cb(lv_event_t *e)
{
    if (lv_event_get_code(e) != LV_EVENT_DRAW_POST_BEGIN) {
        return;
    }
    lv_obj_t *obj = lv_event_get_target(e);
    lv_draw_ctx_t *draw_ctx = lv_event_get_draw_ctx(e);
    /* obj->coords 是屏幕绝对坐标，画之前先判 lv_obj_is_valid / lv_obj_is_visible */
    lv_draw_rect(draw_ctx, &dsc, &area);
}
```

要点：

- **绘制的位置和大小完全由那个容器决定**。容器在 json 里，所以挪动/缩放这块自绘内容是改 json，不是改代码。
- 数据变了要显式 `lv_obj_invalidate(obj)` 触发重绘，不会自动刷新。
- 回调里只做计算和绘制，别申请内存、别阻塞。
- 这类内容**在设计器预览里看不到**（预览不跑 C 代码），只能烧板子看。

### ⚠ 绘制描述符的字段名和 style API 不一样

这条必踩：给 `lv_draw_rect_dsc_t` 设渐变时，**不能照搬 style 属性的名字**。

| | style 属性（`lv_obj_set_style_*` / json） | `lv_draw_rect_dsc_t` |
|---|---|---|
| 渐变色 | `bg_grad_color` | `bg_grad.stops[1].color` |
| 渐变方向 | `bg_grad_dir` | `bg_grad.dir` |

draw dsc 里是一个 `lv_grad_dsc_t`，要填 stops 数组：

```c
dsc.bg_color = top_color;                 /* stops[0] 的别名 */
dsc.bg_grad.dir         = LV_GRAD_DIR_VER;
dsc.bg_grad.dither      = LV_DITHER_NONE;
dsc.bg_grad.stops_count = 2;
dsc.bg_grad.stops[0].color = top_color;  dsc.bg_grad.stops[0].frac = 0;
dsc.bg_grad.stops[1].color = bot_color;  dsc.bg_grad.stops[1].frac = 255;
```

注意 `LV_GRADIENT_MAX_STOPS` 常配成 2，想要三段色得先改 `lv_conf.h`。

写错的报错很直白（`no member named 'bg_grad_dir' in 'lv_draw_rect_dsc_t'`），但 **grep 到 API 声明存在并不能挡住它**——声明在不代表字段名对。涉及 draw dsc 的改动，直接去 `src/draw/lv_draw_rect.h` 读一眼结构体定义，比猜快。

更一般的教训：**静态检查代替不了编译**。改完让用户打包再编译，报错了再修，比在那儿 grep 半天可靠。

## 运行时创建和替换控件的陷阱

> 先看 SKILL.md 的「控件归属：工具建，代码喂」。本节讲的是**已经踩进去之后怎么识别**，
> 而不是在鼓励这种写法——绝大多数运行时创建都该搬回工具里。


有些代码会在运行时新建控件顶替 json 里定义的那个，例如收到网络图片后：

```c
lv_obj_t *img = lv_img_create(img_view);      // 父容器是某个 json 容器
lv_obj_align(img, LV_ALIGN_CENTER, 0, 0);     // 相对父容器居中
lv_obj_set_style_radius(img, 100, ...);       // 圆形裁剪
lv_obj_del(img_dst);                          // 删掉 json 里那个
*img_ptr_to_update = img;                     // 新对象接管原指针
```

这意味着：

- **json 里那个控件的位置尺寸只在被替换之前有效。** 在设计器里调得再准，运行时也会被代码里的 `align` 覆盖。
- **新控件的位置由父容器决定。** 如果你在 json 里把那个父容器挪走或改小，运行时的图就跟着跑偏——这种错在设计器预览里完全看不出来，只有烧板子才发现。
- 样式也可能被代码改写（上面那个 `radius=100` 会把方图裁成圆形）。

所以调整某块 UI 之前，**拿控件名去 `custom/` 目录 grep 一遍**，确认它是不是会被运行时接管。看到 `lv_obj_create` / `lv_img_create` 加上 `lv_obj_del(原控件)` 的组合，就说明这块归代码管，改 json 无效，要改代码里的坐标常量。

## 全局状态开关：某个模式下功能失效

"A 模式正常、B 模式不正常"的现象，很容易误判成数据源不同。但更常见的原因是**某个全局开关被别的页面改了、没人改回来**。

真实例子：某播放器蓝牙播放时频谱正常，SD 卡播放时不动。频谱数据挂在音频总输出上、与播放源无关，所以不是数据的问题。真正的原因是频谱运算有个 bypass 开关：

```c
// 视图切换时
切到歌词   → set_bypass(1)    // 关掉运算省 CPU
切回专辑   → set_bypass(0)

// 进入设置/文件列表页面时
gui_scr_action_cb(SYS_MENU, LOAD) → set_bypass(1)
```

播放页面的 LOAD 里**没有对应的 `set_bypass(0)`**。蓝牙是直接进播放页，开关保持默认的 0；SD 卡播放得先进文件列表选歌，那里把它关了，回到播放页没人再打开——于是只有 SD 卡模式看起来"坏了"。

排查这类问题的要点：

- **别只盯着数据源。** 先确认数据在不在（加打印或看已知正常的模式），数据没问题就去找开关。
- **grep 那个开关函数的所有调用点**，看是否"有人关、没人开"。这类 bypass / enable / suspend 开关是全局的，谁关谁负责开，但跨页面时经常漏。
- **在使用方的入口处主动置位**，而不是依赖别处的状态。上例的修法就是在播放页面的 LOAD 里主动 `set_bypass(0)`，不管之前是谁关的。

## 同一界面服务多个模式

一个播放界面常被蓝牙、本地文件、网络电台共用。**控件是同一套，但底层命令不同**：

```c
void res_event_next_song(lv_event_t *e)
{
    if (get_current_app_mode() == APP_MODE_LOCAL) {
        app_send_message(APP_MSG_LOCAL_MUSIC_NEXT, 0);
    } else {
        app_send_message(APP_MSG_BT_MUSIC_NEXT, 0);
    }
}
```

生成的事件回调默认往往只发一种模式的消息（哪个模式先做的就发哪个），换个模式点按钮完全没反应。**接手这类界面时，把每个按钮回调都对照模式检查一遍**，比等用户报"某某模式下按钮没用"快得多。

数据上报也是同理：蓝牙有 AVRCP 事件驱动，本地播放没有等价事件源，得自己补一路周期上报。

### 同一条消息在不同音源下含义不同

比按钮回调更阴的是**消息本身换了意思**。实测过的例子：`music_lyrics` 这条消息，

- **蓝牙**：SDK 把 AVRCP 的标题（type==1）当成歌词发出来，它其实是**歌名**。蓝牙根本没有歌词
- **本地**：送的才是 `.lrc` 里的**真歌词**

分发函数里不判音源，两边就会串台——蓝牙下歌词区显示歌名，本地下歌名跟着歌词一行行变：

```c
} else if (strstr(msg, "music_lyrics")) {
    char *title = va_arg(argptr, char *);
    if (ui_music_is_bt_source()) {
        ui_music_song_text_update(title, NULL);   /* 蓝牙：这是歌名 */
    } else {
        ui_music_lyrics_handler(title);           /* 本地：这才是歌词 */
    }
}
```

同一类陷阱还有 AVRCP 的 artist 字段（type==2）：对端**可能**发 `"歌手/歌名"` 合并串，也可能只发歌手名。按 `/` 拆出来的那两半，没有斜杠时**整串都是歌手**——此时若把拆出来的"后半段"当歌名推进模型，就会看到"歌名位置显示歌手名"。

**改动态文本的数据链之前，先把每条消息在每种音源下到底装的是什么搞清楚**，别看变量名叫 `title` 就当它是歌名。串口打一行原始内容最省事。

## 送进 UI 的文本必须是 UTF-8

LVGL 一律按 UTF-8 解析字符串，再按 Unicode 码点去字库取字模。可 SDK 底层的文本来源**各说各话**，直接 `post` 给 UI 大概率只能显示英文部分：

| 来源 | 实际编码 | 怎么转 |
|---|---|---|
| FAT 长文件名 | Unicode(u16)，缓冲区以 `0x5C 'U'`（反斜杠+U）开头 | `Unicode2UTF8()` 跳过头两字节 |
| ID3v1 标签 | 无编码标识，中文通常是 GBK | SDK 没有 GBK→UTF-8 的表，只能只认 ASCII、其余回退文件名 |
| ID3v2 标签 | 帧头有编码字节（0=ISO8859-1，1=UTF-16，3=UTF-8） | 按编码字节分支 |
| LRC 歌词 | 看 `LRC_INFO.coding_type`（GBK/UTF16LE/UTF16BE/UTF8） | UTF-8 与 UTF-16 源已被 SDK 统一转成 Unicode，用 `unicode_2_utf8()` 转回；GBK 源没救 |
| 蓝牙 AVRCP | 协议规定 UTF-8 | 直接用 |

转换函数都在 `sdk/include_lib/utils/character_coding.h`：`utf8_2_unicode` / `unicode_2_utf8` / `utf16_to_utf8` / `utf8_to_utf16`。注意**只有 utf8→gbk 方向存在，反方向没有**，所以 GBK 源的中文在这套 SDK 上无解，只能要求资源文件存成 UTF-8。

⚠ **`unicode_2_utf8()` 只管写字节，不补 `'\0'`。** 复用同一个缓冲逐句转换时，上一句比这一句长的那截会留在后面，被当成本句的一部分显示出来——现象是某一行后半段乱码，或者一行里粘着半句别的内容。**每次转换前把整块缓冲 memset 清零**，不是"保险起见"，是必须的：

```c
memset(out, 0, out_size);        /* 必须：unicode_2_utf8() 不负责补结束符 */
unicode_2_utf8((u8 *)out, (int)(out_size - 1), src, (int)len);
```

留出最后一个字节不写（传 `out_size - 1`），清零之后那个 0 就是兜底的结束符。

判断某个中间层到底存了什么编码，不用猜也不用等烧录——**看库引用了哪些符号**：

```bash
llvm-ar x common_lib.a && llvm-nm lyrics.c.o | grep " U "
# 出现 utf8_2_unicode / get_utf8_size → 这个模块内部把文本转成了 Unicode
```

只有英文能显示、中文变成空白或方块，还有另一个原因：**字库里没有那个字**。位图字体只包含设计期扫描到的字符，运行时才知道内容的文本（歌名、歌词）必须用矢量字体（ttf/otf）。见 SKILL.md 的「动态中文必须矢量字体」。

## SDK 里「有实现但没人调用」的功能

杰理 SDK 里有一批功能是**代码齐全、宏也默认打开，但全工程没有任何调用方**——歌词解析就是典型：`TCFG_LRC_LYRICS_ENABLE` 是 1，`music_player_lrc_analy_start()` 实现完整，`lyric_init()` 在播放器创建时就跑了，可是没人调 analy_start，所以 `analysis_flag` 永远是 0，歌词永远取不到。

这类功能表现出来就是"这个特性好像不支持"，实际只差一次调用。**在断定某个能力不存在之前，先反查调用方**：

```bash
grep -rn "music_player_lrc_analy_start" sdk --include=*.c   # 只有定义，没有调用 → 死代码
```

接进 LVGL 时还要注意：SDK 自带的展示接口往往是写给**老 UI 框架**的（参数是控件 ID，如 `lrc_show_api(lrc, text_id, ...)`），LVGL 用不上。要找同一模块里"只取数据不画"的那个兄弟接口（`lrc_get_api()`），拿到数据自己 post 给 UI。

驱动节奏也得自己补：歌词按播放秒数推进，就挂在已有的进度轮询里，每秒调一次 `lrc_get_api(lrc, sec, 0)`。注意它的返回值是"这个时刻能对上某条时间标签"，**不是"这一拍有新词"**——同一句在它的持续时间内每拍都会返回 TRUE 并重复吐出同样的文本。要不要刷 UI 得自己比对，别每拍都 `set_text`。

## 歌词类多行文本：别用 lv_lyrics

`lv_lyrics` 是杰理的 GPU 特效控件（`lv_lyrics_create/set_zoom/set_rotation`），一个对象一张 GPU 纹理，适合单行大字加变换动画。拿它做常规多行歌词有两个坑：

- 每来一句都得销毁重建对象，配上示例自带的「缩放+下落」特效（`lyrics_anim_effect_zooming_in_down`），看起来就是**歌词一直上下浮动**
- 它没有对齐、省略、滚动这些现成能力，折行位置要自己按字宽估算（示例里那一整套 `get_char_width_coefficient` / `find_optimal_split_point` 就是干这个的），估不准就溢出容器

想要"上一句/当前句/下一句"这种固定窗口，用 **`lv_label` + 矢量字体**：建好固定行数的 label 常驻，新歌词来了只 `lv_label_set_text` + 改颜色，**控件位置一个都不动**——位置一动就会看到抖动。长句交给 `LV_LABEL_LONG_SCROLL_CIRCULAR`，旧句用 `LV_LABEL_LONG_DOT`，三行一起滚会很乱。

容器换页会销毁子控件，所以缓存的 label 指针要两道保险：`LV_EVENT_DELETE` 回调里置空，再比对 `parent` 指针是否还是当前页的容器，变了就整组重建。

⚠ 动手删 `custom/` 里的"示例函数"之前，**先跨文件 grep 一遍**。这些示例文件互相借用得很随意——歌词模块的 `create_lyrics_line()` 就被歌名模块 `ui_action_song_info.c` 直接调用，删掉编译期没事，链接才报 undefined reference。

## 无法外移的改动怎么保住

工具会覆盖的目录里，有些改动**没法挪到独立文件**——改的是现有函数内部的逻辑（算法、坐标常量、分支条件），而原代码没留 weak 函数或回调之类的扩展点。硬加扩展点等于还是改了原文件，更绕。

这种情况下务实的做法是**让改动可重放**：

1. 把改好的文件在版本控制外留一份「黄金副本」（例如 `tools/ui_patch/golden/`）
2. 写个幂等脚本：检测目标文件里有没有改动的特征串（某个自定义宏名），没有就从黄金副本恢复、并备份当前版本
3. 每次工具覆盖之后跑一次

关键是特征串要选**只有你的改动才有**的标识（自定义宏名最合适），这样脚本能准确判断"改动还在不在"，而不是盲目覆盖——盲目覆盖会把工具带来的正当更新也冲掉。



界面不对时按这个顺序过一遍：

| 现象 | 先查 |
|---|---|
| 整个界面都不更新 | 消息总入口的守卫条件 |
| 某个数据不更新 | 有没有发送方（搜索范围要够广）、单位、前置数据 |
| 改了 json 但运行时位置不对 | 该控件是否被代码运行时替换/重定位 |
| 自绘内容位置不对 | 承载它的容器在 json 里的位置尺寸 |
| 随机崩溃或花屏 | 是否在非 LVGL 线程直接调了 `lv_*` |
| 改控件名后编译不过 | `custom/` 里对 `<页面名>_<控件名>` 的引用 |
