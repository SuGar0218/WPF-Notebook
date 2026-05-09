# TextBox

## 检测文本变化

**重要 API**

- [TextCompositionManager.PreviewTextInputStart 附加事件](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.input.textcompositionmanager.previewtextinputstart)
- [TextCompositionManager.PreviewTextInput 附加事件](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.input.textcompositionmanager.previewtextinput)
- [TextCompositionManager](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.input.textcompositionmanager)
- [TextComposition](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.input.textcomposition)

在 TextBox 中输入文本时，会引发其自身的 TextChanged 事件。输入中文时，不同输入法会导致不同结果。有些输入法不会把输入过程添加到文本框中（例如“搜狗输入法”），不会引发 TextBox.TextChanged 事件。有些输入法会把输入过程添加到文本框中（例如 Windows 系统自带的拼音输入法），这就会引发 TextBox.TextChanged 事件，但其实这时文本输入并没有完成，此时的 TextChanged 事件并非代表文本“真的”发生变化了。

我们的目的是，区分输入法输入过程造成的文本变化，从而处理“真正的”文本变化事件。

### 参考源代码

关于这个问题，WPF 内置控件 ComboBox 是如何解决的？对 ComboBox 设置以下属性：

- [IsReadOnly](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.controls.combobox.isreadonly) = false;
- [IsEditable](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.controls.combobox.iseditable) = true;
- [IsTextSearchEnabled](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.controls.itemscontrol.istextsearchenabled) = true;

