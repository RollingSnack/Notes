---
Title: Python 101 - 如何生成 PDF
Tags: [类型/译文, 类型/教程, 编程语言/Python, PDF]
Original Link: https://www.blog.pythonlibrary.org/2021/09/28/python-101-how-to-generate-a-pdf/
Original Title: Python 101 - How to Generate a PDF
---

# Python 101 - 如何生成 PDF

---

**便携式文档格式**（**Portable Document Format**, **PDF**）非常流行，能跨越多个平台分享文档。
其目标是创建一种文档，使它能在不同平台上看上去一样，能在不同打印机上打印得一样（或非常相似）。
该格式最初由 Adobe 开发，不过现在已经开源了。

Python 有许多库，它们可以帮助创建新的 PDF，或导出部分已经存在的 PDF。
目前没有可用于就地（in-place）编辑 PDF 的 Python 库。
这里有一些可用的包：

- **ReportLab** - 用于创建 PDF
- **pdfrw** - 用于切分，合并，水印和旋转 PDF
- **PyPDF2 / PyPDF4** - 用于切分，合并，水印和旋转 PDF
- **PDFMiner** - 用于从 PDF 中提取文本

Python 还有很多其他的 PDF 包。

在本文中，你将会学习如何用 ReportLab 创建一个 PDF。
ReportLab 包大约 2000 年时就存在了。
它有一个开源的版本，也有个包含一些额外特性的付费商业版。
这里将要学习的是其开源版本。

在本文中，你将会学习如下内容：

- 安装 ReportLab
- 使用画布（Canvas）创建一个简单的 PDF
- 使用画布（Canvas）创建绘图（Drawing）并添加图像（Image）
- 使用 PLATYPUS 创建多页文档
- 创建表格

ReportLab 可以生成你能想象的几乎所有种类的报告。
这篇文章不会覆盖 ReportLab 提供的所有特性，不过将带你了解 ReportLab 到底多有帮助。

让我们开始吧！

## 安装 ReportLab

可以使用 pip 安装 ReportLab。

```Python
python3 -m pip install reportlab
```

ReportLab 依赖 Pillow 包，它是一个 Python 的图像处理库。
如果系统没有该依赖的话，它也会被一并安装。

现在已经安装好 ReportLab 了，那就已经准备好学习如何创建一个简单的 PDF 了。

## 使用画布创建一个简单的 PDF

用 ReportLab 包创建 PDF 有两种方法。
低级别的方法是在画布上画。
这允许在页面的指定坐标上绘制。
PDF 在内部以 **点**（**points**）丈量其大小。
每英寸有 72 个点。
一封信大小的页面是 612 x 792 个点。
不过，默认页面大小是 A4。
也有很多默认可选的页面尺寸，或创建自己的页面尺寸。

看看代码，总是很容易了解它是怎样工作的。
创建一个名为 `hello_reportlab.py` 的新文件，它的代码：

```Python
# hello_reportlab.py

from reportlab.pdfgen import canvas

my_canvas = canvas.Canvas("hello.pdf")
my_canvas.drawString(100, 750, "Welcome to Reportlab!")
my_canvas.save()
```

这会生成一个 A4 大小的 PDF。

它创建了一个 `Canvas()` 对象，接受要创建的 PDF 的文件路径。
为了向 PDF 添加一些文字，使用 `drawString()`。
这段代码告诉 ReportLab，在距离页面左侧 100 点、底部 750 点的地方开始描绘文字。
如果打算在 (0,0) 开始绘画，文字就会出现在页面的左下角。
可以通过设置画布参数 `bottomup` 为 0 来更改开始绘制时的坐标。

最后一行将 PDF 保存到了硬盘。
不要忘了它，否则将看不到所创建的 PDF。

当打开 PDF 时，它看上去像这样：

![[译文/Python 101 - 如何生成 PDF/Figure-1.png]]

*Hello World 出现在 ReportLab 画布上*

