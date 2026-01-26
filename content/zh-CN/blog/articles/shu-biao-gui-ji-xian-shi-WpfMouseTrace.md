---
title: "WPF 鼠标轨迹追踪软件 WpfMouseTrace 开源"
date: 2026-01-26T17:59:46+08:00
draft: false
---

 [项目链接](https://github.com/dandll/WpfMouseTrace)
 [下载链接](https://github.com/dandll/WpfMouseTrace/releases/download/Release/WpfMouseTrace_Windows.zip)

# WPF 鼠标轨迹追踪项目技术说明

## 项目概述

这是一个基于 WPF (Windows Presentation Foundation) 开发的鼠标轨迹追踪工具，能够在屏幕上实时显示鼠标移动的轨迹效果。项目采用了多种技术优化，实现了高性能的图形渲染和用户友好的配置界面。

## 核心功能

### 1. 实时鼠标轨迹显示
- 通过 Win32 API 获取全局鼠标位置
- 使用自定义 Canvas 控件进行高性能渲染
- 支持线条绘制，轨迹平滑过渡
- 渐变透明度和粗细效果

### 2. 系统托盘集成
- 托盘图标显示
- 右键菜单（设置、退出）
- 双击图标打开设置

### 3. 可配置化
- 最大轨迹数量可调（5-100）
- 轨迹颜色自定义
- 配置持久化保存

![alt text](Image/1.gif)

![alt text](Image/2.gif)

## 技术架构

### 1. 主窗口 (MainWindow)

#### 1.1 Win32 API 集成

```csharp
[DllImport("user32.dll")]
public static extern bool GetCursorPos(out POINT lpPoint);

[DllImport("user32.dll")]
public static extern IntPtr GetWindowLong(IntPtr hWnd, int nIndex);

[DllImport("user32.dll")]
public static extern int SetWindowLong(IntPtr hWnd, int nIndex, int dwNewLong);
```

使用 P/Invoke 调用 Windows API：
- `GetCursorPos`: 获取全局鼠标坐标
- `GetWindowLong` / `SetWindowLong`: 修改窗口样式

#### 1.2 鼠标穿透功能

```csharp
private const int GWL_EXSTYLE = -20;
private const int WS_EX_TRANSPARENT = 0x00000020;

SetWindowLong(hwnd, GWL_EXSTYLE,
    ((int)GetWindowLong(hwnd, GWL_EXSTYLE) | WS_EX_TRANSPARENT));
```

通过设置 `WS_EX_TRANSPARENT` 标志，使窗口对鼠标事件透明，实现点击穿透效果，不影响底层窗口的操作。

#### 1.3 定时器机制

```csharp
_timer = new DispatcherTimer();
_timer.Interval = TimeSpan.FromMilliseconds(33);
_timer.Tick += OnTimerTick;
_timer.Start();
```

使用 33ms 间隔（约 30fps）的定时器：
- 平衡性能和视觉效果
- 降低内存占用
- 减少垃圾回收压力

#### 1.4 坐标转换

```csharp
var point = PointFromScreen(new System.Windows.Point(pt.X, pt.Y));
```

将屏幕坐标（像素）转换为窗口内的设备无关像素（DIP），确保在不同 DPI 设置下正确显示。

### 2. 自定义绘制控件 (TrailCanvas)

#### 2.1 继承 Canvas

```csharp
public class TrailCanvas : Canvas
{
    protected override void OnRender(DrawingContext drawingContext)
    {
    }
}
```

重写 `OnRender` 方法实现自定义绘制逻辑。

#### 2.2 轨迹点管理

```csharp
public Queue<Point> TrailPoints { get; set; } = new Queue<Point>();
private Point[] _pointsCache = new Point[MaxTrailLength];
```

- 使用 `Queue<Point>` 存储轨迹点
- 使用缓存数组避免频繁内存分配
- 动态调整缓存大小

#### 2.3 渲染逻辑

```csharp
for (int i = 0; i < count - 1; i++)
{
    double alpha = (i / (double)count);
    byte alphaByte = (byte)(alpha * 200);
    
    var brush = new SolidColorBrush(Color.FromArgb(alphaByte, TrailColorR, TrailColorG, TrailColorB));
    double thickness = 8 * alpha + 2;
    var pen = new Pen(brush, thickness);
    drawingContext.DrawLine(pen, _pointsCache[i], _pointsCache[i + 1]);
}
```

- 渐变透明度：旧轨迹点更透明
- 渐变粗细：新轨迹点更粗
- 使用 `DrawLine` 绘制连续线条

### 3. 设置管理 (Settings)

#### 3.1 单例模式

```csharp
private static Settings _instance;

public static Settings Instance
{
    get
    {
        if (_instance == null)
        {
            _instance = Load();
        }
        return _instance;
    }
}
```

确保全局唯一的设置实例。

#### 3.2 数据持久化

```csharp
[DataContract]
public class Settings : INotifyPropertyChanged
{
    [DataMember]
    public int MaxTrailLength { get; set; }
    
    public void Save()
    {
        var serializer = new DataContractJsonSerializer(typeof(Settings));
        using (var stream = new FileStream(SettingsFilePath, FileMode.Create))
        {
            serializer.WriteObject(stream, this);
        }
    }
}
```

- 使用 `DataContractJsonSerializer` 序列化配置
- 保存到 `%AppData%\WpfMouseTrace\settings.json`
- 实现 `INotifyPropertyChanged` 支持数据绑定

#### 3.3 属性变更通知

```csharp
public event PropertyChangedEventHandler PropertyChanged;

protected void OnPropertyChanged(string propertyName)
{
    PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
}
```

支持 WPF 数据绑定，实现 UI 自动更新。

### 4. 系统托盘 (NotifyIcon)

#### 4.1 图标加载

```csharp
var iconStream = System.Windows.Application.GetResourceStream(
    new Uri("pack://application:,,,/Resources/favicon(1).ico"))?.Stream;

_notifyIcon = new NotifyIcon
{
    Icon = iconStream != null ? new System.Drawing.Icon(iconStream) : SystemIcons.Application,
    Text = "鼠标轨迹追踪",
    Visible = true
};
```

从程序资源加载图标，使用 WPF Pack URI。

#### 4.2 右键菜单

```csharp
var contextMenu = new ContextMenuStrip();
var settingsItem = new ToolStripMenuItem("设置");
settingsItem.Click += (s, e) => OpenSettings();

var exitItem = new ToolStripMenuItem("退出");
exitItem.Click += (s, e) => ExitApplication();

contextMenu.Items.AddRange(new ToolStripItem[] { settingsItem, exitItem });
_notifyIcon.ContextMenuStrip = contextMenu;
```

使用 Windows Forms 的 `ContextMenuStrip` 创建托盘菜单。

### 5. 颜色选择器集成

```csharp
private void ColorPreviewBorder_MouseLeftButtonUp(object sender, MouseButtonEventArgs e)
{
    var colorDialog = new ColorDialog
    {
        Color = System.Drawing.Color.FromArgb(
            _settings.TrailColorR,
            _settings.TrailColorG,
            _settings.TrailColorB),
        FullOpen = true
    };

    var result = Dispatcher.Invoke(() => colorDialog.ShowDialog());
    
    if (result == System.Windows.Forms.DialogResult.OK)
    {
        _settings.TrailColorR = colorDialog.Color.R;
        _settings.TrailColorG = colorDialog.Color.G;
        _settings.TrailColorB = colorDialog.Color.B;
    }
}
```

- 使用 `ColorDialog` 提供系统颜色选择器
- 通过 `Dispatcher.Invoke` 确保在主线程显示
- 自动更新设置属性

## 性能优化

### 1. 内存优化

#### 1.1 对象缓存

```csharp
private Point[] _pointsCache = new Point[MaxTrailLength];
```

避免每次渲染都创建新数组，减少 GC 压力。

#### 1.2 降低刷新率

```csharp
_timer.Interval = TimeSpan.FromMilliseconds(33); // 30fps
```

从 60fps 降到 30fps，减少内存分配频率。

#### 1.3 资源释放

```csharp
protected override void OnClosing(CancelEventArgs e)
{
    _notifyIcon?.Dispose();
    base.OnClosing(e);
}
```

及时释放托盘图标资源。

### 2. 渲染优化

#### 2.1 使用 DrawingContext

```csharp
protected override void OnRender(DrawingContext drawingContext)
{
    drawingContext.DrawLine(pen, _pointsCache[i], _pointsCache[i + 1]);
}
```

直接使用 `DrawingContext` 绘制，避免创建大量 UI 元素。

#### 2.2 最小化重绘

```csharp
TrailCanvas.InvalidateVisual();
```

只在数据变化时触发重绘，避免不必要的渲染。

## 技术亮点

### 1. WPF 与 WinForms 混合使用

- WPF 用于主界面和渲染
- WinForms 用于托盘图标和颜色选择器
- 通过互操作实现无缝集成

### 2. P/Invoke 调用 Windows API

- 获取全局鼠标位置
- 修改窗口样式实现鼠标穿透
- 跨平台能力（仅限 Windows）

### 3. 数据绑定

- 设置属性实现 `INotifyPropertyChanged`
- XAML 中使用 `Binding` 自动更新 UI
- 简化代码逻辑

### 4. 资源管理

- 图标作为资源嵌入程序集
- 使用 Pack URI 引用资源
- 统一的资源管理方式

## 项目结构

```
WpfMouseTrace/
├── WpfMouseTrace/
│   ├── MainWindow.xaml          # 主窗口界面
│   ├── MainWindow.xaml.cs       # 主窗口逻辑
│   ├── TrailCanvas.cs          # 自定义绘制控件
│   ├── SettingsWindow.xaml      # 设置窗口界面
│   ├── SettingsWindow.xaml.cs   # 设置窗口逻辑
│   ├── Settings.cs            # 设置管理类
│   ├── App.xaml              # 应用程序入口
│   ├── App.xaml.cs           # 应用程序初始化
│   ├── WpfMouseTrace.csproj # 项目文件
│   └── Resources/
│       └── favicon(1).ico   # 应用图标
```

## 依赖项

### .NET Framework
- 目标框架：.NET Framework 4.5.2

### 程序集引用
- PresentationCore
- PresentationFramework
- WindowsBase
- System.Xaml
- System.Drawing
- System.Windows.Forms

## 扩展性

### 1. 添加新的轨迹效果

可以扩展 `TrailCanvas` 类，支持不同的渲染模式：
- 圆点模式
- 虚线模式
- 粒子效果

### 2. 添加更多设置项

在 `Settings` 类中添加新属性：
- 轨迹粗细范围
- 淡出速度
- 启用/禁用功能

### 3. 多主题支持

扩展设置系统，支持：
- 预设主题
- 导入/导出配置
- 主题切换

## 总结

本项目展示了 WPF 开发的多个核心技术：
- 自定义控件和渲染
- Win32 API 集成
- 数据绑定和 MVVM 模式
- 性能优化技巧
- 混合使用 WPF 和 WinForms

通过合理的架构设计和性能优化，实现了一个功能完整、性能良好的鼠标轨迹追踪工具。
