# SolidWorks 自动化建模桥接包 — 通用版 使用教程

> 本包把 **DeepSeek Harness（DSH）** 和 **SolidWorks** 连起来：你只需用中文
> 描述要做的零件（"做一个直径 50 的圆球"、"做一个五段阶梯轴"），AI 自动生成
> 建模脚本并驱动 SolidWorks 实时建模，你全程能看到过程，最终得到 `.sldprt` 文件。
>
> **通用版特点**：不绑定某台电脑、某个 SolidWorks 版本、某个安装路径。
> 只要电脑装了 SolidWorks（2018~2024 均可），按本教程 10 分钟即可跑通。

---

## 目录

- [第 0 步：包结构说明](#第-0-步包结构说明)
- [第 1 步：安装 Python 与依赖（5 分钟）](#第-1-步安装-python-与依赖)
- [第 2 步：安装/启动 SolidWorks](#第-2-步安装启动-solidworks)
- [第 3 步：环境自检 doctor（2 分钟）](#第-3-步环境自检-doctor)
- [第 4 步：第一次建模（10 分钟上手）](#第-4-步第一次建模)
- [第 5 步：在 DSH 里用中文提示词建模（核心用法）](#第-5-步在-dsh-里用中文提示词建模核心用法)
- [第 6 步：提示词怎么写 —— 教学案例](#第-6-步提示词怎么写--教学案例)
- [第 7 步：常用命令速查](#第-7-步常用命令速查)
- [第 8 步：常见问题排查（FAQ）](#第-8-步常见问题排查-faq)
- [附录 A：swapi 高层 API 速查](#附录-aswapi-高层-api-速查)
- [附录 B：支持/不支持的建模操作](#附录-b支持不支持的建模操作)

---

## 第 0 步：包结构说明

拿到手的包里有这些文件：

```
SolidWorksBridge_通用版\
├── sw_bridge.py          # 执行器：DSH 通过它指挥 SolidWorks（勿删）
├── swapi.py              # 高层建模 API：建模脚本都调它（勿删）
├── solidworks-modeling.md # AI 的技能文件：装到 DSH 的 skills 目录后，
│                           # AI 就知道怎么建模（详见第 5 步）
├── README_使用教程.md     # 就是你正在看的这个文件
└── examples\             # 示例脚本：直接可运行的建模样例
    ├── build_plate.py    #   例1：带孔板（拉伸+切除）
    ├── build_axis.py     #   例2：五段阶梯轴+键槽（旋转+槽口切除）
    └── build_cup.py      #   例3：杯子（旋转+切除挖空）
```

> 这些文件可以放在电脑的任何位置（桌面、D 盘、U 盘都行），**不需要安装**。
> 下文用 `<包目录>` 表示这个文件夹的实际路径，例如
> `C:\Users\你的名字\Desktop\SolidWorksBridge_通用版`。

---

## 第 1 步：安装 Python 与依赖

本包用 Python 驱动 SolidWorks（因为 SolidWorks 的 COM 接口在 Python 下最稳定，
且不挑版本）。

1. **装 Python 3.8 以上**（建议 3.10~3.14）：
   - 官网下载：https://www.python.org/downloads/
   - 安装时**务必勾选 "Add python.exe to PATH"**（加到系统路径），
     这样在任何目录敲 `python` 都能用。
   - 装完验证：打开"命令提示符"（Win+R → 输入 `cmd` → 回车），输入：
     ```
     python --version
     ```
     看到 `Python 3.x.x` 即成功。

2. **安装三个依赖库**（pywin32 负责连 SolidWorks，mss 负责截图，
   Pillow 负责图片处理）。在命令提示符里执行：
   ```
   pip install pywin32 mss Pillow
   ```
   看到 `Successfully installed ...` 即成功。
   > 如果提示 pip 不是命令，先执行 `python -m pip install --upgrade pip`。

3. **重启电脑或注销一次**（pywin32 有时需要重启才能生效）。

---

## 第 2 步：安装/启动 SolidWorks

1. 确保电脑上已安装 SolidWorks（本包支持 2018 ~ 2024，标准版/专业版/白金版均可）。
2. **手动启动一次 SolidWorks**，让它完全打开到主界面，再最小化。
   > 第一次务必手动启动：一方面让 SolidWorks 完成初始化，另一方面
   > 让 Windows 记住它的启动方式，后面脚本才能自动唤醒它。

---

## 第 3 步：环境自检 doctor

包内置了自检命令，一条命令告诉你环境是否就绪、缺什么。

在命令提示符里执行（把路径换成你的实际路径）：
```
python "C:\...\SolidWorksBridge_通用版\sw_bridge.py" doctor
```

预期输出（JSON）：
```json
{
  "ok": true,
  "python": "3.14.0",
  "pywin32": "C:\\...\\win32com\\__init__.py",
  "mss": "installed",
  "pillow": "installed",
  "solidworks": {
    "connected": true,
    "revision": "30.0.0",
    "version_major": 30
  },
  "template": "C:\\ProgramData\\SolidWorks\\SOLIDWORKS 2022\\templates\\gb_part.prtdot",
  "problems": []
}
```

- `"ok": true` 且 `"problems": []` → 环境就绪，跳到第 4 步。
- 若 `"ok": false`，看 `"problems"` 里的提示，例如：
  - `pywin32 未安装: pip install pywin32`
  - `无法连接 SolidWorks: 请先启动 SolidWorks`
  - `未找到零件模板`（极少数情况模板目录特殊，见 FAQ）
- `"template"` 是自动探测到的零件模板路径，**不用你手动配置**，这就是
  通用版和旧版最大的区别。

---

## 第 4 步：第一次建模

不用写代码，先跑一个现成示例感受一下：

```
python "<包目录>\sw_bridge.py" run "<包目录>\examples\build_plate.py"
```

你会看到：SolidWorks 窗口自动跳到前台 → 新建零件 → 画草图（正视于平面）→
拉伸 → 挖孔 → 每步自动居中展示 → 最后等轴测视角 + 截图。

执行完终端会输出 JSON，其中 `"screenshot_path"` 指向一张 PNG 截图，
打开看看，一个带孔板就建好了（同时 `<包目录>\examples\DSH_带孔板.sldprt`
就是可交付的 SolidWorks 文件）。

---

## 第 5 步：在 DSH 里用中文提示词建模（核心用法）

本包是给 **DeepSeek Harness（DSH）** 用的。DSH 是运行 AI 助手的桌面工具，
AI 通过 `pwsh` 工具执行 `sw_bridge.py` 来驱动 SolidWorks。

要让 AI 会建模，还需要把技能文件告诉它：

1. 找到 DSH 的技能目录（一般是 `C:\Users\<你的名字>\.dsh\skills\`）。
2. 把包里的 `solidworks-modeling.md` 复制进去（若已有同名文件则覆盖）。
3. 重启 DSH，让技能生效。

之后在 DSH 的对话框里直接说中文需求即可，例如：

> **"用 SolidWorks 建模一个直径 50mm、高 80mm 的圆柱，底部加一个直径 60mm、高 8mm 的法兰盘，保存为 DSH_法兰圆柱"**

AI 会自动完成：生成脚本 → 调用 sw_bridge → 驱动 SolidWorks 建模 →
实时展示过程 → 保存文件。你全程用中文对话，不用写一行代码。

---

## 第 6 步：提示词怎么写 —— 教学案例

> 提示词写得好，模型就建得准。核心是**把尺寸、形状、特征说清楚**。

### 案例 1：最简单的板（新手必试）

```
用 SolidWorks 建模一个长 120、宽 80、厚 10 的板，四个角倒圆角 R5，
保存为 DSH_底板。
```

**要点**：给出 长/宽/厚 三个尺寸 + 圆角半径。AI 会：新建零件 →
前视基准面画 120×80 矩形 → 拉伸 10 → 四条棱边倒 R5 圆角 → 保存。

### 案例 2：阶梯轴（教学重点：旋转体）

```
建模一个五段阶梯轴，总长 150：
- 第1段：直径 20，长 30
- 第2段：直径 25，长 20
- 第3段：直径 30，长 50
- 第4段：直径 25，长 20
- 第5段：直径 20，长 30
中间带键槽：宽 8，长度（圆心距）25，右侧圆心距右端面 7.5。
保存为 DSH_五段阶梯轴。
```

**要点**：
- 轴是回转体，AI 会用"旋转特征"：画上半截面轮廓 + 旋转轴，一次成型，比
  一段段拉伸精确得多。
- 键槽的"长度"默认指**两个半圆圆心之间的距离**（本包约定），
  说清楚"右侧圆心距右端面 7.5"就能正确定位。
- 这样复杂度的提示词，AI 一次成功率很高；万一失败，把报错贴回去，
  说"按这个错误修正重试"即可。

### 案例 3：杯子（教学重点：旋转 + 挖空）

```
建模一个马克杯：外径 80、高 100、壁厚 3，底部厚 4，保存为 DSH_杯子。
```

**要点**：杯子是"旋转外轮廓 + 旋转切除内腔"。AI 会画两个截面
（外壁、内壁），用旋转凸台和旋转切除生成。如果只说"杯子"不给尺寸，
AI 会按常见尺寸建，但最好给全尺寸。

### 案例 4：带孔板（教学重点：面上草图 + 贯穿切除）

```
建模一个 200×100×10 的板，四角 M6 通孔（直径 6），孔心距边 15，
保存为 DSH_带孔板。
```

**要点**：给出孔直径和孔心位置（"距边 15"）。AI 会在板的上表面画四个圆，
然后做"完全贯穿"切除。

### 案例 5：齿轮（教学重点：复杂轮廓）

```
建模一个直齿圆柱齿轮：模数 2、齿数 20、齿宽 30，保存为 DSH_齿轮。
```

**要点**：说出 模数/齿数/齿宽，AI 会算出分度圆直径
（模数×齿数 = 40）并用渐开线近似齿形。复杂轮廓一次成功率略低，
失败时把报错发给 AI 让它修正。

### 提示词写作万能模板

```
用 SolidWorks 建模一个【零件名】：
- 主要尺寸：【长×宽×高 / 直径×长度 / 内外径×高度】= 具体数字
- 特征：【孔 / 槽 / 圆角 R? / 倒角 C? / 螺纹 / 阵列…】，位置用
  "距哪条边多远"、"圆心在哪" 描述
- 保存为：DSH_【名字】
```

**三个禁忌**：
1. 不要说"随便""大概"——AI 会猜，结果可能不是你想要的。
2. 不要同时要求太多特征（圆角+倒角+螺纹+阵列一次做完容易失败），
   分两轮："先建主体，保存；再在上面加孔"。
3. 不要急着让 AI 做质量校验（测量重量/体积）——本包约定建模完跳过校验，
   以免 SolidWorks 卡死。要重量的话单独说"用 sw_bridge.py massprops"。

---

## 第 7 步：常用命令速查

所有命令都在命令提示符里执行（`<包目录>` 换成实际路径）：

| 目的 | 命令 |
|---|---|
| 环境自检 | `python "<包目录>\sw_bridge.py" doctor` |
| 连接状态 | `python "<包目录>\sw_bridge.py" status` |
| 运行建模脚本 | `python "<包目录>\sw_bridge.py" run <脚本.py>` |
| 打开模型 | `python "<包目录>\sw_bridge.py" open <模型.sldprt>` |
| 另存为 | `python "<包目录>\sw_bridge.py" save <输出.sldprt>` |
| 导出 PDF | `python "<包目录>\sw_bridge.py" export-pdf <输出.pdf>` |
| 展示成品 | `python "<包目录>\sw_bridge.py" show` |
| 关闭当前文档 | `python "<包目录>\sw_bridge.py" close` |
| 质量属性 | `python "<包目录>\sw_bridge.py" massprops` |

> 这些命令在 DSH 里由 AI 自动调用，你不需要手动敲；
> 上面列出来是方便你自己测试排查。

---

## 第 8 步：常见问题排查（FAQ）

**Q1：doctor 报 `pywin32 未安装`**
运行 `pip install pywin32`，然后**重启电脑**再试。

**Q2：doctor 报 `无法连接 SolidWorks`**
先手动双击打开 SolidWorks，等它完全进入主界面，再重新跑 doctor。

**Q3：报错 `TYPE_E_ELEMENTNOTFOUND`**
这是 PowerShell 直连 COM 的经典报错，**与本包无关**——本包已改用
Python win32com 后期绑定规避。如果还看到这个错误，说明脚本没走对路径，
确认命令用的是 `python "<包目录>\sw_bridge.py"` 而不是其他方式。

**Q4：`未找到零件模板`（极少数情况）**
本包自动探测常见位置：`C:\ProgramData\SolidWorks\SOLIDWORKS*`、任意盘符的
`*SOLIDWORKS*` 目录。若你的 SolidWorks 装在很特殊的位置，手动告诉它模板：
```
python "<包目录>\sw_bridge.py" new "C:\你的模板目录\Part.prtdot"
```

**Q5：建模到一半失败/特征没生成**
把终端里的 JSON 报错（`"error"` 字段）复制给 AI，说"修正这个错误重试"。
常见原因：轮廓没闭合（AI 会自动合并端点）、尺寸冲突、旧版本没有某个 API
（见附录 B）。

**Q6：窗口在建模时不显示 / 最小化了**
本包会自动把 SolidWorks 窗口调到前台并最大化。如果被其他窗口挡住，
点击任务栏的 SolidWorks 即可。建模完成后 `sw_bridge.py run` 会自动截图，
看截图确认结果也行。

**Q7：保存的文件在哪？**
默认保存在**脚本所在目录**（`<包目录>\examples\` 或你的脚本目录），
文件名是 `DSH_<名字>.sldprt`。

**Q8：能不能用中文以外的版本（英文版 SolidWorks）？**
可以。本包不依赖中文界面：草图按序号选择（`select_sketch_by_index`），
基准面用英文名 "Front Plane"（SolidWorks 任何语言版本内部都是英文名），
模板自动匹配 `Part.prtdot`/`gb_part.prtdot`。

**Q9：2018 的 SolidWorks 能用吗？**
能。两侧对称拉伸的枚举值 2018 是 5、2020+ 是 6，本包按版本号自动切换；
倒角 API 新旧版本方法名不同，本包自动回退。

---

## 附录 A：swapi 高层 API 速查

建模脚本（DSH 自动生成，或你手写）里常用的方法，全部以**毫米**为单位：

```python
import swapi
m = swapi.new_part()                  # 新建零件（自动找模板）
m.begin_sketch("Front Plane")         # 在前视基准面开草图（自动正视+居中）
m.rect(cx, cy, w, h)                  # 中心矩形
m.circle(cx, cy, r)                   # 圆
m.line(x1, y1, x2, y2)                # 直线
m.polyline([(x1,y1), (x2,y2), ...])   # 折线（连成连续线）
m.centerline(x1, y1, x2, y2)          # 中心线（旋转轴）
m.end_sketch()                        # 结束草图（自动合并端点）
m.extrude(depth, symmetric=False)     # 拉伸凸台
m.cut(depth=10, through=True)         # 切除 / 完全贯穿
m.revolve(360, cut=False)             # 旋转凸台/切除（需先画中心线）
m.fillet(radius, [(x,y,z), ...])      # 圆角：边上的采样点坐标
m.chamfer(width, [(x,y,z), ...], 45)  # 倒角
m.save(path)                          # 另存为 .sldprt
m.screenshot(path)                    # 对 SolidWorks 窗口截图 PNG
m.bring_to_front()                    # 窗口置前+最大化
m.set_view_iso()                      # 等轴测+居中+缩放1.2
m.zoom_to_fit()                       # 缩放适应窗口
```

坐标说明：`begin_sketch("Front Plane")` 后草图在 XY 平面（Z=0）；
`begin_sketch_on_face(x, y, z)` 在实体表面上开草图（坐标是毫米）。

## 附录 B：支持/不支持的建模操作

**已实测支持（本包内置、可靠）**：
- 拉伸凸台/切除（含两侧对称、完全贯穿）
- 旋转凸台/切除（轴类、球、法兰、杯子）
- 圆角（恒定半径）、倒角（角度-距离）
- 直槽口/键槽（FullLength 模式）
- 在实体面上画草图并切除（孔）
- 中心矩形/圆/直线/折线/中心线草图

**已知坑（文档化，AI 会避开）**：
- 键槽用 CenterCenter 模式会生成错误几何 → 只用 FullLength 模式
- 直槽口 API 是 2016+ 才有的，2015 及以下需手工拼槽
- 中文草图名依赖语言版本 → 一律按序号选草图
- 质量属性校验容易卡死 SolidWorks → 默认跳过

**暂不支持（别让 AI 尝试）**：
- 装配体配合/运动仿真
- 复杂曲面（放样、扫掠未充分测试）
- 钣金
- 工程图标注

---

*祝建模愉快！有问题把终端报错发给 AI，它会帮你修。*