虽然这表明了用 ReportLab 创建 PDF 有多简单，但这个例子有点无聊。

你还可以在画布上画线、画形状以及使用各种字体。
要学习如何使用，创建一个名为 `canvas_form.py` 的新文件，并在里面输入这段代码：

```Python
# canvas_form.py

from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas

def form(path):
    my_canvas = canvas.Canvas(path, pagesize=letter)
    my_canvas.setLineWidth(.3)
    my_canvas.setFont('Helvetica', 12)
    my_canvas.drawString(30, 750, 'OFFICIAL COMMUNIQUE')
    my_canvas.drawString(30, 735, 'OF ACME INDUSTRIES')
    my_canvas.drawString(500, 750, "12/12/2010")
    my_canvas.line(480, 747, 580, 747)
    my_canvas.drawString(275, 725, 'AMOUNT OWED:')
    my_canvas.drawString(500, 725, "$1,000.00")
    my_canvas.line(378, 723, 580, 723)
    my_canvas.drawString(30, 703, 'RECEIVED BY:')
    my_canvas.line(120, 700, 580, 700)
    my_canvas.drawString(120, 703, "JOHN DOE")
    my_canvas.save()

if __name__ == '__main__':
    form('canvas_form.pdf')
```

这里从 `reportlab.lib.pagesizes` 导入了 `letter` 尺寸，里面还有很多其他尺寸可供使用。
然后在 `form()` 函数里，实例化 `Canvas()` 时设置了 `pagesize`。
接着，用 `setLineWidth()` 去设置线宽，它会在画线时被使用。
再然后，字体被改成了Helvetica 字体，大小占 12 个点。

余下的代码是一系列不同坐标处的字符串绘制和这这那那的线。
当绘制一条 `line()`，传递的是开始坐标（x/y 位置）和结束坐标（x/y 位置），ReportLab 会用设置的线宽画线。

当你打开这个 PDF，会看到如下内容：

![[译文/Python 101 - 如何生成 PDF/Figure-2.png]]

*使用 ReportLab Canvas 创建的表格*

这看上去很不赖。
但如果想往报告上画点什么，又或者加上一个 logo 或一些照片呢？
接下来让我们看看怎么实现！

## 使用画布创建绘图并添加图像

ReportLab 的 `Canvas()` 很灵活。
它允许绘制不同图形，运用不同色彩，改变线宽等等。
创建一个新文件，取名 `drawing_polygons.py`，并添加如下代码来演示看看这些特性：

```Python
# drawing_polygons.py

from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas

def draw_shapes():
    my_canvas = canvas.Canvas("drawing_polygons.pdf")
    my_canvas.setStrokeColorRGB(0.2, 0.5, 0.3)
    my_canvas.rect(10, 740, 100, 80, stroke=1, fill=0)
    my_canvas.ellipse(10, 680, 100, 630, stroke=1, fill=1)
    my_canvas.wedge(10, 600, 100, 550, 45, 90, stroke=1, fill=0)
    my_canvas.circle(300, 600, 50)
    my_canvas.save()

if __name__ == '__main__':
    draw_shapes()
```

在这里创建了一个 `Canvas()` 对象，和之前一样。
可以使用 `setStrokeColorRGB()` 搭配 0 到 1 之间的 RGB 的值来改变边框的颜色。

接下来的几行代码创建了不同的形状。

对于 `rect()` 函数，指定了起始位置的 x 和 y，即矩形的左下角坐标。
然后指定了形状的宽和高。

参数 `stroke` 告知 ReportLab 是否绘制边框，而参数 `fill` 告知 ReportLab 是否向形状内填充颜色。
所有的形状都支持这两个参数。

根据 `ellipse()` 的文档，它接受开始（x,y）和结束（x,y）坐标，坐标表示椭圆形状的最小外接矩形。

