  前两天领导说不用web端的打印插件（C-Lodop），想用C#来实现，让我研究一下调用打印机的方法，就有了这篇总结

  .NET Core或.NET 5的话要引用一下NuGet包 System.Drawing.Common

  获取打印机列表
  ```csharp
  PrinterSettings.InstalledPrinters.Cast<string>();
  ```

  定义打印接口
  ```csharp
public interface IPrinter
{
    /// <summary>
    /// 设置页面大小
    /// </summary>
    /// <param name="paperWidth"></param>
    /// <param name="paperHight"></param>
    void SetPageSize(double paperWidth, int? paperHight);

    /// <summary>
    /// 打印文字
    /// </summary>
    /// <param name="content"></param>
    /// <param name="fontSize"></param>
    /// <param name="stringAlignment"></param>
    /// <param name="width"></param>
    /// <param name="offset"></param>
    void PrintText(
        string content,
        FontSize fontSize = FontSize.Normal,
        StringAlignment stringAlignment = StringAlignment.Near,
        float width = 1,
        float offset = 0
        );

    /// <summary>
    /// 打印图片
    /// </summary>
    /// <param name="image"></param>
    /// <param name="stringAlignment"></param>
    void PrintImage(Image image, StringAlignment stringAlignment = StringAlignment.Near);

    /// <summary>
    /// 打印一行
    /// </summary>
    void PrintSolidLine();

    /// <summary>
    /// 打印一行虚线
    /// </summary>
    /// <param name="fontSize"></param>
    void PrintDottedLine();

    /// <summary>
    /// 新起一行
    /// </summary>
    void NewLine();

    /// <summary>
    /// 开始打印
    /// </summary>
    void Print();
}
  ```

