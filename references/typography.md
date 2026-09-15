# 排版知识：beamer 的坑与验证方法

**这份文档只讲"为什么"和"怎么验证"，不讲"能装多少"。**

容量数字（一行几字、一页几行）住在记忆里 —— 见 `base/styles/default.tex`
与 `base/templates/` 里各文件的注释。这里不重复它们，避免两处对不上。

## 一、为什么会静默裁掉内容

beamer 的 frame 高度是**硬的**。超出部分不会流到下一页，直接消失：

```
Overfull \vbox (5.5pt too high) detected at line 30
```

只有这一条警告，而且**有时连警告都没有**。所以任何一页改完都要渲染成图目视。

**实测的报错位置与溢出量的关系**（本设计标定时量过）：连续加行时，
第一条 overfull 只超 3~5pt（内容还看得见，但已经压到底了），
再往下每加一行多超约 22pt（那一行被裁掉）。**所以"没报错"不等于"装得下"**，
"报了一条小 overfull"也不等于"内容丢了"—— 必须看图。

## 二、折行比溢出更难发现

一行超限会折行。折行本身不算错，但**折行点不受控**，常见结果是
末尾两三个字被甩到第二行，孤零零一行，很难看。

真实例子：一句 42 字的文案折成 `……一部分原因是课程较` / `多。`——
第二行只有一个「多。」。

**修法有两个**，按优先级：

1. 把文案压短（首选）
2. 主动断成两行字数相近的短句（次选 —— 至少折行点是可控的）

### 每个模板的"一行"预算不同，别互相套用

带左侧固定列或标记的版式（表格、`\DeckTagged`、时间轴）都要先扣掉那一列的
宽度。所以：

- 想知道某个模板一行装几个字 → **读那个模板文件头部注释里写的数**
- 想知道基准（无左侧列）是多少 → `\SlideLineUnitsSafe`（推荐）、
  `\SlideLineUnits`（硬上限），两者都由风格文件末尾那段实测得出

**新模板的容量必须自己实测一次**（见第七节），不要从别的模板类推。

## 三、间距的真实行为

### 陷阱：`\vspace` 在段落起点会被丢弃

```latex
\vspace{0.9cm}    % ← 出现在段落起点时，TeX 在换页处把它丢掉，"设了却没生效"
\vspace*{0.9cm}   % ← 必须带星号
```

所以 `primitives.tex` 与 `styles/` 里凡是用于页面结构的间距都用了 `\vspace*`。
写新模板时也要注意。

### 主题会覆盖字体上下文

**实测踩过**：在 `\SlideTextFont`（12pt/17pt）之后直接读 `\baselineskip`，
拿到的不是 17pt 而是 13.6pt —— beamer 的主题与 itemize 模板会重设字体上下文。
所以**不要靠 `\baselineskip` 反推行距**，要实测（见第七节）。

## 四、表格

### 列宽不要手算

两个表格组件内部都用 `tabularx` 的 `X` 列，表宽自动等于 `\linewidth`。
手写 `p{9.40cm}` 这类固定宽度**很容易算错总宽** ——
实测超出的量恰好等于算错的那部分（427pt vs 404pt = 23pt overfull），
而且一旦溢出，长内容会溢出到页面外。

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

改用 `\DeckCourseRule`（内部单独发 `\noalign` + `\hrule`）。两个表格模板都用它。

### 行距可逐页覆盖，字号不行

```latex
% 页面文件开头
\setlength{\DeckTableLeading}{15pt}   % ✅ 允许
\fontsize{11}{15}\selectfont          % ❌ 破坏单一字号约束
```

## 五、图片

### 必须先缩放

原图动辄 4000+ 像素，直接入稿会让 PDF 膨胀到几十 MB。**先缩到长边约 2400px。**

手机拍的图片通常还需要：

- **旋转**：竖拍的照片在幻灯片上是横的（用 System.Drawing 的 `RotateFlip`）
- **裁边**：桌上、键盘、背景杂物入镜时要裁掉

**裁边前先用像素亮度采样定位边界**，不要目测。真实教训：目测裁掉右侧 12%，
实际键盘只占最外 **1.6%**（65px / 4096px），结果把证书的花边裁掉了。
亮度跳变很干净（189 → 144 就是白背景进入键盘），采样一次就能定位。

### 两图并排必须留足宽度余量

- 两图宽度之和**必须明显小于**版心宽，留 0.3cm 以上
- 只差 **0.03cm** 就会让第二张图掉到下一行，并报 **overfull vbox**
  ——报的是竖直溢出，看不出是宽度问题，非常难查
- 不要用 beamer 的 `columns`：栏间距会与给定宽度叠加，试三次都算不准
- 版心宽用 `\SlideTextWidth` 查，不要记数字

### 等高要靠显式 width

`\includegraphics[height=H]` 的宽度按原图比例推算。两张图比例不同又都想等高时，
必须给其中一张显式 `width`：`width = H × (该图宽/该图高)`。

## 六、超链接

`\DeckLink{URL}{文字}` 内部是 `\href`，beamer 自带 hyperref，**不需要额外宏包**。

**验证链接真的生效**（编译通过不代表链接存在）：

