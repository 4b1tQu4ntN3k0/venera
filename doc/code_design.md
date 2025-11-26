# Venera 代码设计文档

## 概述

Venera 是一个使用 Flutter 框架开发的跨平台漫画阅读器应用，支持 Android、iOS、Windows、Linux 和 macOS。它能够阅读本地漫画，也支持通过 JavaScript 插件系统从各种网络源加载漫画。

## 技术栈

- **Flutter 3.35.5** - 跨平台 UI 框架
- **Dart** - 主要编程语言
- **QuickJS** - JavaScript 引擎（用于漫画源插件）
- **SQLite** - 本地数据存储

## 项目结构

```
lib/
├── main.dart           # 应用入口
├── init.dart           # 初始化逻辑
├── headless.dart       # 无头模式支持
├── components/         # UI 组件库
├── foundation/         # 核心基础设施
├── network/           # 网络层
├── pages/             # 页面/视图
└── utils/             # 工具类
```

## 核心架构模块

### 1. 应用核心 (`lib/foundation/`)

#### 1.1 App 单例 (`app.dart`)

`App` 是一个全局单例，提供应用级别的功能和状态管理：

```dart
class _App {
  final version = "1.5.3";
  
  // 平台检测
  bool get isAndroid => Platform.isAndroid;
  bool get isDesktop => Platform.isWindows || Platform.isLinux || Platform.isMacOS;
  
  // 路径管理
  late String dataPath;
  late String cachePath;
  
  // 导航管理
  final rootNavigatorKey = GlobalKey<NavigatorState>();
  
  // 核心管理器
  final Appdata data = appdata;
  final HistoryManager history = HistoryManager();
  final LocalFavoritesManager favorites = LocalFavoritesManager();
  final LocalManager local = LocalManager();
}
```

**设计要点：**
- 使用单例模式管理全局状态
- 提供平台检测功能，便于编写跨平台代码
- 集中管理应用数据路径
- 统一管理导航和各类数据管理器

#### 1.2 应用数据 (`appdata.dart`)

`Appdata` 类管理所有用户配置和应用状态：

```dart
class Appdata with Init {
  final Settings settings = Settings._create();
  var searchHistory = <String>[];
  var implicitData = <String, dynamic>{};
  
  Future<void> saveData([bool sync = true]) async { ... }
  void syncData(Map<String, dynamic> data) { ... }
}
```

**Settings 类的设计特点：**
- 使用 `ChangeNotifier` 实现响应式更新
- 支持默认值配置
- 支持漫画特定设置（`comicSpecificSettings`）
- 支持 WebDAV 数据同步

#### 1.3 JavaScript 引擎 (`js_engine.dart`)

Venera 使用 QuickJS 作为 JavaScript 运行时，实现漫画源插件系统：

```dart
class JsEngine with _JSEngineApi, JsUiApi, Init {
  FlutterQjs? _engine;
  
  // 消息处理器 - 处理 JS 和 Dart 之间的通信
  Object? _messageReceiver(dynamic message) {
    switch (method) {
      case "log": // 日志
      case "http": // 网络请求
      case "html": // HTML 解析
      case "convert": // 数据转换
      case "cookie": // Cookie 管理
      case "UI": // UI 交互
      // ...
    }
  }
}
```

**JS 引擎设计要点：**
- 使用消息传递机制实现 JS ↔ Dart 双向通信
- 提供丰富的内置 API（网络请求、HTML 解析、加密解密等）
- 支持异步操作
- 使用 `JSPool` 进行多线程 JS 执行

### 2. 漫画源系统 (`lib/foundation/comic_source/`)

这是 Venera 最核心的架构设计之一：

#### 2.1 漫画源管理器 (`comic_source.dart`)

```dart
class ComicSourceManager with ChangeNotifier, Init {
  final List<ComicSource> _sources = [];
  
  // 从文件系统加载所有 JS 漫画源
  Future<void> doInit() async {
    final path = "${App.dataPath}/comic_source";
    await for (var entity in Directory(path).list()) {
      if (entity is File && entity.path.endsWith(".js")) {
        var source = await ComicSourceParser()
            .parse(await entity.readAsString(), entity.absolute.path);
        _sources.add(source);
      }
    }
  }
}
```

