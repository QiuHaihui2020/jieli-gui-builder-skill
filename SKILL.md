---
name: jieli-gui-layout
description: 杰理 JLGuiBuilder（LVGL v8 方案）的界面布局编辑——直接改工程的 json 设计文件来调控件坐标、尺寸、文字、样式、层级。当用户要挪动/对齐/缩放某个控件、改按钮或标签位置、调整界面布局、修改 ui_prj 下的界面，或提到 imgbtn/lbl/slider/view 这类控件名时使用，即使没明说"json"或"GUI Builder"。也覆盖代码侧怎么驱动这些控件（取控件指针、UI 消息总线、线程约束、事件回调、自绘内容），以及改完必须走的打包+编译流程——漏掉这步改动进不了固件，是本工具最容易踩的坑。认准 LVGL 这套：工程在 ui_prj/<名字>/jlui/design/ 下，生成 setup_scr_*.c 和 lv_obj_* 代码，资源打包成 UIPACKRES。不适用于走 ui_framework + UITools 的 .uiproj / LCD_UI工程（产物是 JL.sty/JL.res/JL.str），彩屏那套看 jl-lcd-ui、单色点阵看 jl-dot-ui。
---

# 杰理 JLGuiBuilder 布局编辑

适用范围：**杰理走 LVGL v8 + JLGuiBuilder 的彩屏 UI**。杰理有三套并存的 UI 方案，动手前先认准是哪一套，认错了改的文件根本不参与编译：

| 特征 | 本 skill（LVGL） | jl-lcd-ui | jl-dot-ui |
|---|---|---|---|
| 框架 | LVGL v8 | ui_framework + IMB 硬件合成 | ui_framework 软件 framebuffer |
| 工程位置 | `ui_prj/<名字>/jlui/design/` | `cpu/<芯片>/tools/LCD_UI工程/` | `.uiproj` |
| 生成产物 | `setup_scr_*.c`（`lv_obj_*` 调用） | `JL.sty` / `JL.res` / `JL.str` | 同左 |
| 资源包 | `UIPACKRES` | `JL.res` | 同左 |
| 屏幕 | 彩屏 | 彩屏 OSD16 | 单色点阵 OSD1 |

最快的判据：看工程里有没有 `ui_prj/<名字>/jlui/design/ui/*.json`，以及生成的代码里是不是 `lv_obj_set_pos()` 这类 LVGL 调用。是就用本 skill，不是就去那两个。

确认是这一套之后：与其让用户打开 GUI Builder 手工拖控件，不如直接改 json 设计文件——精确、可批量、可复核。下面讲怎么安全地改。

## 工作循环

代码生成和资源打包没有命令行接口，只能在 GUI Builder 里点。所以循环是「AI 改文件 + 人点打包 + 人编译下载」：

```
AI 做 ──┬─ ① 改 json         ui_prj/<工程>/jlui/design/default/ui/<页>.json
        │        ⚠ left/top 改完必须重算整棵树的 absLeft/absTop，否则工具渲染不动
        │        ⚠ Img 还要同步 showLeft/showTop（生成代码取的是这对）
        │
        └─ ② 改 custom 代码  ui_prj/<工程>/custom/*.c  ← 不是 sdk 那份！sdk 侧是副本
                 ⚠ 改完就停，不要手工拷到 sdk 侧
                             ↓
用户做 ─┬─ ③ 工具端：重新打开工程 → 点「打包」
        │        ⚠ 工具不监听 ui/ 目录，不重开工程就读不到 AI 改的 json
        │        重新生成 setup_scr_*.c + 导出资源 + 跑 build_script
        │        ⚠ 这步会把 ui_prj/custom/ 覆盖到 sdk 侧，ui_res/ 也清空重建
        │
        └─ ④ 代码端：编译 → 下载
                 ⚠ 顺序不能反：先编译后打包的话，编译吃的是旧代码
                             ↓
                    ⑤ 看串口验证
```

**③④ 顺序错了是最常见的"改了没反应"。** 判断哪一环断了，看时间戳是否递增：

```
<页>.json  →  setup_scr_<页>.c  →  setup_scr_<页>.c.o  →  app.bin
```

哪一环倒挂，问题就在那一环。

## 铁律

"不知道就会做错"的硬规则，每条都是实战踩出来的。详细展开见后文对应章节。

**1. 改 `custom/` 代码只改 `ui_prj` 那份，改完就停，不要手工拷到 sdk 侧。** `sdk/apps/<应用>/lvgl_v8_ui_app/<风格>/custom/` 是副本，同步由用户点「打包」完成，那一步会把 `ui_prj/<工程>/custom/` 整个覆盖过去。改错地方的症状是"改动莫名消失"，且 sdk 侧文件的 mtime 会诡异地退回到很早以前；而手工 `cp` 过去则会让两侧分不清谁是人改的、谁是工具写的。

**2. 外部改完 json 后，必须让工具重新加载工程，再打包。** 工具只在**打开工程时**把 `ui/*.json` 读进内存，之后**不监听这个目录**（全工程只有两个文件监听器，盯的是 `timeline/` 和 `hardware.json`）。打包时 `ui.json` 是从内存模型生成的，所以工具没重新加载的话，**你改的东西根本不会进固件，而且不报错**。更糟的是反方向：工具里的旧模型可能把你改的页面 json 覆盖回去。校验方法——看 `jlui/design/default/ui.json` 的 mtime 是否晚于你改的 `ui/<页>.json`，晚才算数。

**3. 工具能建的控件，绝不在代码里手写创建；代码只喂内容和数据。** 位置、尺寸、字体、字号、颜色、对齐、长文本策略（`long_mode`）全都是 json 字段。代码里再写一份就有了两处真相，改布局得同步改两处，漏一处就错位。正确形态就是一行 `lv_label_set_text(ui_scr-><页面>_<控件>, text)`。三种错法和三个合法例外见「控件归属：工具建，代码喂」一节。

**4. `absLeft/absTop` 必须和 `left/top` 一起改。** **工具渲染用的是 `absLeft/absTop`**。只改 `left/top` 的症状很迷惑：`width/height` 生效了，位置却纹丝不动。改完固定跑一次递归重算。

**5. Img 控件的三套坐标别记混。** `left/top` 是数据源，`absLeft/absTop` 给工具渲染，`showLeft/showTop` 给生成的 C 代码。三个都要对。

**6. Img 控件尺寸必须等于图片实际尺寸。** LVGL 在控件比图小的时候是**裁切**不是缩放，`zoom` 字段在这条路径上不起作用。要小图标就先把 PNG 缩放成目标尺寸存成新文件。

**7. UI 图标必须是透明底 RGBA，且要矢量重绘，不许从设计稿抠图。** 抠出来的图带着卡片底色（`mode=RGB`），换个背景就露出一圈方块；位图缩到十几像素又必然发糊带毛边。对照工程里的 `ic_*` 那种 RGBA 图标，别学 `btn_*`/`cut_*` 那种实底抠图。做法见「图标贴图」一节。

