# FAS路线：其一

本篇介绍了FAS的目前途径以及未来演进，需要注意未来很可能会重构很多部分，本文仅供参考。

## 总览

目前，我的FAS的架构如下（部分文件进行了标注）：

```text
FurryArtStudio
│  .gitattributes
│  .gitignore
│  App.config
│  FurryArtStudio.sln               # 解决方案文件
│  FurryArtStudio.vbproj
│  FurryArtStudio.vbproj.user
│  LICENSE.txt                      # 许可证文件
│  packages.config
│  README.md                        # 说明文件 - 简体中文
│  
├─.github
│  │  FUNDING.yml                   # 赞助链接（爱发电）
│  │  
│  ├─ISSUE_TEMPLATE
│  │      bug_report.yml            # 漏洞反馈模板
│  │      feature_request.yml       # 新功能反馈模板
│  │      
│  └─workflows
│          release.yml              # CI/CD 构建文件
│          
├─docs
│  │  Banner.png                    # GitHub头图
│  │  CHANGELOG.txt                 # 更新日志，由 release.ps1 自动维护
│  │  README_EN.md                  # 说明文件 - 英语（@element115mc）
│  │  README_JP.md                  # 说明文件 - 日语（@ra1nyxin）
│  │  
│  └─Tutorial                       # 教程库（未开始设计）
├─My Project
│      app.manifest
│      Application.Designer.vb
│      Application.myapp
│      AssemblyInfo.vb
│      Settings.Designer.vb
│      Settings.settings
│                  
├─script
│      release.ps1                  # GitHub Release 推送辅助脚本
│      
└─src
   ├─ArtworkManager                 # 稿件管理相关类
   │      Artwork.vb                # 稿件实体
   │      ArtworkLibrary.vb         # 稿件库实例
   │      LibraryManager.vb         # 库管理工厂类
   │      
   ├─Components                     # 自定义组件类
   │      GalleryImage.vb           # ImageGallery 专属图像类型
   │      ImageGallery.vb           # 图片墙组件
   │      
   ├─Docs                           # 文档
   │      ABOUT.txt                 # 关于
   │      PRIVACY.txt               # 隐私政策
   │      TERMS.txt                 # 用户协议
   │      WHATSNEW.txt              # 新增功能
   │      
   ├─Forms                          # 窗体
   │      AboutForm.vb              # 关于窗体
   │      ColorShowDialog.vb        # 图片主颜色提取结果对话框
   │      DevToolsForm.vb           # 开发工具（未完成）
   │      EditDialogForm.vb         # 新增与编辑稿件对话框
   │      InputDialogForm.vb        # 输入对话框
   │      MainForm.vb               # 主窗体
   │      PrintForm.vb              # 打印对话框
   │      PropertiesForm.vb         # 程序设置窗体
   │      ScriptForm.vb             # *保密*
   │      TextBoxForm.vb            # 文本框窗体
   │      ViewForm.vb               # 图片显示窗体
   │      
   ├─Meta                           # 元，存放一些好玩的东西
   │      Artifacts.vb              # 旧的函数类
   │      BV1GJ411x7h7.png          # RickRoll二维码
   │      
   ├─Resources                      # 资源文件夹
   │  │  FAS.ico                    # 程序图标
   │  │  FAS.png                    # 程序图标
   │  │  Icons.resx                 # 菜单与窗口图标
   │  │  Licenses.resx              # 许可证文本字符串
   │  │  Resources.en-US.resx       # 英文本地化文件
   │  │  Resources.resx             # 回退（Fallback）本地化文件（英文）
   │  │  Resources.zh-HanFurry.resx # 兽体中文本地化文件
   │  │  Resources.zh-Hans.resx     # 简体中文本地化文件
   │  │  Resources.zh-Hant.resx     # 繁体中文本地化文件
   │  │  
   │  ├─Darkmode-Icons              # 深色模式图标资源
   │  │      ...
   │  │      
   │  └─Lightmode-Icons             # 浅色模式图标资源
   │         ...
   │          
   └─Utilities                      # 工具类
           AfdianHandler.vb         # 爱发电赞助者列表检查类
           AppSettings.vb           # 程序设置管理类
           BasicFcn.vb              # 基本函数
           FileTransaction.vb       # 文件原子操作事务类
           ImageChecker.vb          # 图像有效性检查类（@LittleJiu-furry）
           Logger.vb                # 日志管理类（已实现，未使用）
           UpdateChecker.vb         # 更新检查工具类
           WinAPI.vb                # Windows API声明
```