打印接口的实现
```csharp
public class Printer : IPrinter
    {

        #region fields
        private PrintDocument _printDoc = new PrintDocument();

        /// <summary>
        /// 打印对象打印宽度(根据英寸换算而来,paperWidth * 3.937)
        /// </summary>
        private int _paperWidth;
        private int _paperHight;
        private const float _charProportion = 0.7352f;
        private const float _lineHeightProportion = 1.6f;
        private const string _fontName = "SimHei";
        private int _printIndex = 0;
        /// <summary>
        /// 打印模式：1、自动分页模式  2、连续不分页模式
        /// </summary>
        private int _mode;
        private IList<Action<Graphics>> _printActions = new List<Action<Graphics>>();

        /// <summary>
        /// 当前的打印高度，当调用换行或者图片打印时会增加此字段值
        /// </summary>
        private float _currentHeight = 0;

        public float NewLineOffset { get; set; } = (int)FontSize.Normal * _lineHeightProportion;

        #endregion

        #region ctor

        /// <summary>
        /// 初始化机打印对象
        /// </summary>
        /// <param name="printerName">打印机名称</param>
        /// <param name="paperWidth">打印纸宽度</param>
        /// <param name="paperHight">打印纸高度</param>
        internal Printer(string printerName)
        {
            _printDoc.PrinterSettings.PrinterName = printerName;
            _printDoc.PrintPage += PrintPageDetails;

            //默认为58mm
            SetPageSize(58, null);            

            _printDoc.PrintController = new StandardPrintController();
        }


        #endregion

        #region eventHandler
        void PrintPageDetails(object sender, PrintPageEventArgs e)
        {
            while (_printIndex < _printActions.Count)
            {
                var originHeight = _currentHeight;
                _printActions[_printIndex](e.Graphics);
                _printIndex++;
                if (_currentHeight > e.PageBounds.Height - 20)
                {
                    if (_mode == 1)
                    {
                        _currentHeight = 0;
                        // HasMorePages 设置为true后，会继续触发PrintPage，因此_printIndex需要在外部进行定义
                        e.HasMorePages = true;
                        return;
                    }
                }
            }

            e.HasMorePages = false;
        }
        #endregion

        #region IPrinterImplement

        /// <summary>
        /// 设置页面大小
        /// </summary>
        /// <param name="paperWidth"></param>
        /// <param name="paperHight"></param>
        public void SetPageSize(double paperWidth, int? paperHight)
        {
            switch (paperWidth)
            {
                case 80:
                    //80打印纸扣去两边内距实际可打的宽度为72.1
                    paperWidth = 72.1;
                    break;
                case 76:
                    //76打印纸扣去两边内距实际可打的宽度为63.5
                    paperWidth = 63.5;
                    break;
                case 58:
                    //58打印纸扣去两边内距实际可打的宽度为48
                    paperWidth = 48;
                    break;
                default:
                    paperWidth = paperWidth - 10;
                    break;
            }

            _paperWidth = Convert.ToInt32(Math.Ceiling(paperWidth * 3.937));
            _mode = 1;
            if (!paperHight.HasValue)
            {
                paperHight = 297;
                _mode = 2; // 设置为连续不分页模式
            }

            _paperHight = Convert.ToInt32(Math.Ceiling(paperHight.Value * 3.937));
            _printDoc.DefaultPageSettings.PaperSize = new PaperSize("", _paperWidth, _paperHight);
        }

        /// <summary>
        /// 开始打印
        /// </summary>
        public void Print()
        {
            _printIndex = 0;
            _currentHeight = 0;
            _printDoc.EndPrint += (_, _) => Console.WriteLine("打印完成！");
            if (_mode == 2)
            {
                // 通过下面的方式计算出实际页面高度
                using (Bitmap img = new Bitmap(_paperWidth, _paperHight))
                {
                    var g = Graphics.FromImage(img);
                    foreach (var item in _printActions)
                    {
                        item(g);
                    }
                    _paperHight = Convert.ToInt32(Math.Ceiling(_currentHeight)) + 5;
                    _printDoc.DefaultPageSettings.PaperSize = new PaperSize("", _paperWidth, _paperHight);
                    _printIndex = 0;
                    _currentHeight = 0;
                }
            }
            _printDoc.Print();
            _printDoc.Dispose();
            _printDoc = new PrintDocument();
            _printActions.Clear();
        }

        /// <summary>
        /// 新起一行
        /// </summary>
        public void NewLine()
        {
            _printActions.Add(g =>
            {
                _currentHeight += NewLineOffset;
                NewLineOffset = (int)FontSize.Normal * _lineHeightProportion;
            });
        }

        /// <summary>
        /// 打印文字
        /// </summary>
        /// <param name="content"></param>
        /// <param name="fontSize"></param>
        /// <param name="alignment"></param>
        /// <param name="width"></param>
        /// <param name="offset"></param>
        public void PrintText(string content, FontSize fontSize = FontSize.Normal, StringAlignment alignment = StringAlignment.Near, float width = 1, float offset = 0)
        {
            _printActions.Add(g =>
            {
                float contentWidth = width == 1 ? _paperWidth * (1 - offset) : width * _paperWidth;
                string newContent = ContentWarp(content, fontSize, contentWidth, out var rowNum);
                var font = new Font(_fontName, (int)fontSize, FontStyle.Regular);
                var point = new PointF(offset * _paperWidth, _currentHeight);
                var size = new SizeF(contentWidth, (int)fontSize * _lineHeightProportion * rowNum);
                var layoutRectangle = new RectangleF(point, size);
                var format = new StringFormat
                {
                    Alignment = alignment,
                    FormatFlags = StringFormatFlags.NoWrap
                };
                g.DrawString(newContent, font, Brushes.Black, layoutRectangle, format);
                float thisHeightOffset = rowNum * (int)fontSize * _lineHeightProportion;
                if (thisHeightOffset > NewLineOffset) NewLineOffset = thisHeightOffset;
            });
        }

        /// <summary>
        /// 打印图片
        /// </summary>
        /// <param name="image"></param>
        /// <param name="stringAlignment"></param>
        public void PrintImage(Image image, StringAlignment stringAlignment = StringAlignment.Near)
        {
            _printActions.Add(g =>
            {
                int x = 0;
                switch (stringAlignment)
                {
                    case StringAlignment.Near:
                        break;
                    case StringAlignment.Center:
                        x = (_paperWidth - image.Width) / 2;
                        break;
                    case StringAlignment.Far:
                        x = _paperWidth - image.Width;
                        break;
                    default:
                        break;
                }
                var point = new Point(x, Convert.ToInt32(_currentHeight));
                var size = new Size(image.Width, image.Height);
                var rectangle = new Rectangle(point, size);
                g.DrawImage(image, rectangle);
                NewLineOffset = image.Height;
            });
        }

        /// <summary>
        /// 打印一行实线
        /// </summary>
        public void PrintSolidLine()
        {
            _printActions.Add(g =>
            {
                var pen = new Pen(new SolidBrush(Color.Black));
                pen.Width = 1f;
                g.DrawLine(pen, new Point(0, Convert.ToInt32(Math.Ceiling(_currentHeight))), new Point(_paperWidth, Convert.ToInt32(Math.Ceiling(_currentHeight))));
                _currentHeight += 3;
            });
        }

        /// <summary>
        /// 打印一行虚线
        /// </summary>
        public void PrintDottedLine()
        {
            var fontSize = FontSize.Normal;
            int charNum = (int)(_paperWidth / ((int)fontSize * _charProportion));
            var builder = new StringBuilder();
            for (int i = 0; i < charNum; i++)
            {
                builder.Append('-');
            }
            PrintText(builder.ToString(), fontSize, StringAlignment.Center);
        }
        #endregion

        #region methods
        /// <summary>
        /// 对内容进行分行，并返回行数
        /// </summary>
        /// <param name="content">内容</param>
        /// <param name="fontSize">文字大小</param>
        /// <param name="width">内容区宽度</param>
        /// <returns>行数</returns>
        private static string ContentWarp(string content, FontSize fontSize, float width, out int row)
        {
            content = content.Replace(Environment.NewLine, string.Empty);

            //0.7282 字符比例
            var builder = new StringBuilder();
            float nowWidth = 0;
            row = 1;
            foreach (char item in content)
            {
                int code = Convert.ToInt32(item);
                float charWidth = code < 128 ? _charProportion * (int)fontSize : _charProportion * (int)fontSize * 2;
                nowWidth += charWidth;
                if (nowWidth > width)
                {
                    builder.Append(Environment.NewLine);
                    nowWidth = charWidth;
                    row++;
                }
                builder.Append(item);
            }
            return builder.ToString();
        }

        #endregion
    }
```