**8. 底图里不许留任何控件、按钮、文字。** 只留背景、卡片、分隔线这类纯装饰。留了就会和贴上去的控件叠成双影——远看只是"有点糊"，放大 4 倍才看得出是两层。做完底图单独看一眼，还能认出图标或文字就是没抹干净。

**9. 文字一律走 Label 控件，绝不渲染成 PNG。** 文字进了图片就改不了文案、做不了多语言、没法跟状态变色，改个字号还得重做图。

**10. 动态中文文本只能用矢量字体。** 点阵字库只含打包时扫描到的字符，运行时才知道的中文必然缺字（串口刷 `glyph dsc. not found`）。

**11. 多行歌词用 `lv_label` + 矢量字体，不要用 `lv_lyrics`。** `lv_lyrics` 是 GPU 特效控件，一个对象一张纹理，每来一句都要销毁重建，配上示例自带的缩放下落特效就是"歌词一直上下浮动"；而且没有对齐/省略/滚动能力，折行得自己按字宽估。正确做法是固定行数的 label 常驻，新词只 `set_text` + 改颜色，控件位置一个都不动。展开见 `references/code-integration.md` 的「歌词类多行文本」。

**12. `ui_res/` 和 `generated/` 是工具产物，往里放文件必丢。** 手工资源放一个工具不认识的新目录，再从 build_script 拷过去。

**13. 改 `.bat` 必须保持 CRLF，注释别用中文。** LF 会让 cmd 解析多行结构错乱；中文注释在 UTF-8 代码页下可能被字节错位拆成命令。

**14. 送进 UI 的文本必须先转成 UTF-8。** 文件名是 Unicode、ID3v1 是 GBK、LRC 看 `coding_type`——各不相同。不转的症状是"只显示英文部分"。转换函数见 `character_coding.h`，细节见 `code-integration.md`。

**15. 断定"SDK 不支持某功能"之前，先反查调用方。** 杰理 SDK 里有成片的功能实现完整、宏也是打开的，但全工程没人调用（歌词解析就是），表现得像不支持，其实只差一次调用。

**16. 脚本批量改代码时，替换锚点必须唯一。** 先 `assert count == 1` 再替换，改完验证落点在哪个函数。插错函数的症状是"编译通过、语法没问题、运行时完全不执行、日志里连报错都没有"。

## 为什么要小心

这套工具有几个反直觉的地方，不了解就会改了没效果，或者改出错位：

- 同一份坐标数据在工程里存了**两份**，但只有一份是你该动的
- **Img 控件的 `left/top` 不是它的渲染位置**，改了不生效
- 改完 json 要先在工具里「打包」，再回工程编译；顺序反了编译吃的是旧代码

下面逐个说清楚。

## 最要命的一条：custom 代码的源在 ui_prj，不在 sdk

改这类工程的 C 代码之前必须先搞清楚：**`sdk/apps/<应用>/lvgl_v8_ui_app/<风格>/custom/` 下的文件是副本，不是源。**

源在 UI 工程里：`ui_prj/<工程>/custom/`。GUI Builder **每次「打包」都会把 `ui_prj/<工程>/custom/` 整个拷过去覆盖 sdk 侧**，连同 `generated/` 一起。改了 sdk 侧那份，打包一次就没了，而且不会有任何提示。

判断方法——对比两侧文件，内容完全相同、且 sdk 侧的时间戳诡异地"退回"到很早以前（其实那是 ui_prj 侧文件的原始 mtime），就是被覆盖了：

```bash
diff -r ui_prj/<工程>/custom/ sdk/apps/<应用>/lvgl_v8_ui_app/<风格>/custom/
```

所以：

| 代码位置 | 改哪份 |
|---|---|
| `<风格>/custom/` 下的 UI 逻辑 | 改 `ui_prj/<工程>/custom/`，打包后自动同步到 sdk |
| `<风格>/generated/` 下的生成代码 | 都别改，打包重新生成 |
| 应用其它目录（`mode/`、`apps/common/` 等） | 直接改 sdk 侧，不受打包影响 |

**绝不要手工往 sdk 侧拷贝。** 同步是「打包」这一步的职责，人点一下工具就完成了。手工 `cp` 过去有两个坏处：一是 sdk 侧凭空多出一份来源不明、随时会被打包覆盖的修改，git 上看是"改了但下次打包就没了"；二是两侧内容一旦不同，就再也分不清哪份是人改的、哪份是工具写的。

改完 `ui_prj` 侧就停下，让用户去点打包。想验证改动落地，看时间戳递增即可，别自己动手补这一步。

同一个应用里的 C 文件，有的受这个机制影响、有的不受，取决于它在不在 `<风格>/` 目录下——这也是最迷惑的地方：同一次改动，几个文件保住了、另几个没了。

## 第一步：定位文件

杰理 UI 工程的根目录形如 `ui_prj/<工程名>/`（例如 `ui_prj/wifi_soundbox_800x480/`），这也是 GUI Builder 打开工程时选的目录。界面数据在：

```
ui_prj/<工程名>/jlui/design/default/
├── ui/<页面名>.json     ← 只改这里。每个页面一个文件，控件的权威数据
├── ui.json              ← 不要改。编译时工具自己重写的快照
└── scene.json           ← 不要改。只是画布缩放和页面缩略图的摆放位置
```

判断依据：`ui.json` 的修改时间和生成的 `setup_scr_*.c` 完全相同（同一秒），说明它是工具编译时的**输出**而非输入；而 `ui/<页面名>.json` 的时间跟着你在设计器里的操作走。工具读 `ui/<页面名>.json`，产出 `ui.json` 和 C 代码。

`ui.json` 里 `screen[].widgets[]` 确实也有 `pos`/`size`，看着像坐标源，但那是上次编译的快照。改它没有任何作用，下次编译会被覆盖。

不确定页面名时，列一下 `ui/` 目录即可；页面名和生成的 `setup_scr_<页面名>.c` 一一对应。

## 先分清：哪些布局归 json 管

动手之前务必确认目标元素真的是 json 控件。这类工程里，界面元素通常有三个出处，只有第一种能靠改 json 解决：

| 出处 | 特征 | 怎么改 |
|---|---|---|
| json 控件 | 在 `ui/<页面>.json` 的 `children` 里找得到 | 改 json，本 skill 的范围 |
| custom 代码动态创建 | json 里没有，`custom/` 目录的 C 文件里 `lv_obj_create` / `lv_label_create` 出来的，位置常写死成宏 | 改 C 宏或代码 |
| custom 代码直接绘制 | 连控件都不是，在某个容器的绘制事件里用 `lv_draw_*` 画出来 | 改绘制逻辑 |

举个实际例子：某个播放器界面看着有封面、歌名、歌手、歌词、频谱五块，实际上只有封面是 json 控件（`Img`），歌名歌手是 `ui_action_song_info.c` 里 `SONG_NAME_Y_POS 394` 这样的宏定死的，歌词在 `ui_action_music_lyrics.c` 的 `LYRIC_START_Y 147`，频谱则是在容器的 `LV_EVENT_DRAW_POST_BEGIN` 里用 `lv_draw_polygon` 逐帧画的——后面四块改 json 一点用都没有。

