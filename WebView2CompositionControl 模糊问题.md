# WebView2CompositionControl 模糊问题

[GitHub 上的相关问题](https://github.com/MicrosoftEdge/WebView2Feedback/issues/5205)

[微软文档](https://learn.microsoft.com/zh-cn/dotnet/api/microsoft.web.webview2.wpf.webview2compositioncontrol)中提到 WebView2CompositionControl 用一个 Image 控件显示从浏览器捕获的画面。微软给出的解决模糊的方案是在根元素（例如 Window）上设置 `UseLayoutRounding="True"`，但这会导致整个窗口其他控件显示轻微错位。

若要防止 WebView2CompositionControl 显示模糊，本质上是要让它与设备像素对齐。我们将分别解决两个问题：

- 放置 WebView2CompositionControl 时对齐设备像素

- 微调 WebView2CompositionControl 内部的 Image 控件使其对齐设备像素

## 对齐设备像素放置

继承自 Decorator 编写自定义控件 LayoutRoundingDecorator 以对子元素进行对齐设备像素的排列。

``` C#
public class LayoutRoundingDecorator : Decorator
{
    protected override Size MeasureOverride(Size constraint)
    {
        if (Child is null)
            return new Size(0, 0);

        Child.Measure(constraint);
        return Child.DesiredSize;
    }

    protected override Size ArrangeOverride(Size finalSize)
    {
        if (Child is null)
            return finalSize;

        Child.Measure(finalSize);
        Rect originalRect = new Rect(0, 0, finalSize.Width, finalSize.Height);
        Rect snappedRect = Fix(originalRect);
        Child.Arrange(snappedRect);
        return snappedRect.Size;
    }

    private Rect Fix(Rect rect)
    {
        if (rect.IsEmpty ||
            double.IsNaN(rect.X) || double.IsNaN(rect.Y) ||
            double.IsNaN(rect.Width) || double.IsNaN(rect.Height) ||
            double.IsInfinity(rect.X) || double.IsInfinity(rect.Y) ||
            double.IsInfinity(rect.Width) || double.IsInfinity(rect.Height))
            return rect;

        PresentationSource source = PresentationSource.FromVisual(this);
        Visual? root = source?.RootVisual;
        if (source is null || source.CompositionTarget is null || root is null)
            return rect;

        GeneralTransform localToRoot = TransformToAncestor(root);
        GeneralTransform rootToLocal = root.TransformToDescendant(this);
        if (localToRoot is null || rootToLocal is null)
            return rect;

        Matrix toDevice = source.CompositionTarget.TransformToDevice;
        Matrix fromDevice = source.CompositionTarget.TransformFromDevice;
        Point topLeftLocal = rect.TopLeft;
        Point bottomRightLocal = rect.BottomRight;
        Point topLeftRoot = localToRoot.Transform(topLeftLocal);
        Point bottomRightRoot = localToRoot.Transform(bottomRightLocal);
        Point topLeftDevice = toDevice.Transform(topLeftRoot);
        Point bottomRightDevice = toDevice.Transform(bottomRightRoot);
        double snappedLeft = Math.Round(topLeftDevice.X, MidpointRounding.AwayFromZero);
        double snappedTop = Math.Round(topLeftDevice.Y, MidpointRounding.AwayFromZero);
        double snappedRight = Math.Round(bottomRightDevice.X, MidpointRounding.AwayFromZero);
        double snappedBottom = Math.Round(bottomRightDevice.Y, MidpointRounding.AwayFromZero);
        Point snappedTopLeftDevice = new Point(snappedLeft, snappedTop);
        Point snappedBottomRightDevice = new Point(snappedRight, snappedBottom);
        Point snappedTopLeftRoot = fromDevice.Transform(snappedTopLeftDevice);
        Point snappedBottomRightRoot = fromDevice.Transform(snappedBottomRightDevice);
        Point snappedTopLeftLocal = rootToLocal.Transform(snappedTopLeftRoot);
        Point snappedBottomRightLocal = rootToLocal.Transform(snappedBottomRightRoot);
        double snappedWidth = Math.Max(0, snappedBottomRightLocal.X - snappedTopLeftLocal.X);
        double snappedHeight = Math.Max(0, snappedBottomRightLocal.Y - snappedTopLeftLocal.Y);
        return new Rect(snappedTopLeftLocal, new Size(snappedWidth, snappedHeight));
    }
}
```

## 微调内部 Image 控件

微调内部 Image 控件的大小，使其尽量与捕获的图像大小匹配。内部 Image 的 Stretch 属性设为 Uniform，至于为什么我也说不清，这是试了几个值之后感觉效果比较好的。

在 Visual Studio 跳转到 WebView2CompositionControl 代码里，看到内部是这样指定捕获区域大小的：

``` C#
_d3dImage.UpdateSize((int)(base.ActualWidth * _dpiScale), (int)(base.ActualHeight * _dpiScale));
```

ActualWidth 和 ActualHeight 都是 WebView2CompositionControl 上的大小而不是 Image 上的，所以只要调整内部的 Image 就行了而不会对捕获尺寸的计算造成影响。

由于对逻辑像素经过缩放比例换算得到的设备像素数直接强制转换到 int，这会导致 99.999 这样的像素被截断为 99 而不是当作 100 处理。因此需要调整内部 Image 控件的大小以贴合这一特性。程序运行期间查看实时可视化树，会发现内部 Image 控件的 RenderTransform 是一个比例为当前显示缩放倒数的 ScaleTransform，因此在调整 Image 尺寸时没有除以当前显示缩放，RenderTransform 自然会把它缩放到预期大小。

``` C#
internal class WebView2CompositionControlEx : WebView2CompositionControl
{
    public Stretch InnerImageStretch
    {
        get => (Stretch)GetValue(InnerImageStretchProperty);
        set => SetValue(InnerImageStretchProperty, value);
    }

    public static readonly DependencyProperty InnerImageStretchProperty = DependencyProperty.Register(
        nameof(InnerImageStretch),
        typeof(Stretch),
        typeof(WebView2CompositionControlEx),
        new FrameworkPropertyMetadata(
            default(Stretch),
            (d, e) => ((WebView2CompositionControlEx)d).OnStretchChanged(e))
    );

    private void OnStretchChanged(DependencyPropertyChangedEventArgs e)
    {
        if (PART_image is not null)
        {
            PART_image.Stretch = (Stretch)e.NewValue;
        }
    }

    private Image? PART_image;

    public override void OnApplyTemplate()
    {
        base.OnApplyTemplate();
        PART_image = GetTemplateChild(nameof(PART_image)) as Image;
        if (PART_image is not null)
        {
            PART_image.Stretch = InnerImageStretch;
            SizeChanged += OnSizeChanged;
        }
    }

    private void OnSizeChanged(object sender, SizeChangedEventArgs e)
    {
        double dpiScale = VisualTreeHelper.GetDpi(this).DpiScaleX;
        PART_image!.Width = (int)(e.NewSize.Width * dpiScale) + 0.5;
        PART_image!.Height = (int)(e.NewSize.Height * dpiScale) + 0.5;
    }
}
```

## RenderOptions.BitmapScalingMode 附加属性

[BitmapScalingMode](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.media.bitmapscalingmode).NearestNeighbor 可以让未对齐设备像素的像素对齐到最近的设备像素。上述操作我们已经尽可能让 WebView2CompositionControl 内部的 Image 控件尽可能对齐设备像素放置并且大小匹配捕获图像的像素，现在只要设置 `RenderOptions.BitmapScalingMode="NearestNeighbor"` 就可以让几乎所有像素都能一一对应贴合到设备像素上，达到清晰锐利的显示效果。NearestNeighbor 并不总是清晰锐利的，只有像素恰好匹配才行，否则反而导致锐利的模糊和锯齿感。

最终，在 XAML 中结合上述内容。

``` xml
<Window
    x:Class="TryWorkaroundWebView2CompositionControlBlur.MainWindow"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:local="clr-namespace:TryWorkaroundWebView2CompositionControlBlur"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:webview2="clr-namespace:Microsoft.Web.WebView2.Wpf;assembly=Microsoft.Web.WebView2.Wpf"
    Title="MainWindow"
    Width="800"
    Height="450"
    mc:Ignorable="d">

    <Grid Margin="3.14159">
        <local:LayoutRoundingDecorator>
            <local:WebView2CompositionControlEx
                InnerImageStretch="Uniform"
                RenderOptions.BitmapScalingMode="NearestNeighbor"
                Source="https://www.microsoft.com" />
        </local:LayoutRoundingDecorator>
    </Grid>

</Window>
```

完整的代码：https://github.com/SuGar0218/WpfWebView2CompositionControlBlurIssueWorkaround
