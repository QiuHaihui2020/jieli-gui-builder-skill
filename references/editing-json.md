# 第二步～第五步：读懂结构 · 改坐标和尺寸 · 改文字和样式 · 怎么动手改

> 本篇内容原在 `SKILL.md`，整段移出，未作改写。整体工作循环和铁律仍在 `SKILL.md`。

---

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