快速甄别：拿控件名去 `custom/` 目录 grep。json 里有的控件，生成代码里叫 `ui_scr-><页面名>_<控件名>`；grep 不到控件名却在界面上看得见的东西，基本就是代码画的。

这一步花一分钟，能避免改完一堆 json 发现界面纹丝不动。

> ⚠ **一旦判定某块归代码管，改它之前必须先读 `references/code-integration.md` 对应小节**，别照着现有代码调参数就算完。那份文档里记着一批"现有代码本身就是错的"的结论——比如歌词在用 `lv_lyrics` 就是错的（该用 `lv_label`）、自绘控件被代码运行时替换掉、示例函数被跨文件借用不能删。照着错的代码改，只会把错的做法延续下去。

## 控件归属：工具建，代码喂

上一节讲的是"现状归谁管"。这一节讲**应该归谁管**——这两件事经常不一致，而且不一致的那部分基本都是历史遗留，该往回搬。

**原则：凡是工具控件库里有的控件，一律在工具里建；代码只负责喂内容和数据。**

工具里能配的东西比想象的多：位置、尺寸、字体、字号、颜色、对齐、字间距、行间距、长文本策略（`long_mode`）、可见性、绘制层级、事件绑定、模型绑定。这些在代码里再写一份，就有了两处真相——改布局要同步改两处，漏一处就错位，而且**设计器预览和真机不一致**，排查时会以为是渲染问题。

理想形态就是一行：

```c
lv_label_set_text(ui_scr->music_player_lbl_song, name);
```

### 三种错法

都在同一个播放器界面里真实出现过：

**① 坐标写死成宏。** 歌名歌手用 `lv_lyrics_create()` 直接画在 screen 上，位置靠 `SONG_NAME_Y_POS 394` 这类宏。json 里明明有 `lbl_song`，还绑好了模型，却被代码另画一套盖在上面，两份内容重叠显示。

**② 新建控件顶替 json 控件。** 收到蓝牙专辑图后 `lv_img_create()` 新建一张，`lv_obj_set_pos()` 用宏定位，再 `lv_obj_del()` 删掉 json 的 `img_1`，最后把页面结构体里的指针改指向新对象。封面每换一首歌就更新一次，等于反复销毁重建控件。正确做法是 `lv_img_set_src(ui_scr-><页>_img_1, dsc)`，位置尺寸圆角全沿用 json。

**③ 容器里手工建一堆子控件。** 三行歌词在 `view_lyrics` 容器里用 `lv_label_create()` 建出来，行高、字号、颜色、对齐全在代码里。其实这三行角色固定（上一句/当前句/下一句），颜色不随滚动变，完全可以是三个 json Label，代码只 `set_text`。

共同特征：**json 里有个同名控件闲置着，或者有个空容器**。看到这种就是该往回搬的信号。

### 三个合法例外

不是所有东西都能搬，以下三类留在代码里是对的，但要在代码里写清楚"为什么 json 做不到"：

| 例外 | 例子 | 原因 |
|---|---|---|
| json 没暴露的样式字段 | Slider 的 `LV_PART_KNOB` 描边 | 工具的样式面板只有 `bg_*`/`radius`，没有 `border_*` |
| 控件库里没有的东西 | 频谱、波形 | 靠容器上挂 `LV_EVENT_DRAW_POST_BEGIN` 自绘，容器本身仍然是 json 控件 |
| 运行时才能确定的值 | 解码图缩放比 | 图片尺寸随歌曲变，json 不可能预知；但缩放基准要取 `lv_obj_get_width(控件)` 而不是写死 |

判据很简单：**先去 json 里找这个字段，找不到才写代码**。查法见「第四步：改文字和样式」里那段。

### 字体是最容易误判的一项

"动态中文必须矢量字体、所以字体只能代码设"——这个推论**不成立**。工具的 `项目 - 属性 - 资源配置 - 字体 - 压缩方式` 可以整个工程切到矢量：

```
矢量字体 - 源文件(lv_tiny_ttf)
矢量字体 - 源文件(lv_ft_font)     ← FreeType，和代码手动 lv_ft_font_init 同一条路
点阵字体 - Fnt(lv_font_t)
点阵字体 - Bin(lv_font_t)         ← 默认
```

切到 `lv_ft_font` 后，生成的 `gui_fonts.c` 变成 `lv_ft_info_t` + `weight = <json 里的字号>` + `lv_ft_font_init()`，**json 的字体和字号直接生效、中文不缺字**，代码里那套 `ui_music_vector_font_get()` 覆盖字体的东西可以整个删掉。

代价是资源体积：矢量模式下工具会把**每个被引用的 ttf 源文件**打进资源区。混用三种字体就是三份 ttf，原版中文字体动辄 3–9 MB，而 UI 资源分区常常只有 2 MB（看 `isd_config.ini` 的 `UIPACKRES_LEN`）。所以切之前先做两件事：

1. 用 `fontTools` 把中文字体裁到 GB2312 一级（3755 字约 1 MB），放进 `import/font/`，工具的字体下拉框才会列出它
2. 把**所有页面**的 `font_family` 统一成这一份（图标字体如 FontAwesome 除外）。漏一个控件，那份完整 ttf 就会被打进去

改 `font_family` 用脚本扫全部页面 json 最稳，比在工具里逐个点可靠。改完核对 `soundbox_ui_res/`（或对应的打包暂存目录）里有几个 `.ttf`、总大小有没有超 `UIPACKRES_LEN`。

> ⚠ 切矢量后，之前为了绕开点阵字库而手工塞进 build_script 拷贝目录（如 `extra_res/`）的 ttf **要删掉**。工具自己会按资源 ID 生成字体文件，手工那份如果撞上同名（`30000000.ttf` 这种），会在 build_script 里**覆盖掉工具的产物**——代码以为这个 ID 是 A 字体，实际是 B 字体，症状是字形不对但又不报错。

## 第二步：读懂结构

文件顶层就是 Screen 节点，控件挂在 `children` 里，容器控件的 `children` 再往下递归：

```
Screen  music_player  800x480
├── ImgButton  imgbtn_1    (369,399) 64x64
├── Slider     slider_1    (13,333)  751x13
├── Label      lbl_3       (680,353) 103x49
└── View       view_1      (0,0)     800x321
    └── Img    img_1       (300,61)  200x200     ← 坐标相对 view_1，不是相对屏幕
```

每个控件节点的关键字段：

| 字段 | 含义 |
|---|---|
| `id` | `#` + 16 位随机串，工程内唯一。改布局时不要动 |
| `type` | 控件类型：`Screen` `View` `Img` `ImgButton` `Label` `Button` `Slider` `Switch` `ComboBox` `Timer` |
| `name` | 控件名，生成代码里变成 `ui_scr-><页面名>_<name>`。必须字母开头、至少 3 字符、只含字母数字下划线 |
| `left` `top` | **相对父容器**的坐标 |
| `width` `height` | 尺寸 |
| `children` | 子控件数组，只有容器类才有 |
| `style` | 样式数组，见下文 |
| `visible` | 是否显示 |

