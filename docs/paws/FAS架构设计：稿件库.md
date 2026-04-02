# FAS架构设计：稿件库

> *注：标题的FAS指本人即将开发的稿件管理器：Furry Art Studio的缩写，与FaaS或者其他相关的概念无任何关系。*

对于一个稿件管理工具来说，我们先下个定义：稿件是什么？

稿件（Manuscript）或艺术品（Artworks）其实可以认定为一个概念，后者的范围更广。接触过Furry文化的都知道：一个稿件本质是一个JPEG/PNG/GIF等图片格式组成，并且JPEG与PNG格式占稿件的90%以上。同时一份“稿件”包含很多的**差分**：

**差分**，顾名思义，它是一种描述数据变化的趋势，例如在第一秒某物体速度是`1m/s`，而第二秒它的速度变为`1.5m/s`，二者的差分就是`0.5`，通常使用希腊字母德尔塔（`Δ`）表示

在稿件中，差分可以理解为一张图的不同版本，比如在保证主要部分没有变化的前提修改人物的动作，表情，滤镜等等。

这样，我们就清楚了稿件的本质：**一个图像集合**。

要想将稿件数字化并存储，我们需要使用**数据库**（Database），它不仅专业，更能更快的处理相当多的数据。在单机应用里，我们使用`SQLite`数据库，它相比于其他的数据库，本身简单（仅一个文件），不需要中央服务器（应用本身就是这个数据库的“服务器”），非常符合我们的设计。

对于一个稿件，我们在使用传统的修改文件名的方式存储时，通常需要记录：它是**谁画的**，他**叫什么**，这是两个最基本的信息。

我从MP3文件的元数据（Metadata）里得到灵感，使用如下的类来**定义**一个稿件（`Artwork`）：

```vbnet
Public Class Artwork ' 定义稿件属性
    Public Property ID As Integer            ' 数据库自增主键
    Public Property UUID As Guid             ' 稿件UUID
    Public Property Title As String          ' 稿件标题
    Public Property Author As String         ' 稿件作者
    Public Property Characters As String()   ' 稿件内角色数组
    Public Property CreateTime As DateTime   ' 稿件绘制时间
    Public Property ImportTime As DateTime   ' 稿件导入数据库时间
    Public Property UpdateTime As DateTime   ' 稿件更新元数据时间
    Public Property IsDeleted As Integer     ' 稿件状态(正常/已删除)
    Public Property Tags As String()         ' 稿件标签数组
    Public Property Notes As String          ' 稿件备注
    Public Property FileNames As String()    ' 稿件文件名数组
End Class
```
这样，我们就可以成功地将稿件“数字化”了，这里的数字化指的是对稿件本身进行定义（图像集合），打标签（Tagging），建立索引（Index），最终目的是便于我们查询（Query）。

其实，在设计这套系统的时候，我有在想该怎么处理稿件本身的内容（数据），SQLite里有一个类型就是二进制（BLOB类型），我最初设计的是将稿件转换成二进制，存储在数据库里，但是这样有几个问题：

   - 读写文件时，尤其是加载缩略图的时候，需要等待大量的时间，或者需要通过多线程读取数据库文件，这就导致对计算机性能要求很高。
   - 如果数据库文件出现了损坏，那么所有的稿件都会被困在里面，无法读取，这对于画师来说是灾难性的，即使现代的数据库由于ACID的存在，能够尽最大努力保住数据，但仍然需要考虑这种可能性。相比之下文件系统会稳定得多。
   - 所有数据存储在数据库里时，文件大小会变得很大。

详细对比如下：

| 维度 | 仅数据库存储 | 仅文件系统存储 | 混合存储 |
| ---- | ----------- | ------------- | ------- |
| 特点 | 文件易于管理，迁移时作为整体，速度会快很多。但数据易损坏，对计算机性能要求高，文件体积大。 | 对不同设备友好，不需要数据库组件。但是很难查询，需要借助第三方工具，难以标准化维护和管理 | 将元数据等字段通过数据库存储，通过UUID建立的文件夹来存放稿件本身原本的文件，分开管理稿件元数据与稿件本身。

在有了稿件之后，我们就可以定义一个新的类：**稿件列表**（稿件数据库），它的作用就是管理和抽象化那些对于稿件本身，以及元数据的操作的部分，比如我们可以这样定义：

```vbnet
Public Class ArtworkLibrary ' 定义稿件库实例类
    Public Function AddArtwork(artwork As Artwork) As Integer
        ' 添加新的稿件
    End Function

    Public Sub UpdateArtwork(artwork As Artwork)
        ' 更新稿件
    End Sub

    Public Function GetArtworkByUUID(uuid As Guid) As Artwork
        ' 根据UUID获得稿件
    End Function
    ' ...其他方法
End Class
```

这样，我们通过这个类，在实例化的时候将保存的文件夹与数据库文件位置作为参数，我们就可以直接对这个混合存储进行管理了。

但很快我意识到另一个问题：假设我们的系统可以连接多个稿件库（比如有些画师或者设主具有多个设定，或者想把个人工作与共享作品分开），我们就得对`ArtworkLibrary`这个实例类进行管理，于是我这样设计：

```vbnet
Public Class LibraryManager ' 定义稿件库管理器单实例类
    Private Shared _instance As LibraryManager
    Private _libraries As New Dictionary(Of String, ArtworkLibrary)
    Private _currentLibrary As ArtworkLibrary

    Public Shared ReadOnly Property Instance As LibraryManager
        Get
            If _instance Is Nothing Then
                _instance = New LibraryManager()
            End If
            Return _instance
        End Get
    End Property

    Private Sub New()
        '单例模式
    End Sub

    Public Sub AddLibrary(name As String)
        If Not _libraries.ContainsKey(name) Then
            Dim library As New ArtworkLibrary(name)
            _libraries.Add(name, library)
        End If
    End Sub

    Public Function SwitchLibrary(name As String) As Boolean
        If _libraries.ContainsKey(name) Then
            _currentLibrary = _libraries(name)
            Return True
        End If
        Return False
    End Function
    '...其他方法
End Class
```

通过这个方式，我们就可以方便地管理每个稿件库，而每个稿件库下面还有对应的稿件。

下一步就是UI与业务逻辑层了，先实现基本的增删改查（CRUD），至于`PawUI`那部分可能需要过段时间再绘制了。

而我的最终目标，就是尽可能让FAS的数据库与PawCore兼容，这样用户可以无缝衔接过去。

*创作于2026-1-23*