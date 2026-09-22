# 控件字段参考

各控件类型在 `ui/<页面名>.json` 里的特有字段。通用字段（`id` `type` `name` `left` `top` `width` `height` `visible` `children` `style`）见 SKILL.md。

字段值都是实测样本，取值格式以工程里已有的同类控件为准——拿不准时找一个现成的同类型控件照着写，比猜安全。

每节开头附了该控件的官方文档地址。官方文档讲的是设计器属性面板里的属性语义和取值，**不涉及 json 字段名**——两边配合看：官方回答"这个属性是干什么的"，本文件回答"它在 json 里叫什么"。

官方文档总入口：https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/controls/

## 目录

- [Img 图片](#img-图片)
- [ImgButton 图片按钮](#imgbutton-图片按钮)
- [Label 标签](#label-标签)
- [Button 按钮](#button-按钮)
- [Slider 滑动条](#slider-滑动条)
- [Switch 开关](#switch-开关)
- [ComboBox 下拉框](#combobox-下拉框)
- [Timer 定时器](#timer-定时器)
- [View 容器](#view-容器)
- [Screen 页面](#screen-页面)
- [资源存储位置字段](#资源存储位置字段)

## Img 图片

> 官方文档：https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/controls/image.html

```json
"img_path":   "..\import\image\icon_02itzp4fqyho\image_4225532_4.png",
"showLeft":   699,       "showTop":   350,
"offsetX":    -1,        "offsetY":   -1,
"img_width":  48,        "img_height": 48,
"zoom":       256,       "mask_zoom": 256,
"img_align":  "LV_IMAGE_ALIGN_TOP_LEFT"
```

| 字段 | 说明 |
|---|---|
| `img_path` | 图片相对路径，相对 `jlui/design/default/`，用 `\` 转义的反斜杠。空串表示占位图 |
| `showLeft` `showTop` | **实际渲染坐标，生成代码取这个**。改位置时必须和 `left/top` 一起改 |
| `offsetX` `offsetY` | `showLeft = left - offsetX`。一般是 -1，改坐标时保持差值不变 |
| `img_width` `img_height` | 原图像素尺寸，不是显示尺寸。不要手改 |
| `zoom` | 缩放，256 = 100%，512 = 200% |
| `img_align` | 图片在控件框内的对齐方式 |

换图片改 `img_path`，图片要先放进工程的 `import/image/` 下，否则编译时找不到资源。

## ImgButton 图片按钮

> 官方文档：https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/controls/imagebutton.html

```json
"toggle": true,
"src_released":         "..\import\image\...\image_4225207_0.png",
"src_pressed":          "..\import\image\...\image_4225207_0.png",
"src_checked_released": "..\import\image\...\image_4225207_1.png",
"src_checked_pressed":  "..\import\image\...\image_4225207_1.png",
"activeImgBtnIdx": 0
```

四个状态各一张图。`toggle: true` 时按钮有选中/未选中两态，`checked_*` 那两张才会用到；`toggle: false` 时只用 `src_released` 和 `src_pressed`。

播放/暂停这类切换按钮通常 `toggle: true`，用 `checked` 态区分。

## Label 标签

> 官方文档：https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/controls/label.html

```json
"text": "01:27",
"long_mode": "LV_LABEL_LONG_WRAP",
"text_use_i18n": false
```

`long_mode` 决定文字超长时的行为：`LV_LABEL_LONG_WRAP`（换行）、`LV_LABEL_LONG_DOT`（省略号）、`LV_LABEL_LONG_SCROLL`（来回滚动）、`LV_LABEL_LONG_SCROLL_CIRCULAR`（循环滚动）、`LV_LABEL_LONG_CLIP`（裁切）。

歌曲名这类不定长文字常用滚动模式。

`text_use_i18n: true` 时 `text` 只是设计期占位，实际文字来自多语言表，改 `text` 无效。

文字样式的字段在 `style` 里：`font`（字号）、`font_family`（字体名）、`text_color`、`text_align`、`letter_space`、`line_space`，以及 `font_library_id`——它是样式面板里的「字库」，给点阵字体额外预置一批汉字，代价和适用场景见 `fonts.md` 的「字库（font_library_id）」。

## Button 按钮

> 官方文档：https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/controls/button.html

```json
"text": "保存",
"text_width": 60,  "text_height": 20,
"long_mode": "LV_LABEL_LONG_WRAP"
```

Button 内部自带一个 label，`text_width/text_height` 是那个 label 的尺寸，和按钮本身的 `width/height` 是两回事。

## Slider 滑动条

> 官方文档：https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/controls/slider.html

```json
"range_s": 0,  "range_e": 100,
"starting_value": 50,
"slider_mode": "LV_SLIDER_MODE_NORMAL"
```

`range_s`/`range_e` 是量程上下限，`starting_value` 是初值。`slider_mode` 可选 `LV_SLIDER_MODE_NORMAL`、`LV_SLIDER_MODE_SYMMETRICAL`（从中间往两边，适合 EQ 增益）、`LV_SLIDER_MODE_RANGE`（双游标区间）。

滑动条的滑块和轨道是不同的 part，改颜色要找 `style` 里 `part` 为 `LV_PART_INDICATOR`（已填充部分）或 `LV_PART_KNOB`（滑块）的那条，不是 `LV_PART_MAIN`。

## Switch 开关

> 官方文档：https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/controls/switch.html

```json
"checked": true
```

只有开关初始状态一个特有字段。开关的样式同样分 `LV_PART_MAIN`（背景槽）和 `LV_PART_KNOB`（圆钮），选中态在 `LV_STATE_CHECKED`。

## ComboBox 下拉框

> 官方文档：https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/controls/combobox.html

```json
"list": "普通模式\n摇滚模式\n流行模式\n古典模式\n爵士模式\n乡村模式\n自定义模式",
"selected_index": 6,
"direction": "LV_DIR_BOTTOM"
```

选项表是**单个字符串**，用 `\n` 分隔各项——在 json 里就是字面的 `\n` 转义序列，不是真换行。增删选项就是改这个字符串。

`selected_index` 从 0 开始，改选项表后记得检查它没有越界。`direction` 是展开方向。

## Timer 定时器

> 官方控件列表里没有单独的定时器页面，相关说明散见于 https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/controls/ 总览和各案例（如 https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/case/countdown.html 倒计时案例）

```json
"period": 300,
"repeat_count": -1,
"timer_cb_code": "static uint32_t angle_idx = 0;\r\nlv_img_set_angle(...);"
```

非可视控件，在设计器画布上有位置但不渲染，生成代码里也没有 `lv_obj_set_pos`，所以改它的 `left/top` 没有任何实际作用。

`period` 是毫秒周期，`repeat_count: -1` 表示无限重复。

`timer_cb_code` 里是**内嵌的 C 代码**，会被原样贴进生成的 `setup_scr_*.c`。改它等于改代码，要按 C 语法来，写错会导致整个工程编译失败，而报错位置在生成文件里，不容易一眼看出来源。改之前先确认用户是想改定时逻辑而不是改布局。

## View 容器

> 官方文档：https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/controls/view.html

没有特有字段。它的价值是给子控件提供独立坐标系——子控件的 `left/top` 相对 View 的左上角。

整块界面要平移时，改 View 的 `left/top` 比逐个改子控件省事得多，也不容易出错。

## Screen 页面

> 官方文档：https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/controls/screen.html

顶层节点，`width/height` 就是屏幕分辨率（例如 800x480），一般不动。

```json
"is_load_func": true,   "is_unload_func": true,
"setup_scr_inc": "",    "setup_scr_code": "",
"version": 151
```

`setup_scr_inc` / `setup_scr_code` 也是内嵌 C 代码（头文件包含 / 页面初始化追加代码），注意事项同 Timer 的 `timer_cb_code`。

`is_load_func` / `is_unload_func` 控制是否生成页面加载/卸载回调。

`version` 是工具的工程格式版本，不要改。

## 资源存储位置字段

> 官方文档：https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/advanced/res.html

很多控件带这几个字段，控制该控件用到的图片资源打包到哪：

```json
"storage_medium": "default",
"use_fs": true,
"fs_type": "default",
"image_format": "default",
"image_color_format": 3,
"quality": 80,
"alpha_quality": 90
```

`storage_medium` 决定资源进 Flash 还是 SD 卡，它会影响生成的资源 id 落在哪个段（Flash 段还是 SD 段），进而决定运行时去 `mnt/sdfile/...` 还是 `storage/sdX/...` 找文件。

**改成存 SD 卡要当心**：工具会在 `<工程>/sdk/ui_res/sd/` 生成一份资源，需要手工拷到 SD 卡根目录，而且生成代码里的 SD 根路径可能写死成 `storage/sd0/C/`——如果板子实际用的是 SD1，资源就会读不到。改这个字段前先确认板级用的哪个卡槽。

`quality` / `alpha_quality` 是图片压缩质量，影响固件里资源占用的空间。