`children` 数组的顺序就是**绘制顺序**，后面的盖在前面的上面。要调整遮挡关系，移动数组元素的位置即可。

## 第三步：改坐标和尺寸

### 普通控件

直接改 `left` / `top` / `width` / `height`。注意坐标相对父容器：`view_1` 在 (0,0)、`img_1` 在 view_1 内 (300,61)，那么 img_1 在屏幕上就是 (300,61)；如果 view_1 挪到 (0,50)，img_1 的 `left/top` 不用动，它会跟着走。

### Img 控件要额外改 showLeft / showTop

这是最容易踩的坑。Img 类型除了 `left/top`，还有一对 `showLeft/showTop`，**生成代码用的是后者**：

```json
{
  "type": "Img",  "name": "img_2",
  "left": 698,    "top": 349,        // 设计框位置
  "showLeft": 699,"showTop": 350,    // ← 实际渲染位置，代码取这个
  "offsetX": -1,  "offsetY": -1
}
```

关系是 `showLeft = left - offsetX`。上面 `698 - (-1) = 699`，生成的代码里确实是 `lv_obj_set_pos(..., 699, 350)`。

已确认的事实是：生成的 C 代码里出现的是 `showLeft/showTop` 的值。至于只改 `left/top` 而不动 `showLeft/showTop` 时，工具重新生成代码会不会用 `left - offsetX` 把 `showLeft` 重算回来，没有实测过。

所以稳妥的做法是**两个一起改**，保持 `offsetX/offsetY` 的差值不变——无论工具内部取哪个字段，结果都对。只改一个则要看运气。

其它控件类型没有这对字段，改 `left/top` 就够了。

### absLeft / absTop 必须一起改——这是最容易翻车的地方

每个控件还有一对 `absLeft/absTop`，是它在**屏幕上的绝对坐标**。

**工具渲染界面时用的是 `absLeft/absTop`，不是 `left/top`。** 只改 `left/top` 不改它们，在 GUI Builder 里打开会看到控件**纹丝不动**，还停在原位；而同时改的 `width/height` 却生效了——这种"改了一半"的诡异现象就是漏改 `abs` 造成的。

更坑的是，`absLeft/absTop` 有时和 `left/top` **对不上**（实测某个嵌套控件 `left/top=(300,61)` 而 `absLeft/absTop=(-100,-99)`），看着像脏数据，很容易误判成可以忽略。实际上工具认的就是这个值，漏改的话控件会跑到屏幕外去。

正确做法：改完 `left/top` 后**重算整棵控件树**的绝对坐标。

```python
def fix_abs(node, ax=0, ay=0):
    for c in (node.get('children') or []):
        nx, ny = ax + c['left'], ay + c['top']      # 父绝对坐标 + 自身相对坐标
        c['absLeft'], c['absTop'] = nx, ny
        if c.get('type') == 'Img':                   # Img 的渲染坐标和 left/top 同坐标系
            c['showLeft'] = c['left'] - c.get('offsetX', 0)
            c['showTop']  = c['top']  - c.get('offsetY', 0)
        fix_abs(c, nx, ny)
fix_abs(screen_root)
```

递归重算比逐个手改可靠——移动过层级的控件、容器里的子控件都能一次性对齐。改完布局**固定跑一次**这个函数，当作收尾步骤。

三套坐标的关系理一下，别记混：

| 字段 | 坐标系 | 谁在用 |
|---|---|---|
| `left` `top` | 相对父容器 | 数据源，你改的就是它 |
| `absLeft` `absTop` | 相对屏幕 | **工具渲染用它** |
| `showLeft` `showTop` | 相对父容器（仅 Img） | **生成的 C 代码用它** |

### 改尺寸时注意 Img 的另一对字段

Img 的 `img_width/img_height` 是**原图的像素尺寸**，不是显示尺寸。显示尺寸由 `width/height` 和 `zoom`（256 = 100%）决定。要缩放图片改 `width/height` 或 `zoom`，别动 `img_width/img_height`。

## 第四步：改文字和样式

### 文字

`Label` 和 `Button` 的显示文字在 `text` 字段。如果 `text_use_i18n` 为 true，说明走多语言表，改 `text` 无效，要去 `jlui/design/default/` 的 i18n 配置里改。

如果控件有 `bindMsg`（模型绑定），例如：

```json
"bindMsg": { "text": { "model_value": "songs.tol_time", "mode": "oneWayModelToView" } }
```

说明这个文字在运行时由代码动态写入，json 里的 `text` 只是设计期占位。改它对实际显示没用——真正要改的是代码里往那个模型字段写值的地方。看到 `bindMsg` 就要提醒用户这一点。

### 样式

`style` 是数组，每项对应一个 **part + state** 组合：

```json
"style": [
  { "part": "LV_PART_MAIN", "state": "LV_STATE_DEFAULT",  "text_color": "#ffffff", "font": 32, ... },
  { "part": "LV_PART_MAIN", "state": "LV_STATE_FOCUSED",  "disable": true, ... },
  { "part": "LV_PART_MAIN", "state": "LV_STATE_DISABLED", "disable": true, ... }
]
```

- `disable: true` 表示这条样式不生效——工具只是把默认值存在那儿。**改样式前先确认目标那条没有 `disable: true`**，否则改了不会有任何效果。
- 常规改动只动 `LV_PART_MAIN` + `LV_STATE_DEFAULT` 那条（通常是 `style[0]`）就够。
- 颜色是 `#rrggbb` 字符串，透明度 `*_opa` 是 0–255，`font` 是字号数字，`font_family` 是字体名（要和工程已导入的字体对得上，随便写会导致字体加载失败后静默退回默认字体，现象是中文变方块）。

**json 的样式面板只是 LVGL 样式的一个子集，而且每个 part 的子集还不一样。** 想改某个属性之前，先看目标那条 style 里有没有这个字段——没有就是工具没暴露，写进去也不会生成代码。

实例：Slider 的三个 part 里，只有 `LV_PART_MAIN` 有 `outline_*` 和 `shadow_*`，`LV_PART_INDICATOR` 和 `LV_PART_KNOB` 就只有 `bg_*` 和 `radius`，**没有 `border_*`**。所以"给滑块圆钮加一圈描边"这件事 json 做不到，只能在页面加载回调里补一句：

```c
lv_obj_set_style_border_color(slider, lv_color_hex(0x1180FD), LV_PART_KNOB | LV_STATE_DEFAULT);
lv_obj_set_style_border_width(slider, 2, LV_PART_KNOB | LV_STATE_DEFAULT);
lv_obj_set_style_border_opa(slider, LV_OPA_COVER, LV_PART_KNOB | LV_STATE_DEFAULT);
```

