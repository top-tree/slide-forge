# 排版细节与已知陷阱

这份文档记录的是**实测数据**和**踩过的坑**，不是设计理念（理念见 `SKILL.md`）。
所有数字都是在 `aspectratio=169`、正文 12pt、左右边距 0.90cm 的实际编译中量出来的。

## 一、容量上限（最常踩）

| 项 | 实测值 |
|---|---|
| 正文区宽度 | 14.25 cm（404 pt） |
| 一行可容纳 | **约 30 个全角字符**（西文与数字约占半角） |
| 一页正文行数 | **最多 6 行**（`\DeckLine` 这一级） |
| 数值表 | 5 行数据 + 表头 |
| 列式表 | 8 行数据 + 表头 |
| 页面高度 | 255 pt（7.2 cm） |

> **每个模板的"一行"预算不同，别互相套用。** 带左侧固定列/标记的模板
> （表格、T2 `\DeckTagged`、项目本地的 T9 时间轴）都要先扣掉那一列的宽度。
> 实测：T9 的节点列吃掉约 4 个全角字，**一行只装得下约 24 个全角字符** ——
> 拿正文的 30 去写，25 字就折行了。新模板的容量必须自己量一次再写进参考文件。

### 为什么会静默裁掉内容

beamer 的 frame 高度是**硬的**。超出部分不会流到下一页，直接消失：

```
Overfull \vbox (5.5pt too high) detected at line 30
```

只有这一条警告，而且**有时连警告都没有**。所以任何一页改完都要渲染成图目视：

```powershell
pdftocairo -png -r 130 -f <页码> -l <页码> main.pdf out
```

### 折行比溢出更难发现

一行超过 30 字会折行。折行本身不算错，但**折行点不受控**，常见结果是
末尾两三个字被甩到第二行，孤零零一行，很难看。

真实例子：一句 42 字的文案折成 `……一部分原因是课程较` / `多。`——
第二行只有一个「多。」。修法是**把文案压到 30 字以内**，或主动断成两行字数相近的短句。

### 写文案时先数字数

中文一个字算 1，西文一个字母 / 数字 / 半角标点算约 0.5。
一行预算 30，留 1~2 的余量。

### 表格里的行宽又是另一个数（实测，别套用正文的 30）

表格第二列**不等于**正文宽：要先扣掉左列标签和列间距。实测
（`pdftotext -bbox` 量出来的真实坐标，aspectratio=169 / 正文 12pt）：

| 项 | 实测值 |
|---|---|
| 正文左边界 | 25.51 bp |
| 表格右列左边界 | 105.82 bp（左列标签 23.91 + 列间距） |
| 表格右列可用宽 | 约 324 bp |
| 一个全角字 | 11.34 bp |
| 一个西文字母 / 数字 | 约 6.26 bp |

换算成好用的写法：**右列一行只装得下 28 个「宽度单位」**，
其中全角字 = 1、西文与数字 = 0.55。**写文案按 27 单位切句。**

> 30 那个数是正文（`\DeckLine` 一级）的预算，用在表格右列上必然折行。

**测量方法**（比目测靠谱，也比反复编译试更快）：

```powershell
pdftocairo -png -r 150 -f 2 -l 2 main.pdf p     # 先目视看有没有掉行
pdftotext -bbox-layout -f 2 -l 2 main.pdf out.xml
# 读 out.xml，逐行算 xMax - xMin；贴着上方上限（约 324）就是快折行了
```

折行的表现是**多出一行**，不是宽度报错 —— 日志里只有 `Overfull \vbox`，
甚至没有警告。所以每次都要数行数：三块内容各占两行才对齐，
其中一块变成三行，整页节奏就塌了。

## 二、间距的真实行为

| 命令 | 实际值 | 说明 |
|---|---|---|
| `\DeckLine` 条目间 | 0.30 cm | 同组内容 |
| `\DeckGap` | 0.40 cm | 换组。这是**唯一**的分组手段 |
| `\SlideBodyTopSkip` | 0.40 cm | 页标题到正文第一行 |
| `\DeckCite` 两行间 | 0.12 cm | 同一个条目的标题行与说明行 |
| `\DeckCite` 条目间 | 0.34 cm | |

### 陷阱：`\vspace` 在段落起点会被丢弃

