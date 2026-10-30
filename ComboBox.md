# ComboBox

## Popup 鼠标交互

`ComboBox` 展开 Popup 后，其他控件暂时不再响应鼠标，只有 `ComboBox` 本体和展开的 Popup 仍可操作。点击 `ComboBox` 和 Popup 以外的区域，Popup 会自动收起。

> GitHub 上的源代码：[ComboBox.cs](https://github.com/dotnet/wpf/blob/main/src/Microsoft.DotNet.Wpf/src/PresentationFramework/System/Windows/Controls/ComboBox.cs)

### 重要 API

- [Mouse.Capture](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.input.mouse.capture#system-windows-input-mouse-capture(system-windows-iinputelement-system-windows-input-capturemode))

- [Mouse.LostMouseCapture](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.input.mouse.lostmousecapture)

### 展开 Popup

#### 捕获鼠标输入

展开 Popup 前，以 `CaptureMode.SubTree` 方式捕获鼠标输入，捕获 ComboBox 及其子树以外的鼠标输入到 ComboBox 自身，这就做到了阻止其他控件响应鼠标，而自身和 Popup 不受影响。关闭 Popup 后，释放鼠标捕获。

### 关闭 Popup

#### 丢失鼠标捕获

有各种原因会引发 `Mouse.LostMouseCapture` 附加事件，例如窗口丢失焦点、其他控件抢走鼠标捕获。如果是 ComboBox 本体内部或 Popup 内部的元素抢走鼠标捕获，不应该关闭 Popup，还应该在内部元素释放鼠标捕获后重新由 ComboBox 捕获鼠标输入。

在 ComboBox 上处理 `Mouse.LostMouseCapture` 附加事件，通过 `e.OriginalSource` 判断哪个元素刚才丢失了鼠标捕获，通过 `Mouse.Captured` 获取哪个元素正捕获了鼠标。由于 `Mouse.LostMouseCapture` 附加事件是冒泡传播的路由事件，内部元素发生 `Mouse.LostMouseCapture` 时，也会在当前元素上引发事件。

对于 `Mouse.LostMouseCapture` 事件要连同已处理的事件一同处理，因为要靠这个追踪刚才丢失捕获的和现在正在捕获鼠标输入的元素。

##### 本体正在捕获

这种情况下，在 ComboBox 本体上引发此事件是由于本体捕获了鼠标输入，导致内部内部元素引发了 `Mouse.LostMouseCapture` 路由事件，此时 Popup 无需关闭。

##### 内部正在捕获

由于内部元素的交互逻辑内部元素捕获鼠标输入，从而在 ComboBox 本体引发了 `Mouse.LostMouseCapture` 事件，此时 Popup 无需关闭。

##### 外部正在捕获

因 ComboBox 之外的元素捕获了鼠标输入，导致 ComboBox 或其内部元素引发了 `Mouse.LostMouseCapture` 事件，此时 Popup 需要关闭。

##### 没有元素捕获

所有元素都没有捕获鼠标输入，此时 Popup 需要关闭。

如果现在正是 ComboBox 本体捕获了鼠标，那么Popup 无需关闭；如果刚才丢失鼠标捕获的元素位于 ComboBox 或 Popup 内部，说明内部元素的交互逻辑就是要捕获鼠标输入的，交互结束后释放了鼠标捕获，那就再由 ComboBox 本体继续捕获鼠标；如果现在没有捕获鼠标输入的元素，或捕获了的元素在外面，那么 Popup 需要关闭。

#### 点击外面时关闭

鼠标输入捕获期间，所有位于外面的鼠标事件都会输入到 ComboBox 本体，并且引发事件参数里的 `OriginalSource` 也是 ComboBox 本体。因此只需要处理 ComboBox 本体接收到的未被处理的 `MouseDown` 路由事件即可准备关闭 Popup，关闭后应释放鼠标捕获。

需要注意 `Popup` 的内容是独立的视觉树，判断元素是否位于内部时不能只用基于视觉树的逻辑，`Popup.Child` 是进入这个独立视觉树的一个简便方式。

### 其他鼠标事件

Popup 展开时，鼠标捕获期间，需要把到达 `ComboBox` 本体时仍未处理的鼠标事件标记为已处理，让鼠标事件不再向外传播，以此满足 Popup 展开期间用户操作与外界相对隔离的预期。

### 通用工具类

`ComboBox` 封装了太多功能，展开的 Popup 内容也不自由。现在要编写一个工具类 `PopupOpenStateController`，封装类似 `ComboBox` 控制 Popup 展开和关闭的控制逻辑，以便为其他需要类似 Popup 控制逻辑的场景使用。

#### 设计

##### 构造函数

直观感受上，只需要传入要使用此逻辑的控件即可，但由于 `Popup` 里的内容在单独的视觉树中，并且不知道控件内部有几个 Popup，还需要传入要控制的 `Popup` 参数。

```csharp
public PopupOpenStateController(UIElement uiElement, Popup containedPopup)
{
    _uiElement = uiElement;
    _containedPopup = containedPopup;
    _mouseDownEventHandler = new MouseButtonEventHandler(OnMouseDown);
    _mouseWheelEventHandler = new MouseWheelEventHandler(OnMouseWheel);
    _lostMouseCaptureEventHandler = new MouseEventHandler(OnLostMouseCapture);
    _uiElement.AddHandler(Mouse.MouseDownEvent, _mouseDownEventHandler);
    _uiElement.AddHandler(Mouse.MouseWheelEvent, _mouseWheelEventHandler);
    _uiElement.AddHandler(Mouse.LostMouseCaptureEvent, _lostMouseCaptureEventHandler, handledEventsToo: true);
}

private readonly UIElement _uiElement;
private readonly Popup _containedPopup;
private readonly MouseButtonEventHandler _mouseDownEventHandler;
private readonly MouseWheelEventHandler _mouseWheelEventHandler;
private readonly MouseEventHandler _lostMouseCaptureEventHandler;
```

##### 取消订阅

控制逻辑需要订阅一些路由事件，为了在合适的时候取消订阅，此类应实现 `IDisposable` 并在 `Dispose` 中取消订阅事件。

```csharp
public void Dispose()
{
    _uiElement.RemoveHandler(UIElement.MouseDownEvent, _mouseDownEventHandler);
    _uiElement.RemoveHandler(UIElement.MouseWheelEvent, _mouseWheelEventHandler);
    _uiElement.RemoveHandler(UIElement.LostMouseCaptureEvent, _lostMouseCaptureEventHandler);
}
```

##### 控制方式

虽然构造时传入了 Popup，但不应直接控制 `Popup.IsOpen` 属性，因为要留给使用者自行控制展开和关闭行为的余地，例如过渡动画。

将采用事件的方式，在需要展开或关闭的时机通知，由调用处自行处理。

```csharp
public event EventHandler OpenPopupRequested;
public event EventHandler ClosePopupRequested;
```

还提供 `ShouldPopupOpen` 属性，用于随时查询开关状态。

```csharp
public bool ShouldPopupOpen { get; set; }
```

此属性是可被外部设置的，这意味着外部也可以通过设置 `ShouldPopupOpen` 属性来引发 `OpenPopupRequested` 和 `ClosePopupRequested` 事件。

这样做的好处是，当调用处希望直接控制 Popup 展开或关闭时，可以直接设置 `ShouldPopupOpen` 属性，不需要分别处理了来自自身的需求和 `PopupOpenStateController` 的事件，这样调用处就可以完全依赖 `OpenPopupRequested` 和 `ClosePopupRequested` 事件去触发真正的 Popup 开关行为。

#### 实现

##### 鼠标交互

```csharp
/// <remarks>
/// 展开 Popup 时，已经通过 <c>Mouse.Capture(this, CaptureMode.SubTree)</c> 捕获了鼠标输入，
/// 所有其他位置的鼠标输入都会输入到捕获鼠标交互的控件身上，
/// 随之引发的事件参数 <see cref="RoutedEventArgs.OriginalSource"/> 也是当前控件。
/// </remarks>
private void OnMouseDown(object sender, MouseButtonEventArgs e)
{
    if (Mouse.Captured == sender)
    {
        e.Handled = true;
    }
    if (e.OriginalSource == sender)
    {
        ShouldPopupOpen = false;
    }
}

private void OnMouseWheel(object sender, MouseWheelEventArgs e)
{
    if (_containedPopup.IsOpen)
    {
        e.Handled = true;
    }
}

private void OnLostMouseCapture(object sender, MouseEventArgs e)
{
    if (!ShouldPopupOpen)
        return;

    if (Mouse.Captured == sender)
        return;

    if (ContainsElement((DependencyObject)e.OriginalSource))  // 刚才丢失鼠标捕获的元素位于内部
    {
        Mouse.Capture(_uiElement, CaptureMode.SubTree);
    }
    else if (Mouse.Captured is null || !ContainsElement((DependencyObject)Mouse.Captured))
    {
        ShouldPopupOpen = false;
    }
}

/// <summary>
/// <paramref name="element"/> 是否在 <see cref="_uiElement"/> 里面
/// </summary>
private bool ContainsElement(DependencyObject element)
{
    if (_uiElement != element && _uiElement.IsAncestorOf(element))
        return true;

    UIElement popupChild = _containedPopup?.Child;
    if (popupChild != null && popupChild.IsAncestorOf(element))
        return true;

    return false;
}
```

##### 通知 Popup 状态

```csharp
public bool ShouldPopupOpen
{
    get;
    set
    {
        if (field == value)
            return;

        field = value;
        if (value)
        {
            RequestOpenPopup();
        }
        else
        {
            RequestClosePopup();
        }
    }
}

private void RequestOpenPopup()
{
    Mouse.Capture(_uiElement, CaptureMode.SubTree);
    OpenPopupRequested?.Invoke(this, EventArgs.Empty);
}

private void RequestClosePopup()
{
    _uiElement.ReleaseMouseCapture();
    ClosePopupRequested?.Invoke(this, EventArgs.Empty);
}
```

##### 辅助逻辑

判断控件本体和 Popup 是否包含指定元素。

```csharp
/// <summary>
/// <paramref name="element"/> 是否在 <see cref="_uiElement"/> 里面
/// </summary>
private bool ContainsElement(DependencyObject element)
{
    if (_uiElement != element && _uiElement.IsAncestorOf(element))
        return true;

    if (_containedPopup?.Child != null && _containedPopup.Child.IsAncestorOf(element))
        return true;

    return false;
}
```

#### 完整代码

```csharp
/// <summary>
/// 类似 <see cref="System.Windows.Controls.ComboBox"/> 候选项展开菜单的 Popup 开启状态控制逻辑。
/// </summary>
/// <remarks>
/// 打开 Popup 时，捕获鼠标输入，处理掉鼠标事件，阻止鼠标事件向外传播。
/// 打开 Popup 期间，如果鼠标点击了既不是控件本身也不是 Popup 以外的区域，则关闭 Popup.
/// </remarks>
public class PopupOpenStateController : IDisposable
{
    public PopupOpenStateController(UIElement uiElement, Popup containedPopup)
    {
        _uiElement = uiElement;
        _containedPopup = containedPopup;
        _mouseDownEventHandler = new MouseButtonEventHandler(OnMouseDown);
        _mouseWheelEventHandler = new MouseWheelEventHandler(OnMouseWheel);
        _lostMouseCaptureEventHandler = new MouseEventHandler(OnLostMouseCapture);
        _uiElement.AddHandler(UIElement.MouseDownEvent, _mouseDownEventHandler);
        _uiElement.AddHandler(UIElement.MouseWheelEvent, _mouseWheelEventHandler);
        _uiElement.AddHandler(UIElement.LostMouseCaptureEvent, _lostMouseCaptureEventHandler, handledEventsToo: true);
    }

    private readonly UIElement _uiElement;
    private readonly Popup _containedPopup;
    private readonly MouseButtonEventHandler _mouseDownEventHandler;
    private readonly MouseWheelEventHandler _mouseWheelEventHandler;
    private readonly MouseEventHandler _lostMouseCaptureEventHandler;

    public event EventHandler OpenPopupRequested;
    public event EventHandler ClosePopupRequested;

    public bool ShouldPopupOpen
    {
        get;
        set
        {
            if (field == value)
                return;

            field = value;
            if (value)
            {
                RequestOpenPopup();
            }
            else
            {
                RequestClosePopup();
            }
        }
    }

    public bool IsPopupOpen => _containedPopup.IsOpen;

    private void RequestOpenPopup()
    {
        Mouse.Capture(_uiElement, CaptureMode.SubTree);
        OpenPopupRequested?.Invoke(this, EventArgs.Empty);
    }

    private void RequestClosePopup()
    {
        Mouse.Capture(null);
        ClosePopupRequested?.Invoke(this, EventArgs.Empty);
    }

    /// <remarks>
    /// 展开 Popup 时，已经通过 <c>Mouse.Capture(this, CaptureMode.SubTree)</c> 捕获了鼠标输入，
    /// 所有其他位置的鼠标输入都会输入到捕获鼠标交互的控件身上，
    /// 随之引发的事件参数 <see cref="RoutedEventArgs.OriginalSource"/> 也是当前控件。
    /// </remarks>
    private void OnMouseDown(object sender, MouseButtonEventArgs e)
    {
        if (Mouse.Captured == sender)
        {
            e.Handled = true;
        }
        if (e.OriginalSource == sender)
        {
            ShouldPopupOpen = false;
        }
    }

    private void OnMouseWheel(object sender, MouseWheelEventArgs e)
    {
        if (_containedPopup.IsOpen)
        {
            e.Handled = true;
        }
    }

    private void OnLostMouseCapture(object sender, MouseEventArgs e)
    {
        if (!ShouldPopupOpen)
            return;

        if (Mouse.Captured == sender)  // 内部元素失去鼠标捕获，自身正在捕获鼠标输入
            return;

        if (Mouse.Captured != null && ContainsElement((DependencyObject)Mouse.Captured))  // 内部元素正在捕获鼠标输入
            return;

        if (ContainsElement((DependencyObject)e.OriginalSource))  // 刚才丢失鼠标捕获的元素位于内部
        {
            Mouse.Capture(_uiElement, CaptureMode.SubTree);
        }
        else  // 没有元素或外部元素正在捕获鼠标输入
        {
            ShouldPopupOpen = false;
        }
    }

    /// <summary>
    /// <paramref name="element"/> 是否在 <see cref="_uiElement"/> 里面
    /// </summary>
    private bool ContainsElement(DependencyObject element)
    {
        if (_uiElement != element && _uiElement.IsAncestorOf(element))
            return true;

        if (_containedPopup?.Child != null && _containedPopup.Child.IsAncestorOf(element))
            return true;

        return false;
    }

    public void Dispose()
    {
        _uiElement.RemoveHandler(UIElement.MouseDownEvent, _mouseDownEventHandler);
        _uiElement.RemoveHandler(UIElement.MouseWheelEvent, _mouseWheelEventHandler);
        _uiElement.RemoveHandler(UIElement.LostMouseCaptureEvent, _lostMouseCaptureEventHandler);
    }
}
```