这是"在代码里设样式"的**合法例外**——前提是确认过 json 真的没有这个字段，而不是图省事。属于这一类的还有 `clip_corner`（圆角裁剪开关）。凡是这么补的，都在代码里写清楚"json 没暴露这个字段"，免得后人以为是重复配置又去删。

查字段最快的办法是直接读 json：

```python
st = [s for s in node['style'] if s['part'] == 'LV_PART_KNOB'
      and s['state'] == 'LV_STATE_DEFAULT'][0]
print(sorted(st.keys()))
```

## 第五步：怎么动手改

### 先备份

**`ui_prj/` 一般被工程的 `.gitignore` 排除掉了**，设计文件不在 git 里，改坏了没法 `git checkout` 回退。动手前先确认：

```bash
git check-ignore -v ui_prj/           # 有输出就说明被忽略
```

被忽略就先手工复制一份到临时目录再改。这步别省——改完发现方向不对想退回来，没备份就只能靠 GUI Builder 重画。

### 两种改法，按改动量选

这些 json 是 **UTF-8 无 BOM、LF 换行、2 空格缩进**的展开格式，每个字段独占一行。

**小改动（一两个字段）用 Edit 精确替换。** 定位技巧：同一个文件里 `"left":` 会出现几十次，直接替换会改错控件。先 Grep 找到目标控件的 `"name": "imgbtn_3"` 所在行号，再读那一段确认字段位置，然后把控件名连同要改的字段一起作为上下文替换。

**批量改动（多个控件、移动层级）用 Python 读写整个文件。** 关键是 dump 参数要和工具的输出格式对齐：

```python
json.dumps(j, ensure_ascii=False, indent=2)     # 末尾不要再加换行符
```

实测这样 dump 出来的和工具自己写的文件**逐字节一致**（4415 行全部对得上），所以 diff 里只出现你真正改的那几行，用户照样能复核。写文件时指定 `newline` 为换行符，避免在 Windows 上被转成 CRLF。

别用默认参数 dump——`ensure_ascii=True` 会把中文转成 `\uXXXX` 形式，不给 `indent` 会挤成一行，那才会产生满屏无意义 diff。

改完用 `json.load()` 验证一遍文件没坏，很便宜，能挡住手滑。

## 动态中文文本：必须用矢量字体

界面上运行时才知道内容的文字（歌名、歌词、来电号码、文件名），**点阵字体一定显示不出中文**。

原因是点阵字库按"打包时扫描到的字符"生成。设计稿里写死的「播放」「设置」会被扫进去，但运行时才拿到的中文不在其中。现象很好认——串口刷这种警告，而英文数字正常：

```
lv_draw_sw_letter: glyph dsc. not found for U+5361
```

想靠扩大点阵字库覆盖全部汉字也不现实：GB2312 一级 3755 字，28px、4bpp 单字号就约 1.4 MB，**而且每个字号都要独立一份**。三个字号就 4 MB 以上，远超一般工程的 UI 资源区。（工具里给点阵字库批量加字的入口是样式面板的「字库」，见下文「字库（font_library_id）」）

所以这类文本只能走**矢量字体 + freetype**。一份裁剪过的 ttf 通吃所有字号：

| 字体 | 字符集 | 体积 |
|---|---|---|
| 楷体 | GB2312 一级 3755 字 | 1817 KB |
| 楷体 | 常用 2000 字 | 900 KB |
| **宋体** | **GB2312 一级 3755 字** | **983 KB** |
| 宋体 | 常用 2000 字 | 502 KB |

宋体笔画简单、压缩率高，同样字数比楷体小得多。用 `fontTools` 的 `subset` 裁剪，字符集用 GB2312 一级（从 `bytes([hi,lo]).decode('gb2312')` 枚举 0xB0A1–0xD7FE 得到）加 ASCII 和常用标点。

官方 FAQ 提醒矢量字体有渲染开销（每个字形首次要从存储读取并 `FT_Render_Glyph`），对策是调大 `LV_FREETYPE_CACHE_SIZE` 并开启 `LV_FREETYPE_SBIT_CACHE`。这两项很多工程默认已经配好，先确认再动。

### 把字体打进 Flash（摆脱 SD 卡依赖）

不少方案默认把 ttf 放 SD 卡（资源 id 落在 SD 段，路径形如 `storage/sdX/C/xxx.ttf`），于是不插卡就没有中文——蓝牙播放这种场景本不该要求插卡。

搬到 Flash 的做法：文件名用 **flash 段的 id**（比如 `0x30000000` 段就命名 `30000000.ttf`），放进资源打包目录，运行时路径就是 `<flash_dir>/30000000.ttf`。代码里把字体路径宏改成这个。

**关键：别放进工具会重建的目录**（见下一节）。

给控件挂矢量字体：

```c
lv_ft_info_t info = {0};
info.name = 字体路径;
info.weight = 字号;              // 必须与 json 里该控件的 font 一致
info.style = FT_FONT_STYLE_NORMAL;
if (lv_ft_font_init(&info)) {
    lv_obj_set_style_text_font(label, info.font, LV_PART_MAIN | LV_STATE_DEFAULT);
}
```

字体对象要**按字号缓存**，别每次渲染都 init。挂载时机放在页面加载回调里，那时控件已由 setup_scr 创建完毕。

改字号时记住**两处要同步**：json 里的 `font` 决定设计器预览，代码里 `lv_ft_info_t.weight` 决定运行时实际渲染。只改 json 不改代码，运行时还是旧字号。

## 哪些目录是工具产物，放东西会丢

除了前面说的 `custom/`，还有别的目录会被工具重建。判断方法是打包后看目录的 mtime——被刷新了就是产物。

| 目录 | 性质 | 能否手工放文件 |
|---|---|---|
| `<工程>/sdk/ui_res/` | **每次打包清空重建** | 不能，必放必丢 |
| `<风格>/generated/` | 每次打包重新生成 | 不能 |
| `<风格>/custom/` | 打包时从 ui_prj 侧覆盖过来 | 改 ui_prj 那份 |
| `<工程>/import/` | 用户导入的素材 | 可以 |
| build_script（如 `copy_ui_res.bat`） | 非产物 | 可以，且是挂钩点 |

手工资源（裁剪字体、额外素材）的稳妥放法：**新建一个工具不认识的目录**（例如 `<工程>/sdk/extra_res/`），再在 build_script 里加一段把它拷进打包暂存区：

```bat
if exist ".\extra_res" (
	for /r ".\extra_res" %%F in (*) do (
		copy /y "%%F" "%targetDir%"
	)
)
```

注意 build_script 通常会先 `del /q` 清空暂存区再拷贝，所以这段要放在清空之后。

### 改 .bat 必须保持 CRLF

这类脚本是 Windows 批处理，**必须 CRLF 换行**。写成 LF 的话 cmd 解析多行结构（`for ... do (` 的括号块）会错乱，`::` 注释也会被拆开当命令执行，报出一堆莫名其妙的 `'xxx' is not recognized`。