#### 2.2 ComicSource 类

每个漫画源包含以下核心功能模块：

```dart
class ComicSource {
  final String name;           // 源名称
  final String key;            // 唯一标识
  
  // 功能模块
  final AccountConfig? account;              // 账号系统
  final CategoryData? categoryData;          // 分类数据
  final FavoriteData? favoriteData;          // 收藏功能
  final List<ExplorePageData> explorePages;  // 探索页面
  final SearchPageData? searchPageData;      // 搜索功能
  
  // 核心方法
  final LoadComicFunc? loadComicInfo;        // 加载漫画信息
  final LoadComicPagesFunc? loadComicPages;  // 加载漫画页面
  final CommentsLoader? commentsLoader;      // 加载评论
  // ...
}
```

#### 2.3 漫画源插件架构

```
JavaScript 漫画源
       ↓
ComicSourceParser (解析 JS 文件)
       ↓
ComicSource (Dart 对象)
       ↓
JsEngine (执行 JS 代码)
       ↓
应用功能（搜索、浏览、阅读等）
```

### 3. 页面架构 (`lib/pages/`)

#### 3.1 主页面结构

```dart
class MainPage extends StatefulWidget {
  final _pages = [
    const HomePage(),           // 主页
    const FavoritesPage(),      // 收藏
    const ExplorePage(),        // 探索
    const CategoriesPage(),     // 分类
  ];
  
  // 使用自定义导航栏组件
  return NaviPane(
    paneItems: [...],
    pageBuilder: (index) => _pages[index],
  );
}
```

#### 3.2 阅读器架构 (`lib/pages/reader/`)

```
reader/
├── reader.dart        # 阅读器主逻辑
├── scaffold.dart      # 阅读器脚手架/布局
├── images.dart        # 图片列表管理
├── comic_image.dart   # 单个漫画图片组件
├── gesture.dart       # 手势处理
├── chapters.dart      # 章节管理
└── loading.dart       # 加载状态
```

### 4. 组件库 (`lib/components/`)

Venera 有一套完整的自定义组件库：

| 组件文件 | 功能 |
|---------|------|
| `appbar.dart` | 自定义应用栏 |
| `comic.dart` | 漫画卡片组件 |
| `image.dart` | 图片加载组件 |
| `navigation_bar.dart` | 导航栏 |
| `menu.dart` | 菜单组件 |
| `flyout.dart` | 弹出菜单 |
| `select.dart` | 选择器 |
| `loading.dart` | 加载指示器 |
| `gesture.dart` | 手势处理 |

### 5. 网络层 (`lib/network/`)

```
network/
├── app_dio.dart       # Dio HTTP 客户端封装
├── cache.dart         # 网络缓存
├── cookie_jar.dart    # Cookie 管理（SQLite）
├── download.dart      # 下载管理
├── file_downloader.dart # 文件下载器
├── images.dart        # 图片加载
├── proxy.dart         # 代理设置
└── cloudflare.dart    # Cloudflare 处理
```

### 6. 工具类 (`lib/utils/`)

提供各种辅助功能：

| 工具文件 | 功能 |
|---------|------|
| `translations.dart` | 多语言翻译 |
| `tags_translation.dart` | 标签翻译 |
| `data_sync.dart` | 数据同步（WebDAV） |
| `import_comic.dart` | 漫画导入 |
| `cbz.dart` / `epub.dart` / `pdf.dart` | 各种格式支持 |
| `opencc.dart` | 简繁转换 |

## 设计模式与最佳实践

### 1. 单例模式

全局服务使用单例模式：

```dart
// 使用工厂构造函数实现单例
class ComicSourceManager {
  static ComicSourceManager? _instance;
  
  factory ComicSourceManager() => _instance ??= ComicSourceManager._create();
  
  ComicSourceManager._create();
}
```

### 2. 初始化模式 (Init Mixin)

统一的异步初始化模式：

```dart
mixin Init {
  Future<void> doInit();  // 子类实现
  Future<void> init() async {
    // 统一的初始化逻辑
    await doInit();
  }
}
```

### 3. 响应式状态管理