```latex
\vspace{0.9cm}    % ← 出现在段落起点时，TeX 在换页处把它丢掉，"设了却没生效"
\vspace*{0.9cm}   % ← 必须带星号
```

`style.tex` 里凡是用于页面结构（`\DeckGap`、`\DeckSlide`）的间距都用了 `\vspace*`。

## 三、表格

### 列宽不要手算

`\DeckCourseTable` / `\DeckListTable` 内部用 `tabularx` 的 `X` 列，
表宽自动等于 `\linewidth`。手写 `p{9.40cm}` 这类固定宽度**很容易算错总宽**——
实测超出的量恰好等于算错的那部分（427pt vs 404pt = 23pt overfull），
而且一旦溢出，长课名会溢出到页面外。

### 两种表的列宽方向不同，不能混用

| 组件 | 列结构 | 适用 |
|---|---|---|
| `\DeckCourseTable` | 名称占满 + 数值**右**对齐 | 成绩、指标（长名称 + 短数值） |
| `\DeckListTable` | 标签窄左 + 文字占满**左**对齐 | 奖项、事件（短标签 + 长文字） |

**用错的症状**：长文字被右对齐得零碎，像被拆开的碎片，且可能溢出列宽。

### `\hline` 会让表格深度异常

```latex
\hline\noalign{\vspace{0.14cm}}   % ← 实测让 5 行表的深度变成 77pt
```

改用 `\DeckCourseRule`（内部单独发 `\noalign` + `\hrule`）画细线。

### 行距可逐页覆盖，字号不行

```latex
% 页面文件开头
\setlength{\DeckTableLeading}{15pt}   % ✅ 允许
\fontsize{11}{15}\selectfont          % ❌ 破坏单一字号约束
```

## 四、图片

### 必须先缩放

原图动辄 4000+ 像素，直接入稿会让 PDF 膨胀到几十 MB。**先缩到长边约 2400px。**

手机拍的图片通常还需要：
- **旋转**：竖拍的照片在幻灯片上是横的（用 System.Drawing 的 `RotateFlip`）
- **裁边**：桌上、键盘、背景杂物入镜时要裁掉

**裁边前先用像素亮度采样定位边界**，不要目测。真实教训：目测裁掉右侧 12%，
实际键盘只占最外 **1.6%**（65px / 4096px），结果把证书的花边裁掉了。
亮度跳变很干净（189 → 144 就是白背景进入键盘），采样一次就能定位。

### 两图并排必须留足宽度余量

```latex
\DeckFigures{左图}{右图}{左图高度}{右图宽度}
```

- 两图宽度之和 **必须明显小于 14.25cm**，留 0.3cm 以上
- 只差 **0.03cm** 就会让第二张图掉到下一行，并报 **overfull vbox**
  ——报的是竖直溢出，看不出是宽度问题，非常难查
- 不要用 beamer 的 `columns`：栏间距会与给定宽度叠加，试三次都算不准

### 等高要靠显式 width

`\includegraphics[height=H]` 的宽度按原图比例推算。两张图比例不同又都想等高时，
必须给其中一张显式 `width`：`width = H × (该图宽/该图高)`。

## 五、超链接

`\DeckLink{URL}{文字}` 内部是 `\href`，beamer 自带 hyperref，**不需要额外宏包**。

**验证链接真的生效**（编译通过不代表链接存在）：

```powershell
pdftohtml -xml -f <页码> -l <页码> main.pdf out.xml
# 然后在 out.xml 里搜 <a href= 确认
```

**不要用文本搜索 PDF 判断链接**：PDF 默认用对象流压缩，`Subtype/Link` 和 URL
在原始字节里搜不到，会误判成"链接没生效"。

另外注意日志里出现 `hyperref Message: Stopped early` 时，要用 pdftohtml 复验一次。

## 六、字体

### 中文没有真斜体

微软雅黑只有 Regular / Bold 两档，**没有真斜体字形**。用 `\emph` / `\textit`
会得到合成的机械倾斜，中文看起来发虚发脏。

**全篇禁用斜体。** 强调只用 `\textbf` 加粗。

### 字号必须是标准尺寸

非标准字号（如 15pt）会让 LaTeX 报警告：