ComboBox 支持输入文本，自动根据文本搜索选项。如果使用 Windows 自带的拼音输入法，会发现自动搜索行为发生在输入法完成输入后，并非像 TextChanged 事件一样只要有文字输入就触发。[源代码](https://github.com/dotnet/wpf/blob/7f005faa89e79b0b1fa1cb2c21283bab7916c092/src/Microsoft.DotNet.Wpf/src/PresentationFramework/System/Windows/Controls/ComboBox.cs#L642)中的做法是处理 TextBox.PreviewTextInput 事件：

``` C#
// When the IME composition we're waiting for completes, run the text search logic
private void OnEditableTextBoxPreviewTextInput(object sender, System.Windows.Input.TextCompositionEventArgs e)
{
    if (IsWaitingForTextComposition &&
        e.TextComposition.Source == EditableTextBoxSite &&
        e.TextComposition.Stage == System.Windows.Input.TextCompositionStage.Done)
    {
        IsWaitingForTextComposition = false;
        TextUpdated(EditableTextBoxSite.Text, true);

        // ComboBox.Text has just changed, but EditableTextBoxSite.Text hasn't.
        // As a courtesy to apps and controls that expect a TextBox.TextChanged
        // event after ComboTox.Text changes, raise such an event now.
        // (A notable example is TFS's WpfFieldControl)
        EditableTextBoxSite.RaiseCourtesyTextChangedEvent();
    }
}
```

WPF 所有 UIElement 的派生类都有 [TextInput](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.uielement.textinput) 和 [PreviewTextInput](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.uielement.previewtextinput) 事件，对于 TextBox，TextInput 事件已由内部处理，外部事件处理不会被执行，所以在此处理 TextBox.PreviewTextInput 事件。

源代码通过 TextComposition 类对象的 Stage 属性判断当前输入法是否完成输入，但 [TextComposition.Stage](https://github.com/dotnet/wpf/blob/7f005faa89e79b0b1fa1cb2c21283bab7916c092/src/Microsoft.DotNet.Wpf/src/PresentationCore/System/Windows/Input/TextComposition.cs#L361) 是 WPF 的内部属性，虽然我们无法使用，但可以感觉到判断输入法是否输入完成这件事与 [TextComposition](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.input.textcomposition) 和 TextInput 之类的事件有关。

### 结论

[UIElement.TextInput](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.uielement.textinput) 事件在文档中提到它是 [TextCompositionManager.TextInput 附加事件](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.input.textcompositionmanager.textinput) 的“别名”，顺着这个信息找过去就能找到这两个附加事件：

- [TextCompositionManager.PreviewTextInputStart](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.input.textcompositionmanager.previewtextinputstart) : 在 TextComposition 开始时发生
- [TextCompositionManager.PreviewTextInput](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.input.textcompositionmanager.previewtextinput) : 在 TextComposition 完成时发生

通过 Windows 自带输入法输入法输入时，开始输入正好对应 TextComposition 开始，结束输入正好对应 TextComposition 完成。只需要一个标记变量即可在 TextChanged 中区分文本有没有“真的”变化。

### 验证

``` xml
<TextBox
    x:Name="PART_TextTextBox"
    TextChanged="OnTextChanged"
    TextCompositionManager.PreviewTextInputStart="OnTextCompositionManagerPreviewTextInputStart"
    TextCompositionManager.PreviewTextInput="OnTextCompositionManagerPreviewTextInput" />
```

``` C#
private bool _isTextComposing;

private void OnTextChanged(object sender, TextChangedEventArgs e)
{
    if (_isTextComposing)
        return;

    TextBox textBox = (TextBox) sender;
    Debug.WriteLine($"文本发生变化：{textBox.Text}");
}

private void OnTextCompositionManagerPreviewTextInputStart(object sender, TextCompositionEventArgs e)
{
    _isTextComposing = true;
    Debug.WriteLine("TextComposition 开始");
}

private void OnTextCompositionManagerPreviewTextInput(object sender, TextCompositionEventArgs e)
{
    _isTextComposing = false;
    Debug.WriteLine("TextComposition 完成");
}
```

### 封装

封装一个 TextBoxInputMonitor 类，通过[附加依赖属性](https://learn.microsoft.com/zh-cn/dotnet/desktop/wpf/properties/attached-properties-overview)和[附加事件](https://learn.microsoft.com/zh-cn/dotnet/desktop/wpf/events/attached-events-overview)的方式提供：

- TextReallyChanged 事件：文本“真的”发生变化，排除输入法影响
- InputMethodEditingStart 事件：输入法编辑启动
- InputMethodEditingComplete 事件：输入法编辑完成
- IsInputMethodEditing 属性：是否正在进行输入法编辑

这个类的设计是，通过 TextBoxInputMonitor.IsEnabled 附加属性决定是否为指定 TextBox 启用该功能，如果启用，将订阅相关事件，如果禁用，将取消订阅相关事件。核心功能和启用禁用分别写在了两个文件中。

TextBoxInputMonitor.cs
``` C#
public partial class TextBoxInputMonitor : IDisposable
{
    public TextBoxInputMonitor(TextBoxBase textBox)
    {
        TargetTextBox = textBox;
        TargetTextBox.TextChanged += OnTextBoxTextChanged;
        TextCompositionManager.AddPreviewTextInputStartHandler(TargetTextBox, OnTextCompositionStart);
        TextCompositionManager.AddPreviewTextInputHandler(TargetTextBox, OnTextCompositionComplete);
    }

    public void Dispose()
    {
        TargetTextBox.TextChanged -= OnTextBoxTextChanged;
        TextCompositionManager.AddPreviewTextInputStartHandler(TargetTextBox, OnTextCompositionStart);
        TextCompositionManager.AddPreviewTextInputHandler(TargetTextBox, OnTextCompositionComplete);
    }

    #region 附加事件 TextReallyChanged

    public static void AddTextReallyChangedHandler(DependencyObject dependencyObject, RoutedEventHandler handler) => (dependencyObject as UIElement)?.AddHandler(TextReallyChangedEvent, handler);
    public static void RemoveTextReallyChangedHandler(DependencyObject dependencyObject, RoutedEventHandler handler) => (dependencyObject as UIElement)?.RemoveHandler(TextReallyChangedEvent, handler);

    public static readonly RoutedEvent TextReallyChangedEvent = EventManager.RegisterRoutedEvent(
        "TextReallyChanged",
        RoutingStrategy.Direct,
        typeof(RoutedEventHandler),
        typeof(TextBoxInputMonitor)
    );

    #endregion

    #region 附加事件 InputMethodEditingStart

    public static void AddInputMethodEditingStartHandler(DependencyObject dependencyObject, RoutedEventHandler handler) => (dependencyObject as UIElement)?.AddHandler(InputMethodEditingStartEvent, handler);
    public static void RemoveInputMethodEditingStartHandler(DependencyObject dependencyObject, RoutedEventHandler handler) => (dependencyObject as UIElement)?.RemoveHandler(InputMethodEditingStartEvent, handler);

    public static readonly RoutedEvent InputMethodEditingStartEvent = EventManager.RegisterRoutedEvent(
        "InputMethodEditingStart",
        RoutingStrategy.Bubble,
        typeof(RoutedEventHandler),
        typeof(TextBoxInputMonitor)
    );

    #endregion

    #region 附加事件 InputMethodEditingComplete

    public static void AddInputMethodEditingCompleteHandler(DependencyObject dependencyObject, RoutedEventHandler handler) => (dependencyObject as UIElement)?.AddHandler(InputMethodEditingCompleteEvent, handler);
    public static void RemoveInputMethodEditingCompleteHandler(DependencyObject dependencyObject, RoutedEventHandler handler) => (dependencyObject as UIElement)?.RemoveHandler(InputMethodEditingCompleteEvent, handler);

    public static readonly RoutedEvent InputMethodEditingCompleteEvent = EventManager.RegisterRoutedEvent(
        "InputMethodEditingComplete",
        RoutingStrategy.Bubble,
        typeof(RoutedEventHandler),
        typeof(TextBoxInputMonitor)
    );

    #endregion

    #region 附加属性 IsInputMethod

    public static bool GetIsInputMethodEditing(TextBoxBase target) => (bool) target.GetValue(IsInputMethodEditingProperty);
    private static void SetIsInputMethodEditing(TextBoxBase target, bool value) => target.SetValue(IsInputMethodEditingProperty, value);

    public static readonly DependencyProperty IsInputMethodEditingProperty = DependencyProperty.RegisterAttached(
        "IsInputMethodEditing",
        typeof(bool),
        typeof(TextBoxBase),
        new PropertyMetadata(default(bool))
    );

    #endregion

    public event TextChangedEventHandler TextReallyChanged;

    public TextBoxBase TargetTextBox { get; }

    private bool _isTextComposing;

    private void OnTextBoxTextChanged(object sender, TextChangedEventArgs e)
    {
        if (_isTextComposing)
            return;

        TextReallyChanged?.Invoke(this, e);
        TargetTextBox.RaiseEvent(new RoutedEventArgs(TextReallyChangedEvent));
    }

    private void OnTextCompositionStart(object sender, TextCompositionEventArgs e)
    {
        _isTextComposing = true;
        SetIsInputMethodEditing(TargetTextBox, true);
        TargetTextBox.RaiseEvent(new RoutedEventArgs(InputMethodEditingStartEvent));
    }

    private void OnTextCompositionComplete(object sender, TextCompositionEventArgs e)
    {
        _isTextComposing = false;
        SetIsInputMethodEditing(TargetTextBox, false);
        TargetTextBox.RaiseEvent(new RoutedEventArgs(InputMethodEditingCompleteEvent));
    }
}
```

TextBoxInputMonitorEnable.cs
``` C#
public partial class TextBoxInputMonitor
{
    public static bool GetIsEnabled(TextBoxBase target) => (bool) target.GetValue(IsEnabledProperty);
    public static void SetIsEnabled(TextBoxBase target, bool value) => target.SetValue(IsEnabledProperty, value);

    public static readonly DependencyProperty IsEnabledProperty = DependencyProperty.RegisterAttached(
        "IsEnabled",
        typeof(bool),
        typeof(TextBoxBase),
        new PropertyMetadata(default(bool), OnIsEnabledChanged)
    );

    private static void OnIsEnabledChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        TextBoxBase textBox = (TextBoxBase) d;
        if ((bool) e.NewValue)
        {
            _monitors.Add(textBox, new TextBoxInputMonitor(textBox));
        }
        else if (_monitors.TryGetValue(textBox, out var monitor))
        {
            monitor.Dispose();
            _monitors.Remove(textBox);
        }
    }

    private static readonly Dictionary<TextBoxBase, TextBoxInputMonitor> _monitors = new Dictionary<TextBoxBase, TextBoxInputMonitor>();
}
```

### 测试

``` xml
<StackPanel>
    <TextBox
        x:Name="testTextBox"
        helpers:TextBoxInputMonitor.InputMethodEditingComplete="OnInputMethodEditingComplete"
        helpers:TextBoxInputMonitor.InputMethodEditingStart="OnInputMethodEditingStart"
        helpers:TextBoxInputMonitor.IsEnabled="True"
        helpers:TextBoxInputMonitor.TextReallyChanged="OnTextReallyChanged" />
    <Button Click="OnChangeTextButtonClick" Content="把文本改为“Lorem ipsum”" />
</StackPanel>
```

``` C#
private void OnTextReallyChanged(object sender, RoutedEventArgs e)
{
    if (sender is TextBox textBox)
    {
        Debug.WriteLine($"TextReallyChanged: {textBox.Text}");
    }
}

private void OnInputMethodEditingStart(object sender, RoutedEventArgs e)
{
    Debug.WriteLine("InputMethodEditingStart");
}

private void OnInputMethodEditingComplete(object sender, RoutedEventArgs e)
{
    Debug.WriteLine("InputMethodEditingComplete");
}

private void OnChangeTextButtonClick(object sender, RoutedEventArgs e)
{
    testTextBox.Text = "Lorem ipsum";
}
```

在 Visual Studio 调试启动，使用 Windows 自带拼音输入法输入“中文”，输出窗格显示：

``` txt
InputMethodEditingStart
InputMethodEditingComplete
TextReallyChanged: 中文
```

点击按钮改变文本，输出窗格显示：

``` txt
TextReallyChanged: Lorem ipsum
```