用 Python 改脚本时别带 `newline='
'`——那个参数用在 `.c`/`.json` 上是对的，用在 `.bat` 上就会坏。

另外 `::` 注释里**尽量别写中文**。cmd 在 UTF-8 代码页下按字节解析，某些中文的字节组合会被当成命令分隔符，把注释拆成命令。原有的中文注释能跑是碰巧字节没踩雷，新增的用 ASCII 最稳。

## 还原设计稿：底图 + 控件叠加

拿到一张设计稿要求"做得一模一样"时，不要试图用控件的 `style` 去凑。LVGL 的 style 只有矩形和圆角矩形，**切角多边形、霓虹辉光、不规则渐变、装饰纹理这些都做不出来**，硬凑的结果是"布局像、质感完全不像"。

正确的分工是：**静态视觉走图片资源，动态内容走控件**。

### 步骤

1. **把设计稿缩放到屏幕分辨率。** 设计稿往往不是正好的目标比例，直接 resize 会变形。按宽度缩放到目标宽度，高度不足的部分用边缘像素条带延展补齐，这样面板本身不会被拉伸。源图偏小时先降噪（`MedianFilter`）再用 `LANCZOS` 放大，最后 `UnsharpMask` 锐化找回边缘，能明显压掉锯齿和 JPG 噪点。

2. **底图只留纯装饰，其余全部抹掉。** 底图里该留的只有：背景照片/渐变、顶栏半透明条、分隔线、面板卡片这类永远不变且不可点的东西。**图标、按钮、文字、滑条、进度条、频谱、封面——一律抹掉，改由控件去画。**

   漏抹是最常见也最难自查的错误：你在原位贴上新控件，和底图里的旧图形严丝合缝地叠在一起，远看只觉得"有点糊、有点重影"，放大到 4 倍才看得出是两层。**做完底图先自检一遍：把底图单独看一眼，凡是还能认出是"一个图标/一个按钮/一行字"的，就是没抹干净。**

   擦除别用纯色填充，按区域的底色结构选手法：
   - 照片区（天空、湖面这类纵向渐变）→ 取区域**上下两条干净行**做垂直插值
   - 顶栏、底部卡片（纵向有明确边缘，横向近乎均匀）→ 取**左右两侧干净列**做水平插值，垂直插值会把卡片上下沿抹平

   抹除有先后顺序：**相邻的两块，被挡住的那块要先抹**。比如封面正下方就是频谱，封面还在时，频谱的上取样带会落进封面里，把深色一路拖到下面去。

3. **静态图标和文字也要做成控件，但别做成 Label 图片。**
   - 图标 → `Img` 控件（不可点）或 `ImgButton`（可点），贴图见下一节
   - 文字 → **一律 `Label` 控件，绝不把文字渲染成 PNG 贴上去**。文字进了图片就没法改文案、没法多语言、没法跟状态变色，字号一改还得重做图

4. **按下态/选中态靠改贴图，不靠 style。** `src_pressed` 用同一张图整体压暗一档（浅色 UI 上压暗比提亮更像"按下去了"），`src_checked_released` 把图标笔画换成品牌色。注意这些变体只能改**有像素的地方**，透明区必须保持透明。

5. **底图设为 Screen 的背景图**：`style` 里的 `bg_img_src` 指向图片，`bg_img_opa` 设 255，同时把 `bg_opa` 设 255、`bg_grad_dir` 设 `LV_GRAD_DIR_NONE`，避免底色透上来。

### 图标贴图：透明底 + 矢量重绘，禁止从设计稿抠图

这一节是返工重灾区，三条硬规矩：

**1. 图标必须是 RGBA 透明底，不许带任何背景底色。**

看一眼工程里已有的图标就知道规矩：`ic_*` 那种是 `mode=RGBA`、四周全透明、只有图形本身；而 `btn_*`、`cut_*` 那种 `mode=RGB` 的是从设计稿直接抠的，四角带着卡片底色。带底色的贴图只在"贴回抠图时的原坐标、且底图分毫未动"时才看不出破绽——控件挪一像素、底图重做一版、或者同一个图标想复用到别的页面，立刻露出一圈方块。

自查：`Image.open(p).convert('RGBA')` 后看 alpha 通道，`(alpha==0).mean()` 应该有相当比例（图形外的区域），如果是 0 就说明这张图是实底的。

**2. 不许从设计稿抠图，要按形状重新矢量绘制。**

设计稿是位图，图标在目标分辨率下往往只有十几二十像素。从 1600px 的稿子上抠一块再缩到 19px，抗锯齿会把细笔画摊成一片半透明灰——**看起来就是又糊又带毛边**。正确做法是在 **8 倍超采样**的画布上用 `ImageDraw` 把形状画出来，再 `LANCZOS` 降到目标尺寸，边缘是干净的抗锯齿。

画之前先把设计稿对应区域放大到 8–10 倍看清楚形状再动手。凭印象画出来的图标必然不对——齿轮是"矩形齿 + 大圆孔的环"还是"尖齿实心圆"、循环是"两段弧 + 实心三角箭头"还是"一个圈加个钩"，差一点就完全不是一个东西。

小尺寸下**笔画粗细不能按图标尺寸等比给**。19px 的图标上 0.30 倍半宽的线宽会让交叉处连成一团墨；设计稿实际约是 0.20–0.22 倍。画完务必按 1:1 贴到底图上看，再放大到 6 倍和设计稿并排比。

**3. 复用工程已有图标要先确认风格一致。**

工程里常有一整套现成图标（`suiji` / `suijibofang` / `shezhi` / `eq` / `shengyin` 这类，64×64 白色 RGBA）。能直接重上色缩放当然省事，但**先看风格对不对**：那一套通常是**描边**风格，而清新/扁平类设计稿多是**实心**风格，混用会明显不搭。

要复用就整组复用，别只换一两个。缩放时 `LANCZOS` 之后对 alpha 走一条 S 形曲线把中间调往两头推，边缘才收得住，否则 64→20 的描边会只剩一层灰。

**一句话流程**：放大看清形状 → 8 倍画布矢量绘制 → 降采样 → 存 RGBA → **立刻拷进资源目录**，贴回底图 1:1 看一遍再做下一个。攒一批再给人看，错的地方会成批返工。

### use_fs 决定图片是文件还是 C 数组

每个用到图片的节点（含 Screen）都有 `use_fs`，它决定图片怎么进固件：

| `use_fs` | 图片去向 | 生成代码里的写法 |
|---|---|---|
| `true` | 打包进资源区，运行时按路径读 | 路径字符串 |
| `false` | 转成 C 数组编进固件 | `&_<目录>_<图名>_alpha<W>x<H>` 取地址 |

**大图必须 `use_fs: true`。** 一张 800×480 内嵌成数组是 768 KB，工具往往不会真的生成那个数组文件，于是链接阶段报：