当前版本已经更新至`1.0.10`，FAS也经历了多次代码重构与迭代，为了避免**未来的维护者难以读懂程序源码**，我将在这里进行部分讲解，以及我的部分优化手段：

## NuGet包

```text
FurryArtStudio
  ├─Chromis.1.0.3
  ├─Dapper.2.1.72
  ├─Krypton.Toolkit.100.26.1.19
  ├─Microsoft.Bcl.AsyncInterfaces.10.0.5
  ├─Microsoft.Bcl.HashCode.6.0.0
  ├─Ookii.Dialogs.WinForms.4.0.0
  ├─Stub.System.Data.SQLite.Core.NetFramework.1.0.119.0
  ├─System.Buffers.4.6.1  
  ├─System.Collections.Immutable.10.0.5
  ├─System.Data.SQLite.2.0.3
  ├─System.Data.SQLite.Core.1.0.119.0
  ├─System.Formats.Nrbf.10.0.5
  ├─System.IO.Pipelines.10.0.5
  ├─System.Memory.4.6.3
  ├─System.Numerics.Vectors.4.6.1
  ├─System.Reflection.Metadata.10.0.5
  ├─System.Resources.Extensions.10.0.5
  ├─System.Runtime.CompilerServices.Unsafe.6.1.2
  ├─System.Text.Encoding.CodePages.10.0.5
  ├─System.Text.Encodings.Web.10.0.5
  ├─System.Text.Json.10.0.5
  ├─System.Threading.Tasks.Extensions.4.6.3
  └─System.ValueTuple.4.6.2
```
这些包为FAS目前使用的工具，SQLite数据库相关的库**不应随意更新**，需要测试新版本兼容性后再应用在生产环境中。

其余工具均应该按照许可证分发。

另外还有一个NuGet包为`ScriptEngine`，暂时还在设计中，没有开源。

`Chromis`是一个我自己开发的工具，一个用来提取图片主色调的小型类库，感兴趣的可以点[这里](https://github.com/xionglongztz/Chromis)查看源代码或者点[这里](https://www.nuget.org/packages/Chromis)在NuGet上查看，如果发现问题也欢迎向我提出并维护。

## TODO清单

目前的FAS仍然具有很多问题，正在尽力维护和实现：

 - 没有教程库，需要设计一套特殊的教程库文件（一个特殊的稿件库，包含教程），在用户没有网络的前提下为FAS用户提供帮助，需要在程序**首次运行**时，或者在用户使用过程中检测到没有下载帮助教程库包文件，自动从CDN上下载zip文件。
 - 没有编写[在线文档](https://github.com/PawLaboratory/Docs)（Docs），大概需要一段时间编写，其中包括一些我在程序里新增的进阶用法，例如特殊组合键等。
 - 新增`CODE_OF_CONDUCT.md`，`CONTRIBUTING.md`，`SECURITY.md`等文件，保障未来合作规范，效率与安全。
 - 需要设计一个矢量图版本的LOGO。
 - 开源**更新检查服务器**与**爱发电赞助名单**服务器后端程序（RESTful API，使用Node.js构建），这是FAS几乎唯一的联网行为。
 - 对于**数据库优化**方向仍然存在问题，例如语句拼接，加载数据量等等
 - 设计一套**搜索处理库**，可以根据条件搜索符合条件的稿件。
 - 部分本地化仍然存在未完善的问题，需要完善，另需要**日语**，**韩语**，**俄语**版本的本地化以满足更多用户需求。
 - 以及各种各样的小BUG需要修复...

## 跨平台 & V2

目前暂时不考虑跨平台的可能性，由于程序架构比较复杂，可能需要一段时间维护，若当前版本已达维护生命周期（除.NET框架更新外足够稳定），则会归档FAS仓库并开启V2的研发。

实际V2开发进程取决于当前版本生命周期，项目热度等因素影响。

V2相比于当前版本，改进内容请看先前的[文章](关于Paw系列暂停开发与新项目的启动.md)，这里有PawLab的产品的最终的形态，也就是**跨平台，可自己部署，专业，可扩展的数字资产管理工具**。