```
LaTeX Font Warning: Font shape `OT1/cmr/m/n' in size <15> not available
```

在 `\fontsize` 里做数学运算（如 `0.9\baselineskip`）也会触发。
需要特殊字号时用整数 pt 值。

### 数学模式会加载 Computer Modern

`$\cdot$` 这类数学符号会去加载 Computer Modern 字体，与 Segoe UI 不搭，
还会在非标准字号下报警告。用文本字符代替：`\textperiodcentered` 而不是 `$\cdot$`。

## 七、`.slide-forge/` 与编译工作目录（三层架构的坑）

项目的 `main.tex` 里写的是 `\input{style.tex}`，而 `style.tex` 末尾用
`\IfFileExists{.slide-forge/templates/format.tex}` 加载本 deck 的 .slide-forge/ —— 这些都是**相对路径**。

**TeX 的相对路径按「当前工作目录」解析，不按被包含文件的位置。** 所以：

```powershell
# ✅ 正确：cwd 是 <项目>/slide/
cd <项目>\slide; xelatex -interaction=nonstopmode main.tex

# ❌ 失败：cwd 是 <项目>/
cd <项目>; xelatex -interaction=nonstopmode slide\main.tex
```

**实测症状**（别按猜的写文档）：失败报的是

```
! LaTeX Error: File `style.tex' not found.
! Emergency stop.
```

**不是** `Undefined control sequence` —— 因为它在 `\input{style.tex}` 这一步就停了，
根本没走到 `\DeckMoment` 那一行。看到 `Undefined control sequence` 才该去查
`.slide-forge/templates/components.tex` 里到底有没有那个命令。

### ⚠ 读日志前必须先删日志

**失败的 xelatex 往往根本不写 `main.log`**，所以上一次成功那份会**原样留着**。
拿 `Select-String -Path main.log` 去查，读到的是**过期证据** —— 会得出
"编译成功了""本地层加载了"这种完全相反的结论。

这个坑在写验证脚本时实地踩到过一次：脚本断言"换个 cwd 会失败"，
结果因为读到旧日志而判成"没失败"。

```powershell
Remove-Item main.log -Force -ErrorAction SilentlyContinue
xelatex -interaction=nonstopmode main.tex
Select-String -Path main.log -Pattern "^! |Overfull|Font Warning"
```

**验证本地层真的加载了**：在当次生成的 `main.log` 里搜文件名。

```powershell
Remove-Item main.log -Force -ErrorAction SilentlyContinue   # 见上文，先删！
xelatex -interaction=nonstopmode main.tex
Select-String -Path main.log -Pattern "\.slide-forge|evolved/templates"
# 有 (.slide-forge/templates/components.tex 这一行 = 本地层加载成功
# 若出现 evolved/templates —— 说明有人把它写回了加载链，那是 bug：
#   仓库不参与编译，写回去会让吸收静默改掉所有已有 deck 的排版
```

### 在 `.slide-forge/` 里定义组件必须用 `\DeckDefine`

实测报错原文（用 `\newcommand` 在项目里覆盖吸收来的组件时）：

```
! LaTeX Error: Command \DeckProbeAbsorbed already defined.
```

`\newcommand` 不能重复定义，`\renewcommand` 又要求命令已存在 ——
所以 `base/templates/style.tex` 提供了 `\DeckDefine`：

```latex
\DeckDefine{\DeckTimeline}[2]{...}   % 新建、覆盖都行（= providecommand + renewcommand）
```

**症状对照**：页面里出现 `Undefined control sequence. \Deck某名字`
说明该组件压根没被定义（层级没加载，或名字写错）；
出现 `already defined` 说明你在 `.slide-forge/` 里用了 `\newcommand` 去覆盖。

## 八、工具速查

```powershell
# 编译（必须用 xelatex，中文字体与 hyperref 都需要）
# cwd 必须是 <项目>/slide/
xelatex -interaction=nonstopmode main.tex

# 检查错误 / 警告 / 溢出
Select-String -Path main.log -Pattern "^! |Overfull|Font Warning"

# 渲染某页为图，目视确认
pdftocairo -png -r 130 -f 3 -l 3 main.pdf out

# 提取某页文字（确认内容真的渲染出来了）
pdftotext -layout -f 3 -l 3 main.pdf -

# 验证超链接
pdftohtml -xml -f 3 -l 3 main.pdf out.xml

# 页数
pdfinfo main.pdf | Select-String Pages
```

**用 `pdftocairo` 而不是把 PDF 直接当图片读**：PDF 不是受支持的图片格式，
必须先栅格化。