```
undefined reference to `_icon_xxx_ui_bg_alpha800x480'
```

看到 `undefined reference to` 加上 `_alpha<宽>x<高>` 这种符号名，就是漏改 `use_fs` 了。

Screen 节点的 `use_fs` 默认常常是 `false`，给它设背景图（`style` 里的 `bg_img_src`）时特别容易踩。改的时候顺手把图片编码参数（`image_color_format`、`image_rle_block`、`quality`）也照抄同工程里一个已经正常走文件系统的 Img 控件，别自己猜值。

改完 `use_fs` 必须让工具**重新生成代码**——生成的是取地址还是路径字符串，在代码里是写死的，光改 json 不重新生成没用。

### Img 控件：尺寸必须等于图片实际尺寸

这条会直接毁掉效果。**LVGL 的 Img 控件在 `width/height` 小于图片尺寸时是裁切，不是缩放**，`zoom` 字段在这条路径上不起作用。拿 64×64 的图去填 28×28 的控件，显示出来的是原图左上角的 28×28，看起来就是"图标碎了一角"。

所以要用小图标时，**先把 PNG 缩放成目标尺寸存成新文件**，再让 `width`/`height`/`img_width`/`img_height` 四个值都等于它，`zoom` 保持 256。ImgButton 同理。

### 字体和字号

字号不限于工程现有的那几档，工具会按需生成对应字号的字体资源。改 `style` 里的 `font`（字号）和 `font_family`（字体名，要和 `import/font/` 下的文件对得上）即可。

新用到的「字体+字号」组合最好同步加进 `jlui/design/default/projectFont.json`，免得工具漏生成。代价是每多一档字号就多一份字体资源，别无节制地用。

设计稿里的英文和数字用无衬线字体（如 `montserratMedium`）比用中文楷体更接近原稿观感。


### 「字库」（`font_library_id`）：给点阵字体额外预置汉字

控件**样式**面板里、紧挨着「字体大小」的那个「字库」下拉，对应 json 的 `font_library_id`（就在 `font_family` 和 `letter_space` 之间）。选项是 `无` / `3500常用字(system_pinyin_3500)` / `7000常用字(system_pinyin_7000)` / `所有汉字(system_pinyin_all)`。

它的作用是**扩大点阵字库的字符集**。点阵字体默认按需裁剪，只烘设计稿里出现过的字——看 `projectFont.json` 就很直白：

```json
{"font_family": "simsun", "font_size": 12,
 "content": ["随","机","循","环"],      ← 只有设计稿里写死的这 4 个字
 "range": "0x0020-0x007E"}              ← 外加 ASCII
```

所以运行时才知道的中文必然缺字。官方 FAQ 里「通过自定义代码修改控件文本后出现部分字体乱码显示不全 / 解决方法：手动添加字库」，说的就是这一项。

**代价很大，先算够不够放**。字库按「字体 + 字号」各生成一份，每多一档字号就多一整份。24px、4bpp 的量级估算：

| 选项 | 单个字号约 |
|---|---|
| 3500常用字 | ~1 MB |
| 7000常用字 | ~2 MB |
| 所有汉字（CJK 基本区两万多字） | ~6 MB |

而 `sdk/cpu/<芯片>/tools/isd_config.ini` 里 `UIPACKRES_LEN` 常见是 `0x200000`（2 MB）。3500 一个字号就吃掉一半，7000 顶满，所有汉字必爆。

**切到矢量字体之后这一项就没意义了**：矢量是把 ttf 打进资源区、运行时用 freetype 现渲染，一份 ttf 通吃所有字号，不存在"打包时没扫到"。判断工程当前走哪条路——看打包产物：

```
sdk/ui_res/flash/font/  下是 .ttf   → 矢量字体，「字库」保持「无」
                        下是 .bin   → 点阵，缺字才需要考虑这一项
```

⚠ **别和虚拟键盘的「字库」搞混**，两者 UI 上同名但完全无关：

| | 样式面板的「字库」 | 虚拟键盘的「字库」 |
|---|---|---|
| json 字段 | `font_library_id`（控件 **style** 里） | `chinese_library_id`（Keyboard **控件属性**，和 `kb_mode`/`py_mode` 并排） |
| 作用 | 往字体资源里多烘一批字形 | 拼音输入法的**候选字词库** |
| 在哪配 | 样式面板 | 资源 → 字库配置 |

## 新增控件和调整层级

### 新增控件：复制，不要手写

一个 Label 节点在 json 里是 **298 行、72 个顶层字段、4 组 style**（每组 35–39 个字段）。手写必漏，漏了字段轻则属性异常，重则工具打不开工程。

可靠的做法是**从同一个文件里找一个同类型控件，深拷贝它的整个节点**，然后只改这几项：

| 必改 | 说明 |
|---|---|
| `id` | `#` + 16 位随机字母数字，**工程内不能重复** |
| `name` | 页面内唯一，字母开头、至少 3 字符、只含字母数字下划线 |
| `left` `top` `width` `height` | 目标位置尺寸（Img 还要同步 `showLeft/showTop`） |
| 类型特有字段 | Label 的 `text`、ImgButton 的 `src_*`、Img 的 `img_path` 等 |

还要清掉从源控件带过来、不该继承的东西：`bindMsg`（模型绑定）、`event`（事件回调）、`animation`，否则新控件会莫名其妙跟着别的数据变。

把新节点 append 到目标父容器的 `children` 数组即可。

### 调整层级：移动节点 + 换算坐标

把控件从一个容器挪到另一个，就是把它的节点从源 `children` 里删掉、append 到目标 `children`。

坐标要换算：`left/top` 是**相对父容器**的。控件从父 A 移到父 B 后，要保持屏幕位置不变，新坐标 = 旧坐标 + A的绝对坐标 − B的绝对坐标。如果 A 在 (0,0) 而 B 是 Screen，坐标不用变。

`children` 数组顺序就是绘制顺序，**append 到末尾 = 画在最上层**。

## 第六步：改完之后（不能省）

json 改好只是第一步，改动还没进固件。整个流程分**工具端**和**代码端**两截，都是用户做的：

| 端 | 操作 | 做了什么 |
|---|---|---|
| 工具端（GUI Builder） | 编辑界面 → 点**「打包」** | 重新生成 `setup_scr_*.c`、导出资源、跑 build_script，并把 `ui_prj/<工程>/custom/` 同步到 sdk 侧 |
| 代码端（工程） | **编译** → **下载** | 编出固件并烧进板子 |

工具端只需要「打包」这一个动作，不要让用户去点别的。编译和下载都在代码端做，GUI Builder 不负责这两件事。

顺序不能反：先编译后打包的话，编译吃的是打包前的旧代码。

这两端的操作都不要替用户做，也不要手工 `cp` 去"帮忙同步"（见铁律 1）——AI 该做的就是改好 `ui_prj` 下的 json 和 `custom/` 代码，然后停下来告诉用户去打包。

### 校验是否真的生效

让用户操作完后，用时间戳自查，这三个时间必须**递增**：