`wedge()` 形状与之相似，再一次指定了一系列的点，代表楔形的不可见的最小外接矩形。
你需要做的只是假设有一个圆在该矩形内，并描述这个矩形的大小。
第 5 个参数是 `startAng`，它是楔形的起始角度。
第 6 个参数是 `extent`，它告诉楔形可以展开到多远。

最后，创建了一个 `circle()`，它接受一个（x,y）坐标作为圆心，然后是它的半径。
此处跳过了 `stroke` 和 `fill` 参数的设置。

当你执行这段代码，最终会得到一个像这样的 PDF：

![[Figure-3.png]]

*使用 ReportLab 创建多边形*

这看上去相当好。
你可以自行修改这些值，并看看能否找出如何以不同方式变更这些形状。

虽然画的形状很有趣，但它们在任何专业公司的文档上都不太行。
如果想要往 PDF 报告上加一个公司 logo 呢？
可以使用 ReportLab，通过向文档加入图片的方式实现它。
来看看如何实现的，创建一个名为 `image_on_canvas.py` 的文件然后加上这段代码：

```Python
# image_on_canvas.py

from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas

def add_image(image_path):
    my_canvas = canvas.Canvas("canvas_image.pdf", pagesize=letter)
    my_canvas.drawImage(image_path, 30, 600, width=100, height=100)
    my_canvas.save()

if __name__ == '__main__':
    image_path = 'snakehead.jpg'
    add_image(image_path)
```

为了在画布上绘制图像，可以使用 `drawImage()` 方法。
它接受图像的路径作为参数，x 和 y 作为其起始位置，以及希望的宽和高。
这个方法不会维持传入图像的宽高比。
如果没有正确地设置宽和高，图像将被拉伸。

当你执行这段代码，最终会得到像这样的 PDF：

![[Figure-4.png]]

*往 PDF 中添加一张图片*

`Canvas()` 类相当强大。
不过仍需要保持追踪在页面上的位置，并且告诉画布创建新页面的时机。
如果不把代码变得相当复杂，实现起来就很困难。
幸运的是，有一个更好的方式，你会在下个章节中了解它。

## 使用 PLATYPUS 创建多页文档

ReportLab 有一个简洁的概念，叫做 **PLATYPUS**，它代表了 “脚本式的页面布局和排版”。
它是 ReportLab 提供的一个高阶布局库，用于轻松地编程式创建复杂的布局，却只需要极少的代码。
PLATYPUS 基本上会处理好分页、布局和样式。

PLATYPUS 中有一些类可供使用。
这些类被称为 **Flowable**。
Flowable 可以被添加到文档中，并且自动拆分到多个页面中去。
以下 Flowable 将是最常用的：

- `Paragraph()` - 用于添加文字
- `getSampleStyleSheet()` - 用于向 Paragraph 添加样式
- `Table()` - 用于表格数据
- `SimpleDocTemplate()` - 文档模板，用于保存其他 Flowable

要看看如何使用这些类，创建一个名为 `hello_platypus.py` 的文件并添加如下代码：

```Python
# hello_platypus.py

from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Paragraph
from reportlab.lib.styles import getSampleStyleSheet

def hello():
    doc = SimpleDocTemplate(
            "hello_platypus.pdf",
            pagesize=letter,
            rightMargin=72, leftMargin=72,
            topMargin=72, bottomMargin=18,
            )
    styles = getSampleStyleSheet()

    flowables = []

    text = "Hello, I'm a Paragraph"
    para = Paragraph(text, style=styles["Normal"])
    flowables.append(para)

    doc.build(flowables)

if __name__ == '__main__':
    hello()
```

这段代码从 `reportlab.platypus` 新导入了两个类：`simpleDocTemplate()` 和 `Paragraph()`。
还从 `reportlab.lib.styles` 导入了 `getSampleStyleSheet()`。
然后在 `hello()` 函数中，创建了一个文档模板对象。
这是传递希望创建 PDF 文件的路径的地方。
到这还类似于 `Canvas()`，只不过实现在 PLATYPUS 中。
这里还设置了 `pagesize` 并指定了它的边距。
你其实没必要去设置边距，但应当知道你可以做到，这是这段代码出现在这的原因。