使用 `ChangeNotifier` 实现响应式更新：

```dart
class Settings with ChangeNotifier {
  operator []=(String key, dynamic value) {
    _data[key] = value;
    notifyListeners();  // 通知监听者
  }
}
```

### 4. 插件化架构

通过 JavaScript 实现可扩展的漫画源系统：

```javascript
class NewComicSource extends ComicSource {
    name = "源名称"
    key = "unique_key"
    
    // 实现各种功能
    search = { load: async (keyword, options, page) => { ... } }
    comic = { loadInfo: async (id) => { ... } }
    // ...
}
```

## 数据流

### 漫画加载流程

```
1. 用户选择漫画
       ↓
2. ComicSource.loadComicInfo(id)
       ↓
3. JsEngine 执行 JavaScript
       ↓
4. 网络请求获取数据
       ↓
5. 解析返回 ComicDetails
       ↓
6. 渲染漫画详情页
```

### 图片加载流程

```
1. 阅读器请求图片
       ↓
2. ComicSource.loadComicPages(comicId, epId)
       ↓
3. 返回图片 URL 列表
       ↓
4. ImageProvider 按需加载图片
       ↓
5. 缓存管理器缓存图片
       ↓
6. 渲染到屏幕
```

## 初始化流程

应用启动时的初始化顺序（`init.dart`）：

```dart
Future<void> init() async {
  await App.init();                    // 1. 初始化路径
  await SingleInstanceCookieJar.createInstance();  // 2. Cookie 管理
  
  await Future.wait([
    Rhttp.init(),                      // 3. 网络库
    App.initComponents(),              // 4. 数据管理器
    AppTranslation.init(),             // 5. 翻译
    TagsTranslation.readData(),        // 6. 标签翻译
    JsEngine().init(),                 // 7. JS 引擎
    ComicSourceManager().init(),       // 8. 漫画源
    OpenCC.init(),                     // 9. 简繁转换
  ]);
  
  CacheManager().setLimitSize(...);    // 10. 缓存配置
}
```

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `lib/main.dart` | 应用入口、主题配置 |
| `lib/init.dart` | 初始化流程 |
| `lib/foundation/app.dart` | 全局应用状态 |
| `lib/foundation/appdata.dart` | 用户配置管理 |
| `lib/foundation/js_engine.dart` | JavaScript 执行引擎 |
| `lib/foundation/comic_source/comic_source.dart` | 漫画源核心定义 |
| `lib/foundation/comic_source/parser.dart` | JS 漫画源解析器 |
| `lib/pages/main_page.dart` | 主界面布局 |
| `lib/pages/reader/reader.dart` | 阅读器核心 |
| `lib/network/app_dio.dart` | 网络请求封装 |

## 扩展开发指南

### 添加新的漫画源

1. 创建 JavaScript 文件，继承 `ComicSource` 类
2. 实现必要的方法（search、loadInfo、loadEp 等）
3. 将文件放入 `${App.dataPath}/comic_source/` 目录
4. 重启应用或在设置中重新加载漫画源

详细文档参见：[Comic Source 文档](./comic_source.md) 和 [JavaScript API 文档](./js_api.md)

### 添加新页面

1. 在 `lib/pages/` 创建新的页面文件
2. 使用 `components/` 中的组件构建 UI
3. 通过导航系统（`App.rootContext.to()`）进行页面跳转

### 添加新的网络功能

1. 在 `lib/network/` 中创建新的模块
2. 使用 `AppDio` 进行网络请求
3. 通过 `CookieManagerSql` 管理 Cookie

## 总结

Venera 的架构设计具有以下特点：

1. **模块化设计** - 清晰的职责分离，便于维护和扩展
2. **插件化架构** - 通过 JavaScript 实现灵活的漫画源系统
3. **跨平台支持** - 统一的代码库支持多个平台
4. **响应式状态管理** - 使用 ChangeNotifier 实现 UI 更新
5. **异步初始化** - 优化应用启动性能
6. **完善的组件库** - 统一的 UI 组件风格

这种架构设计使得 Venera 具有良好的可维护性和可扩展性，开发者可以轻松添加新功能或新的漫画源。