```
ui_prj/<工程名>/jlui/design/default/ui/<页面名>.json                    设计数据
sdk/apps/<应用>/lvgl_v8_ui_app/<风格目录>/generated/gui_scr/setup_scr_<页面名>.c   生成代码
sdk/apps/<应用>/board/<板型>/objs/.../setup_scr_<页面名>.c.o            目标文件
```

（`.c` 和 `.o` 的具体路径按实际工程结构找，`objs/` 目录镜像了源码树。）

哪一环时间戳是倒着的，问题就卡在那一环：

- `.c` 比 `.json` 旧 → 代码没重新生成，让用户确认打包确实跑完了（没报错、没弹窗卡住）
- `.o` 比 `.c` 旧 → 编译没吃到新代码。工具重新生成 `setup_scr_*.c` 的方式有时会让 make 的增量编译认不出文件变了，`.o` 停在旧时间戳，链接出来还是老布局（实测出现过源码 11:44:07、`.o` 还停在 11:22:45）。这时清掉 `objs/` 下对应的 `.o` 或整体全量重编即可
- 都正常但界面还是旧的 → 检查是不是改了 Img 却漏了 `showLeft/showTop`，或者目标样式带 `disable: true`，或者该控件有 `bindMsg` 被代码动态覆盖

顺带一提，工具自己保存时偶尔会把坐标改动 1 像素（Img 的 `offsetX/offsetY` 机制导致），看到设计文件和你写的值差 1 属正常，不是你改错了。

## 代码侧：让控件动起来

json 只决定控件长什么样、在哪。运行时把数据填进去、响应点击、画自定义内容，都是 C 代码的事。

动手改布局之前，有一件事值得先做：**拿控件名去 `custom/` 目录 grep 一遍**。原因是有些控件会被代码在运行时替换掉——典型写法是 `lv_img_create(容器)` 新建一个再 `lv_obj_del(原控件)`，然后用 `lv_obj_align()` 重新定位。这种情况下你在 json 里调的位置尺寸**运行时会被代码覆盖**，而且设计器预览里完全看不出来，只有烧板子才发现。看到这种组合，就说明那块归代码管。

同理，频谱、波形这类自绘内容在 json 里只有一个空容器，**它们的位置尺寸由那个容器决定**，所以挪动它们是改 json；但形态（圆形还是柱状、什么颜色）在绘制回调里，那是改代码。

详细内容见 `references/code-integration.md`：控件指针怎么取、为什么绝不能在非 LVGL 线程调 `lv_*`、UI 消息总线的机制和三个坑（`strstr` 子串匹配、`va_arg` 类型、总入口守卫）、数据不更新时的排查顺序、按钮事件的弱符号覆盖、自定义绘制、以及运行时替换控件的陷阱。

排查界面问题时那份文档末尾有张对照表，按现象查起因，比从头读快。

## 参考资料（按需读，别一次全看）

| 文件 | 什么时候读 |
|---|---|
| `references/widget-fields.md` | **改控件属性**。10 类控件的特有字段、取值、各自的官方文档链接 |
| `references/code-integration.md` | **写代码 / 查现象**。控件指针、线程约束、UI 消息总线、事件回调、自绘内容、数据不更新的排查顺序 |
| `JLGuiBuilder_doc/` | **本 skill 没写到的任何东西**。官方文档离线全量副本，见下节检索方法 |

改一个界面的典型顺序：本文「工作循环」→ 按「铁律」避坑 → `widget-fields.md` 查字段 → 改 json → `code-integration.md` 接数据。

> ⚠ **要动哪类控件，就把 `widget-fields.md` 里那一节读完。** 比如改 Img 的位置，只看"坐标"那段会漏掉 `showLeft/showTop`；改 Slider 的配色，不看那节会漏掉颜色分散在 `LV_PART_INDICATOR` / `LV_PART_KNOB` 两个 part 上。

> ⚠ **界面上看得见的东西不一定是 json 控件。** 动手前先拿控件名去 `custom/` 目录 grep 一遍，确认它不是代码运行时创建/替换的，也不是绘制回调画出来的。详见 `code-integration.md`。

## 更多字段

各控件类型的完整特有字段（Slider 的量程、ComboBox 的选项表、Timer 的周期、ImgButton 的四态图片等）见 `references/widget-fields.md`，那里每个控件都附了对应的官方文档链接。需要改坐标尺寸以外的属性时再去查。

## 官方文档（本地副本，优先查这里）

`JLGuiBuilder_doc/` 是官方文档站的完整离线副本（500+ 文件）。**本 skill 没覆盖到的问题，先在这里找**，比在线翻快，也不受网络影响。

```
JLGuiBuilder_doc/
├── controls/     各控件的属性面板说明（screen/button/imagebutton/label/image/
│                 slider/switch/combobox/bar/arc/meter/chart/album/各种菜单/view/flex）
├── advanced/     时间轴动画 model-bind(模型绑定) res(资源管理) i18n(国际化)
│                 theme group screen-manage dyn(动态页面) hardware-simulation monkey
├── extras/       generated-interfaces.html ← 工具生成的 API 接口清单
├── faqs/         faq_compiler / faq_lvgl / faq_simulator / faq_uitool / faq_other
├── operation/    quick(快速使用) helloworld board(板子配置) video_use hardware
├── plugin/       插件：字体合并 图片编辑 截图 项目合并 视频转图片 图片转换
└── case/         案例：倒计时 菜单滑动 自定义Symbol 暗色键盘主题
```

### 怎么检索

文件是 HTML，直接 Read 会被标签淹没。有效做法是**先 grep 定位文件，再剥标签取正文**：

```bash
# 1) 按关键词找是哪篇
grep -rli "矢量字体\|freetype" JLGuiBuilder_doc --include=*.html | grep -v assets
```

```python
# 2) 剥掉标签读正文（按关键词取上下文，别整篇打印）
import re, io
s = io.open(path, encoding='utf-8', errors='replace').read()
s = re.sub(r'<script.*?</script>|<style.*?</style>', '', s, flags=re.S)
s = re.sub(r'<[^>]+>', ' ', s)
s = re.sub(r'\s+', ' ', s)
for m in re.finditer(关键词, s):
    print(s[max(0, m.start()-150) : m.start()+250])
```

每个页面都带完整的左侧导航菜单文本，grep 命中导航里的词是常见误判——**看命中处的上下文是不是正文**，不是就换个更具体的关键词。

### 这份文档讲什么、不讲什么

- **讲**：设计器里每个属性的语义和取值、工具的操作流程、插件用法、常见问题的官方解释。
- **不讲**：json 的字段名和文件格式（那是工具内部存储，官方没公开），本 skill 里的 json 字段说明都是从实际工程逆向出来的。

所以"这个属性是干什么的、有哪些合法取值"查官方；"它对应 json 里哪个字段"看本 skill。两边对不上时，以工程里现有的同类控件为准——那是工具自己写出来的，一定合法。

### 在线地址

https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/ —— 本地副本可能滞后于线上，涉及新版本特性时再去线上核对。