然后是获取简单的样式表。
这个样式变量是一个 `reportlab.lib.styles.StyleSheet` 类型的对象。
可以经由样式表访问不同的样式。
该例中使用的是 `Normal` 样式表。

下一段代码创建了一个使用 `Paragraph()` 的单一 Flowable。`Paragraph()` 可以接受多个不同参数。
该例中，它传递了一些文本以及希望应用到文本上的样式。
如果你查看样式表的代码，会发现可以往文本中应用各种 “标题”（“Heading”）样式，以及 “代码”（“Code”）样式、“斜体”（“Italic”）样式等等。

为了让文档正确生成，维护一个 Flowable 构成的 Python 列表。
该例子里，列表中只有一个元素：一个 `Paragraph()`。
可以调用 `build()` 并传递 Flowable 的列表来创建 PDF，作为 `save()` 的替代品。

现在 PDF 已经生成了。
它将看上去像这样：

![[Figure-5.png]]

*在 ReportLab 中创建一个 Paragraph*

简洁！
不过它只有一个元素，看上去仍然有点单调。

要了解 PLATYPUS 框架多有用，将创建一个包含数十个 Flowable 的文档。
继续创建一个名为 `platypus_multipage.py` 的文件并加入这段代码：

```Python
# platypus_multipage.py

from reportlab.lib.pagesizes import letter
from reportlab.lib.styles import getSampleStyleSheet
from reportlab.lib.units import inch
from report.pltypus import SimpleDocTemplate, Paragraph, Spacer

def create_document():
    doc = SimpleDocTemplate(
            "platypus_multipage.pdf",
            pagesize=letter,
            )
    styles = getSampleStyleSheet()
    flowables = []
    spacer = Spacer(1, 0.25*inch)

    # Create a lot of content to make a multipage PDF
    for i in range(50):
        text = 'Paragraph #{}'.format(i)
        para = Paragraph(text, styles["Normal"])
        flowables.append(para)
        flowables.append(spacer)

    doc.build(flowables)

if __name__ == '__main__':
    create_document()
```

该例子创建了 50 个 `Paragraph()` 对象。
还创建了一个 `Spacer()`，用于在 Flowable 之间添加空格。
当创建一个 `Paragraph()` 时，把它以及一个 `Spacer()` 添加到了 Flowable 的列表中。
最后，列表中就有了 100 个 Flowable。

当你执行这段代码，会生成一个 3 页的文档。
第一页会以如下截图这样开始：

![[Figure-6.png]]

*创建一个多页的 PDF*

那并不太难！
现在让我们考虑如何向 PDF 里添加一个表格。

## 创建表格

在 ReportLab 中最复杂的 Flowable 之一是 `Table()`。
它允许在行列中展示表格数据。
表格允许在每个单元格里放入其他类型的 Flowable。
这使之可以创建复杂的文档。

首先，创建一个名为 `simple_table.py` 的文件并添加这些代码：

```Python
# simple_table.py

from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Table

def simple_table():
    doc = SimpleDocTemplate("simple_table.pdf", pagesize=letter)
    flowables = []

    data = [
            ['col_{}'.format(x) for x in range(1, 6)],
            [str(x) for x in range(1, 6)],
            ['a', 'b', 'c', 'd', 'e'],
            ]

    tbl = Table(data)
    flowables.append(tbl)

    doc.build(flowables)

if __name__ == '__main__':
    simple_table()
```

这次导入了 `Table()` 代替 `Paragraph()`。
其他部分的导入看上去差不多。

要往表格里添加数据，需要有一个 Python 列表构成的列表。
列表中的每一项必须是字符串或 Flowable。
在该例子中，三行字符串被创建出来了。
第一行是列首行，它标注了接下来几行是什么。

