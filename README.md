# DSH SolidWorks 建模技能

DeepSeek Harness (DSH) 技能包：通过 Python (win32com) 桥接驱动 **SolidWorks 自动建模**。
只用 DeepSeek 官方 API，不接任何第三方模型 / 视觉端点。

## 功能

- 实时可视化建模：SolidWorks 前台窗口 + 等轴测视角 + 每步特征后自动缩放截图
- 单位约定：默认毫米（mm），用户没说明就是 mm
- 命名约定：自动保存为 `DSH_<名称>.sldprt`（中文名称保留中文）
- `doctor` 自检：环境缺什么一目了然（Python / pywin32 / mss / SW 连接 / 模板探测）

## 安装

1. Python 依赖：`python -m pip install pywin32 mss Pillow`
2. 手动启动一次 SolidWorks 到主界面（首次必须，让 COM 可用）
3. 把整个目录放进 DSH 技能扫描根 `~/.dsh/skills/solidworks-modeling/`（或用 dsh-skills 面板上传）
4. 自检：`python sw_bridge.py doctor`（期望 `"ok": true`、`"problems": []`）

> 技能入口是 `SKILL.md`，放进扫描根后 DSH 的 `/` 斜杠菜单会自动发现。
> 文中的 `<包目录>` 占位符需替换成实际目录（或直接用当前目录）。

## 使用

在 DSH 对话里直接说：

- "用 SolidWorks 建模一个五段阶梯轴"
- "建模一块 120×80 的板，厚度 10mm"

DSH 会生成建模脚本并调用 `sw_bridge.py run <script.py>` 驱动 SolidWorks，全程实时展示建模过程。

## 目录结构

| 文件 | 作用 |
|---|---|
| `SKILL.md` | 技能定义（DSH 读取入口） |
| `sw_bridge.py` | 执行器：doctor / run / show / status / open / save / export-pdf |
| `swapi.py` | 高层建模封装（DSH 生成的脚本只调这个） |
| `README_使用教程.md` | 给新用户的详细教学文档 |
| `examples/` | 示例建模脚本（轴 / 杯 / 板） |

## 兼容性

- Windows + SolidWorks 2023 SP03（实测），其他版本理论可用
- 需要 Python 3.8+
- 桥接底层：pywin32 (win32com) 驱动 SolidWorks COM API，全程本地执行
