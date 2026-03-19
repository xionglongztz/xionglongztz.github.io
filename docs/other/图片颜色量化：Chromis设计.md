# 图片颜色量化：Chromis设计

我的[FurryArtStudio](https://github.com/xionglongztz/FurryArtStudio)（下文简称**FAS**）终于有一些人气了，在短短一个月左右的时间，已经收到7个Pull Requests，3个Issues，贡献者达4名，提交总数超100，目前我考虑设计一个新的功能在我的项目里，也就是对图片进行量化处理。

## 前言

在很久之前，我设计了一个还没开源的小工具**ImageWizard**，它是一个图片编辑器，我最初设想是制作一个WinForms应用，能够比PhotoShop开启的更快，同时能够进行简单的图片编辑。但这个项目很快就放弃了，因为我还没研究过**图形API**，我是使用`Drawing`的`SetPixel()`对图像进行操作的，甚至我还没有使用并行处理，速度极慢，靠我的**i7-10870**CPU处理的话，大致需要3秒才会处理完一张`1920*1080`的图片。

不过，我想到有些Furry创作者可能会使用**颜色提取**功能，因为用作参考非常方便，所以我把我之前的**ImageWizard**里面的算法提取出来，用作我的**FAS**的一部分。

在1.0.6版本，我考虑添加一个属性窗口，允许用户在图片预览器里按下特定组合键并查看**稿件属性**（SQLite里存储的元数据字段），**文件属性**（文件大小，类型，哈希等），**图像属性**（图像位深，分辨率，以及今天要讲的**颜色提取**）。

我搜了下相关内容，发现主流的颜色提取算法大致如下：

| 算法 | K-Means聚类 | 中位切分（Median Cut） | 八叉树（Octree） |
| --- | ------------ | --------------------- | --------------- |
| 速度 | 慢 | 中等 | **快** |
| 准确度 | **精准** | 粗略 | 中等 |
| 性能 | 平衡 | **快速** | 内存高效 |
| 量化程度 | 受初始聚类点影响 | 较为粗略，受限于算法性质 | **精准** |

三种方法各有优劣，取决于具体使用场景。

## K-Means聚类算法

 - 首先定义n个初始点，这n个点作为聚类中心。
 - 然后计算所有像素距离这些聚类中心的距离（**欧几里得距离**，也就是**两点最短长度**），寻找最小的那个，把像素添加到对应的聚类中。
 - **欧几里得算法**具体计算方式是计算`Δr=R1-R2`，`Δg=G1-G2`，`Δb=B1-B2`，随后计算**根号下**`Δr^2 + Δg^2 + Δb^2`。
 - 随后把每个聚类中心的全部点的像素求平均值，也就是把全部的R加起来，除以点总数求出平均颜色的R值，同理计算G和B的值。
 - 继续迭代，直至颜色**收敛**或**达到设定迭代次数**。

## 中位切分（Median Cut）

 - 将全部颜色放到一个立方体空间中（将R，G和B分别映射为空间中X，Y和Z坐标）。
 - 通过**递归**，切分立方体里最长的边，切的方向与立方体最长的平面垂直。
 - 继续切分，直至立方体数量等于所需要的像素数量。
 - 将每个立方体内颜色取平均值。

## 八叉树（Octree）

这是我觉得最有意思的一个算法。

 - 首先将颜色转换成二进制，得到它们的RGB分量。
 - 例如，`R=255`的二进制就是`0x11111111`，同理求出G和B的二进制。
 - 将二进制竖向排列，第一位的数字结合，得到一个三位二进制数。
 - 根据这些数，放置在树的叶片上
 - 一旦节点数足够，则清除这个节点，并把它下方的颜色合并

总的来说，还是很复杂，我其实也不是特别了解，只有K-Means算法我稍微读懂了一些。

## 开始设计

于是我设计了一个开源项目**Chromis**，它就是专门处理颜色量化的，最初我考虑将他用在跨平台项目里，但是它还是有些问题，比如在`.NET 8.0`下，它不支持`System.Drawing`下的`Image`和`Bitmap`等类，所以我尝试使用[ImageSharp](https://github.com/SixLabors/ImageSharp)等第三方库去处理，然而，我发现我的FAS是`.NET Framework 4.7.2`，新版本的`ImageSharp`不能完全支持。

于是后来，我就想到我这个类库的初衷：可以不处理**图像**，只处理**算法**，只要我使用的全都是跨平台的，最底层的数据类型，就可以不用担心跨平台操作了。

于是我自定义了一套数据类型，以及它的一些定义：

```vbnet
Public Class ColorExtractor
    ''' <summary>
    ''' 颜色结构体
    ''' </summary>
    Public Structure RGBColor
        Public R As Byte
        Public G As Byte
        Public B As Byte
        ''' <summary>
        ''' 根据RGB构造RGBColor
        ''' </summary>
        ''' <param name="r">红色</param>
        ''' <param name="g">绿色</param>
        ''' <param name="b">蓝色</param>
        ''' <returns><seealso cref="RGBColor"/>类型</returns>
        Public Shared Function FromRGB(r As Integer, g As Integer, b As Integer) As RGBColor
            Dim _color As New RGBColor With {
                .R = r,
                .G = g,
                .B = b
            }
            Return _color
        End Function

        ''' <summary>
        ''' 转换成十六进制
        ''' </summary>
        ''' <returns>获得十六进制格式(如#7CBDFF)</returns>
        Public Function ToHex() As String
            Return $"#{R:X2}{G:X2}{B:X2}"
        End Function

        ''' <summary>
        ''' 获得三元组格式
        ''' </summary>
        Public Function ToTuple() As (Byte, Byte, Byte)
            Return (R, G, B)
        End Function
    End Structure
    '...
End Class
```

这样，使用`.NET Framework`的可以使用`Drawing`下自带的库，使用`.NET Core`的可以食用`ImageSharp`等第三方库，真正做到跨平台。

随后，为了提升程序的可扩展性，我给每个算法提供了标准的接口定义：

```vbnet
Public Class ColorExtractor
    Public Interface IColorExtractor
        Function Extract(pixels As List(Of RGBColor), colorCount As Integer) As IReadOnlyList(Of ColorInfo)
    End Interface
End Class

Public Class KMeans
    Implements IColorExtractor
    Public Function Extract(pixels As List(Of RGBColor), clusterCount As Integer) As IReadOnlyList(Of ColorInfo) Implements IColorExtractor.Extract
        '...
    End Function
End Class

Public Class MedianCut
    Implements IColorExtractor
    Public Function Extract(pixels As List(Of RGBColor), colorCount As Integer) As IReadOnlyList(Of ColorInfo) Implements IColorExtractor.Extract
        '...
    End Function
End Class

Public Class Octree
    Implements IColorExtractor
    Public Function Extract(pixels As List(Of RGBColor), colorCount As Integer) As IReadOnlyList(Of ColorInfo) Implements IColorExtractor.Extract
        '...
    End Function
End Class
```

（完整代码可以看我的[GitHub](https://github.com/xionglongztz/Chromis)仓库）

同时，为了让返回的值标准化，我写了一个新的结构体，包含了之前定义的颜色结构体：

```vbnet
    ''' <summary>
    ''' 包含一套颜色与占比的结果结构体
    ''' </summary>
    Public Structure ColorInfo
        ''' <summary>
        ''' 颜色
        ''' </summary>
        Public Color As RGBColor
        ''' <summary>
        ''' 该颜色对应的比率, 范围为 0 到 1
        ''' </summary>
        Public Ratio As Single
    End Structure
```

为了保证方便使用，我在`ColorExtracter.vb`这个主类里添加了一个快捷使用方法，通过接口定义。同时，为了保证这个类库的扩展性，我添加了这么一个工厂，允许用户自定义自己的提取方法，它可以添加自己的方法到枚举中，同时通过前文定义的接口来维持契约：

```vbnet
    ''' <summary>
    ''' 提取类型
    ''' </summary>
    Public Enum ExtractType
        ''' <summary>
        ''' K-Means聚类
        ''' </summary>
        KMeans
        ''' <summary>
        ''' 中位切分
        ''' </summary>
        MedianCut
        ''' <summary>
        ''' 八叉树
        ''' </summary>
        Octree
    End Enum

    ''' <summary>
    ''' 提取图像的主要颜色
    ''' </summary>
    ''' <param name="pixels">要被处理的像素点</param>
    ''' <param name="colorCount">颜色数量</param>
    ''' <param name="extractType">(可选)提取方式,默认为<seealso cref="Octree"/></param>
    Public Shared Function Extract(pixels As List(Of RGBColor), colorCount As Integer, Optional extractType As ExtractType = ExtractType.Octree) As List(Of ColorInfo)
        Dim extractor = ExtractorFactory(extractType)(colorCount)
        Return extractor.Extract(pixels, colorCount)
    End Function

    ''' <summary>
    ''' 注册一个自己的提取算法
    ''' </summary>
    ''' <param name="type">提取类型名称</param>
    ''' <param name="factory">工厂</param>
    Public Shared Sub Register(type As ExtractType, factory As Func(Of Integer, IColorExtractor))
        ExtractorFactory(type) = factory
    End Sub

    Private Shared ReadOnly ExtractorFactory As New Dictionary(Of ExtractType, Func(Of Integer, IColorExtractor)) From {
    {ExtractType.KMeans, Function(count) New KMeans()},
    {ExtractType.MedianCut, Function(count) New MedianCut()},
    {ExtractType.Octree, Function(count) New Octree()}
}
```

在使用时，需要将图片进行量化（降低处理时间），同时需要将`Rgb64`（`SixLabors.ImageSharp`）或`Color`（`System.Drawing`）等颜色类型转换成`RGBColor`（`Chromis`）需要使用的类型，而接收函数返回的数据时，需要反过来，示例代码如下：

```vbnet
Imports Chromis

Dim pixels As New List(Of ColorExtractor.RGBColor)
'将已量化的像素转换成 ColorExtractor.RGBColor 类型
For Each sampledColor In GetPixelsFromImage(PictureBox1.Image)
    pixels.Add(ColorExtractor.RGBColor.FromRGB(sampledColor.R, sampledColor.G, sampledColor.B))
Next
'提取
Dim colorInfos = ColorExtractor.Extract(pixels, 10)
'输出颜色值与百分比
For Each ci In colorInfos
    Console.WriteLine($"{ci.Color.R}, {ci.Color.G}, {ci.Color.B} - {ci.Ratio:P}")
Next
```

在本地测试没有问题后，编写一个GitHub Actions脚本，在构建成功后提交到NuGet：

```yaml
name: Publish NuGet

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: windows-latest
    environment: Nuget

    steps:
    - uses: actions/checkout@v3

    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 8.0.x

    - name: Restore
      run: dotnet restore

    - name: Build
      run: dotnet build -c Release --no-restore

    - name: Pack
      run: dotnet pack -c Release --no-build

    - name: Publish to NuGet
      run: dotnet nuget push "**/*.nupkg" --api-key ${{ secrets.NUGET_API_KEY }} --source https://api.nuget.org/v3/index.json
```

在使用前，我们首先在[NuGet](https://www.nuget.org/account/apikeys)那边获得一个APIkey，点击`Create`，设置名字（名字随意，我是设置成`Chromis Release`），然后设置`Glob pattern`为`Chromis*`，这就可以保证这个Key只能用作更新Chromis来使用。

**注意：Key只有生成时复制这一次机会，一旦刷新当前网页便再也不能复制，同时不要将这个APIKey泄露给其他人。**

回到GitHub，在项目仓库的顶端`Settings`下找到`Secrets and variables`，找到`Actions`，我们先新增一个`Environment`（环境，这里我写的是`Nuget`，注意要和GitHub Actions脚本里要保持一致，也就是`environment: Nuget`这一行）。

然后我们添加一个新的密钥，名称是`NUGET_API_KEY`（同样记得和GitHub Actions脚本保持一致），值填写我们前面复制的那串字符（大概像这样：`oy2pp***uv63q`）。

随后，我们就可以在每次提交带有`v`的标签时自动触发构建了。

为了更优雅，我编写了一个**PowerShell脚本**，可以直接运行：

```powershell
# ASCII Art
Write-Host "   ____ _                         _     " -ForegroundColor Cyan
Write-Host "  / ___| |__  _ __ ___  _ __ ___ (_)___ " -ForegroundColor Cyan
Write-Host " | |   | '_ \| '__/ _ \| '_ `` _ \| / __|" -ForegroundColor Cyan
Write-Host " | |___| | | | | | (_) | | | | | | \__ \" -ForegroundColor Cyan
Write-Host "  \____|_| |_|_|  \___/|_| |_| |_|_|___/" -ForegroundColor Cyan
Write-Host

# 定义
$scriptDir = $PSScriptRoot # 脚本路径

# 读取 Chromis.vbproj
$projectFile = Join-Path $scriptDir "Chromis.vbproj"
$content = Get-Content -Path $projectFile -Raw -ErrorAction Stop

# 正则匹配版本号
$pattern = '<Version>(.*?)</Version>'
$match = [regex]::Match($content, $pattern)

# 提取版本号并去除前后空格
$version = $match.Groups[1].Value.Trim()
if ([string]::IsNullOrWhiteSpace($version)) {
    Write-Error "Version is empty" -ForegroundColor Red
    Start-Sleep -Seconds 5 # 自动退出
    exit 1
}

# 构建标签名
$tagName = "v$version"
Write-Host "Version: $version"

# 检查本地与远程版本号
$localExists = $false
git rev-parse -q --verify "refs/tags/$tagName" > $null 2>&1
if ($LASTEXITCODE -eq 0) {
    $localExists = $true
    Write-Host "Deleting local version..."
    git tag -d $tagName
}
$remoteExists = $false
git ls-remote --exit-code origin "refs/tags/$tagName" > $null 2>&1
if ($LASTEXITCODE -eq 0) {
    $remoteExists = $true
    Write-Host "Deleting remote version..."
    git push origin :refs/tags/$tagName
}

git tag $tagName
git push origin $tagName

Write-Host "Nuget publish successfully." -ForegroundColor Green
Start-Sleep -Seconds 5 # 自动退出
```

运行后，它能够自动读取当前版本号，然后拼接到git命令上，最后自动推送，触发GitHub那边的生成和发布。

在发布后，大概需要等几分钟，验证成功后包就会在[NuGet](https://www.nuget.org/packages/Chromis)那边看到了，随后再等几分钟，所有使用这个项目的库在自己的Visual Studio 的NuGet包管理器里就可以看到更新了

## LOGO设计

这个名字是**ChatGPT**帮我起的，我其实想要让我的这个库足够小（实测只有十几KB），同时解决巨大的功能，类似于[Dapper](https://github.com/DapperLib/Dapper)这样的名字和这样的功能。

而当我设计LOGO时，我搜了下这个名字，它的名字叫[光鳃鱼](https://zh.wikipedia.org/wiki/%E5%85%89%E9%B0%93%E9%AD%9A)，不得不说ChatGPT这个起名有点图形学的感觉了。

于是我就开始设计LOGO，我想的是构思一个抽象的鱼的图案，同时周围有多个不同颜色的点作为点缀，同时代表它量化颜色的能力，最后还得考虑将他压缩至`128×128`尺寸以适配NuGet包管理器页面的要求，所以需要尽可能保留最多的细节。

在设计的基本没问题时，我同时设计了Banner图片，让它看起来更像是一个正经的库了。

现在就差坐着猛猛吸Star了（笑）。