```powershell
pdftohtml -xml -f <页码> -l <页码> main.pdf out.xml
# 然后在 out.xml 里搜 <a href= 确认
```

**不要用文本搜索 PDF 判断链接**：PDF 默认用对象流压缩，`Subtype/Link` 和 URL
在原始字节里搜不到，会误判成"链接没生效"。

另外注意日志里出现 `hyperref Message: Stopped early` 时，要用 pdftohtml 复验一次。

## 七、字体

### 中文没有真斜体

微软雅黑只有 Regular / Bold 两档，**没有真斜体字形**。用 `\emph` / `\textit`
会得到合成的机械倾斜，中文看起来发虚发脏。**全篇禁用斜体。**

### 字号必须是标准尺寸

非标准字号（如 15pt）会让 LaTeX 报警告：

```
LaTeX Font Warning: Font shape `OT1/cmr/m/n' in size <15> not available
```

在 `\fontsize` 里做数学运算（如 `0.9\baselineskip`）也会触发。
需要特殊字号时用整数 pt 值。

### 数学模式会加载 Computer Modern

`$\cdot$` 这类数学符号会去加载 Computer Modern 字体，与 Segoe UI 不搭，
还会在非标准字号下报警告。用文本字符代替：
`\textperiodcentered` 而不是 `$\cdot$`。

## 八、编译工作目录（最容易踩的一条）

**TeX 的相对路径按「当前工作目录」解析，不是按被包含文件的位置。**

`config.tex` 住在 `<deck>/slide/.slide-forge/` 里，但编译时的 cwd 是
`<deck>/slide/`（`main.tex` 所在处）—— 所以 `config.tex` 里每一条
`\input` 都要写全 `.slide-forge/` 前缀。

```powershell
# ✅ 正确：cwd 是 <deck>/slide/
cd <deck>\slide; xelatex -interaction=nonstopmode main.tex

# ❌ 失败：cwd 是 <deck>/
cd <deck>; xelatex -interaction=nonstopmode slide\main.tex
```

**实测症状**：

```
! LaTeX Error: File `.slide-forge/config.tex' not found.
! Emergency stop.
```

### ⚠ 读日志前必须先删日志

**失败的 xelatex 往往根本不写 `main.log`**，上一次成功那份会**原样留着**。
拿 `Select-String -Path main.log` 去查，读到的是**过期证据** —— 会得出
"编译成功了""模板加载了"这种完全相反的结论。

这个坑在写验证脚本时实地踩到过：脚本断言"换个 cwd 会失败"，
结果因为读到旧日志而判成"没失败"。

```powershell
Remove-Item main.log -Force -ErrorAction SilentlyContinue
xelatex -interaction=nonstopmode main.tex
Select-String -Path main.log -Pattern "^! |Overfull|Font Warning"
```

**验证记忆真的加载了**：在当次生成的 `main.log` 里搜文件名。

```powershell
Select-String -Path main.log -Pattern "\.slide-forge"
# 有 (.slide-forge/templates/xxx.tex 这一行 = 加载成功
```

### 定义组件必须用 `\DeckDefine`

实测报错原文（用 `\newcommand` 定义已存在的命令时）：

```
! LaTeX Error: Command \DeckProbeAbsorbed already defined.
```

`\newcommand` 不能重复定义，`\renewcommand` 又要求命令已存在 ——
所以 `primitives.tex` 提供了 `\DeckDefine`：

```latex
\DeckDefine{\DeckTimeline}[2]{...}   % 新建、覆盖都行
```

**症状对照**：页面里出现 `Undefined control sequence. \Deck某名字`
说明该组件压根没被定义（记忆文件没加载，或名字写错）；
出现 `already defined` 说明你在记忆文件里用了 `\newcommand`。

## 九、怎么实测一个模板的容量

**不要猜，也不要从别的模板类推。** 步骤：

1. 造一个测试页，把目标模板的内容一路加到明显超出
2. 逐页 `pdftotext` 数**实际渲染出来的条数**（溢出会被静默裁掉，所以
   数出来的数就是真实能装下的数）
3. 同时看 `main.log` 里第一条 `Overfull \vbox` 出现在哪一条 ——
   那一条就是硬上限
4. 把两个数都写进模板文件头部注释：**硬上限**告诉人余量还有多少，
   **推荐值**（留余量后的）给用户看

数条数时用**西文标记**（如 `ROW1`、`ROW2`），不要用中文 ——
`pdftotext` 会在中文字符间插入空格，正则容易匹配不上。

```powershell
# 数某页实际渲染出几条
$txt = (pdftotext -f <页码> -l <页码> main.pdf - 2>$null) -join "`n"
([regex]::Matches($txt, 'ROW\d+')).Count
```

**注意**：`pdftotext` 返回的是**行数组**；对数组用 `-notmatch` 会返回
"不匹配的元素"而不是布尔值，断言会失真。要先 `-join` 成单个字符串。

## 十、工具速查

```powershell
# 编译（必须用 xelatex，中文字体与 hyperref 都需要）
# cwd 必须是 <deck>/slide/
Remove-Item main.log -Force -ErrorAction SilentlyContinue
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