接着创建了 `Table()` 并将列表的列表作为 `data` 传递过去。
最后 `build()` 文档，正如之前所做那样。

你的 PDF 现在应该有一个表格了，它看上去像这样：

![[Figure-7.png]]

*在 ReportLab 中创建一个简单表格*

`Table()` 默认不显示边框和单元格边框。
这个 `Table()` 完全没有应用任何样式。

表格当然也可以应用样式，需要通过使用 `TableStyle()` 实现。
表格样式就像是应用在 `Paragraph()` 上的样式表一样。
观察它们是如何工作的，创建一个名为 `simple_table_with_style.py` 的新文件并添加如下代码：

```Python
# simple_table_with_style.py

from reportlab.lib import colors
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle

def simple_table_with_style():
    doc = SimpleDocTemplate(
            "simple_table_with_style.pdf",
            pagesize=letter,
            )
    flowables = []

    data = [
            ['col_{}'.format(x) for x in range(1, 6)],
            [str(x) for x in range(1, 6)],
            ['a', 'b', 'c', 'd', 'e'],
            ]

    tblstyle = TableStyles([
        ('BACKGROUND', (0, 0), (-1, 0), colors.red),
        ('TEXTCOLOR', (0, 1), (-1, 1), colors.blue),
        ])

    tbl = Table(data)
    tbl.setStyle(tblstyle)
    flowables.append(tbl)

    doc.build(flowables)

if __name__ == '__main__':
    simple_table_with_style()
```

这次加了一个 `TableStyle()`，它是 Python 列表构成的元组。
元组包含了希望应用的样式类型，以及应用到哪些单元格上。

第一个元组标明，希望在 column 0, row 0 处开始应用一个红色背景。
这个样式会一直应用到最后一列，最后一列在这里被指定为 -1。

第二个元组将蓝色应用在文本上，从 column 0, row 1 开始到 row 1 的所有列。
要把样式添加到表格上，调用 `setStyle()` 并且把创建的 `TableStyle()` 实例传递过去。

注意，需要从 `reportlab.lib` 导入 `colors` 来得到要应用的两种颜色。

当你执行这段代码，最终会得到如下表格：

![[Figure-8.png]]

*ReportLab 表格及样式*

如果想向表格和单元格加边框，要添加一个元组并告知哪个单元格应用它，该元组使用 “GRID” 作为样式命令。

## 总结

ReportLab 是可用于由 Python 创建 PDf 的最全面的包。
在本文中，你学习了如下内容：

- 安装 ReportLab
- 使用画布（Canvas）创建一个简单的 PDF
- 使用画布（Canvas）创建绘图（Drawing）并添加图像（Image）
- 使用 PLATYPUS 创建多页文档
- 创建表格

本文仅触及了操作 ReportLab 可实现效果的皮毛。
使用 ReportLab 时，可以搭配不同类型的字体，加上页面与页脚，插入条形码，等等等等。
你可以在我的书《[**ReportLab: PDF Processing with Python**](https://leanpub.com/reportlab)》中了解这些内容以及其他 Python PDF 包。

## 相关文章

你也可以尝试本文之外的更多 ReportLab 内容。
查阅其他一些教程以了解更多：

- [ReportLab: Adding a Chart to a PDF with Python](https://www.blog.pythonlibrary.org/2019/04/08/reportlab-adding-a-chart-to-a-pdf-with-python/)
- [Creating Interactive PDF Forms in ReportLab with Python](https://www.blog.pythonlibrary.org/2018/05/29/creating-interactive-pdf-forms-in-reportlab-with-python/)
- [Adding SVGs to PDFs with Python and ReportLab](https://www.blog.pythonlibrary.org/2018/04/12/adding-svg-files-in-reportlab/)
- [ReportLab 101: The textobject](https://www.blog.pythonlibrary.org/2018/02/06/reportlab-101-the-textobject/)