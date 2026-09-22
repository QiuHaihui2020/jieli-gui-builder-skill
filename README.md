# jieli-gui-layout

杰理（JieLi）JLGuiBuilder 界面开发参考文档集，以 Skill 形式组织，供 AI 编程助手检索。内容覆盖工程数据格式、界面布局修改方法、UI 代码对接方式，以及资源打包与编译流程。

## 适用范围

适用于采用 LVGL v8 + JLGuiBuilder 方案的杰理彩屏工程，判定条件如下，三项需同时满足：

| 判定项 | 特征 |
| --- | --- |
| 界面工程路径 | `ui_prj/<工程名>/jlui/design/` |
| 生成代码 | `setup_scr_<页面名>.c`，内容为 LVGL 接口调用 |
| 资源分区 | `UIPACKRES` |

杰理其余 UI 方案基于 ui_framework 体系，工程数据格式与本文档不兼容。

## 目录结构

| 路径 | 说明 |
| --- | --- |
| `SKILL.md` | 主文档。工程数据格式、布局修改方法、开发流程与约束条件 |
| `references/widget-fields.md` | 控件字段参考。各控件类型的专有字段、取值范围及官方文档索引 |
| `references/code-integration.md` | 代码对接参考。控件引用、线程约束、消息机制、事件回调、自定义绘制、问题定位 |
| `JLGuiBuilder_doc/` | JLGuiBuilder 官方文档离线副本 |

`SKILL.md` 为入口文档，其余文档由主文档在对应章节引出，无需通读。

## 部署

本仓库为纯 Markdown 文本，不含可执行脚本，不依赖特定运行环境。

### 通用 AI 编程助手

第一步，将仓库置于工程目录下：

```bash
git clone https://github.com/QiuHaihui2020/jieli-gui-builder-skill.git \
  <工程目录>/.ai/jieli-gui-layout
```

第二步，在助手的规则文件中声明引用路径：

```markdown
修改 JLGuiBuilder 界面工程（ui_prj 下的设计文件、custom 目录代码、
资源打包流程）前，先读取 .ai/jieli-gui-layout/SKILL.md。
```

规则文件的名称因工具而异，常见为 `AGENTS.md`、`CLAUDE.md`、`.cursorrules`、`.github/copilot-instructions.md`、`.windsurfrules`。

未配置规则文件时，可在需要时将 `SKILL.md` 作为上下文直接提供，该文档为自包含结构。

### Claude Code

置于 Skill 目录下即可由工具按需自动加载：

```bash
# 全局生效
git clone https://github.com/QiuHaihui2020/jieli-gui-builder-skill.git \
  ~/.claude/skills/jieli-gui-layout

# 仅对单个工程生效
git clone https://github.com/QiuHaihui2020/jieli-gui-builder-skill.git \
  <工程目录>/.claude/skills/jieli-gui-layout
```

Windows 平台的全局目录为 `C:\Users\<用户名>\.claude\skills\`。

`SKILL.md` 头部的 YAML 字段（`name`、`description`）用于该机制的触发判定，其余工具将其作为普通文本处理，不影响使用。

## 版权声明

`JLGuiBuilder_doc/` 为杰理官方文档站 <https://doc.zh-jieli.com/JLGuiBuilder/zh-cn/master/> 的离线副本，**著作权归珠海市杰理科技股份有限公司所有**，收录于此仅用于离线检索工具属性语义。

该目录为可选项，删除后不影响其余文档使用，文档中的引用处均同时给出在线地址。