创建打印机的工厂类
```csharp
public static class PrinterFactory
{
    public static IEnumerable<string> GetAllPrints()
    {
        return PrinterSettings.InstalledPrinters.Cast<string>();
    }

    public static Printer GetPrinter(string printerName)
    {
        if (string.IsNullOrEmpty(printerName)) throw new ArgumentException(nameof(printerName));
        return new Printer(printerName);
    }
}
```

使用
```csharp
static void Main(string[] args)
{
    // Microsoft XPS Document Writer 是测试时使用的，实际使用中，要替换成真正的打印机
    var printer = PrinterFactory.GetPrinter("Microsoft XPS Document Writer");
    printer.SetPageSize(80, null);
    printer.NewLine();
    var img = GetLogo();
    printer.PrintImage(img, StringAlignment.Center);
    printer.NewLine();
    printer.NewLine();
    printer.PrintText("永辉超市", FontSize.Large, alignment: StringAlignment.Center);
    printer.NewLine();
    printer.NewLine();
    printer.PrintText("单号：XD000269");
    printer.PrintText("流水号：000269", offset: 0.5f);
    printer.NewLine();
    printer.PrintText("收银员：***");
    printer.PrintText("日期：" + DateTime.Now.ToString("yyyy/MM/dd"), offset: 0.5f);
    printer.NewLine();
    printer.PrintText("VIP客户卡号：001");
    printer.NewLine();
    printer.PrintSolidLine();
    printer.NewLine();
    printer.PrintText("名称");
    printer.PrintText("单价", offset: 0.35f);
    printer.PrintText("数量", offset: 0.65f);
    printer.PrintText("金额", alignment: StringAlignment.Far);
    printer.NewLine();
    printer.PrintText("芹菜", width: 0.35f);
    printer.PrintText("2.9", width: 0.2f, offset: 0.35f);
    printer.PrintText("1", width: 0.2f, offset: 0.65F);
    printer.PrintText("2.9", alignment: StringAlignment.Far);
    printer.NewLine();
    printer.PrintDottedLine();
    printer.NewLine();
    printer.PrintText("合计");
    printer.PrintText("1", offset: 0.65f);
    printer.PrintText("2.90", alignment: StringAlignment.Far);
    printer.NewLine();
    printer.PrintText("满0.00减0.00折扣");
    printer.PrintText("-0.00", alignment: StringAlignment.Far);
    printer.NewLine();
    printer.PrintText("优惠金额：2.90");
    printer.PrintText("实收金额：0", offset: 0.5f);
    printer.NewLine();
    printer.PrintText("收款金额：0.00");
    printer.PrintText("找零金额：-2.90", offset: 0.5f);
    printer.NewLine();
    printer.PrintDottedLine();
    printer.NewLine();
    printer.PrintText("会员卡：001");
    printer.NewLine();
    printer.PrintText("本次积分：");
    printer.PrintText("会员余额：43.87", offset: 0.5f);
    printer.NewLine();
    printer.PrintText("可用积分：");
    printer.NewLine();
    printer.PrintSolidLine();
    printer.NewLine();
    printer.PrintText("永辉超市", FontSize.Large, alignment: StringAlignment.Center);
    printer.NewLine();
    printer.PrintText("欢迎光临，谢谢惠顾！", FontSize.Large, alignment: StringAlignment.Center);
    printer.NewLine();

    printer.Print();
    GC.Collect();
    
    Console.ReadKey();
}

private static Image GetLogo()
{
    var path = Path.Combine(Directory.GetCurrentDirectory(), "logo.jpg");
    using (var fileStream = File.OpenRead(path))
    {
        fileStream.Seek(0, SeekOrigin.Begin);
        return Image.FromStream(fileStream);
    }
}
```

### 效果如下

![](https://github.com/JolyneStone/Blogs/blob/master/images/202108172050.png?raw=true)


demo已上传至 [github](https://github.com/JolyneStone/demo/tree/master/PrintDemo)