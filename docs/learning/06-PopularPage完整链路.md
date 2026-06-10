# 05 - PopularPage 完整链路

> 从 `main.dart` 到番剧卡片显示的完整调用链。
> 理解了这个链路，其他页面的模式基本一样。

## 完整链路图

```
┌─────────────────────────────────────────────────────────────┐
│                        启动阶段                              │
└─────────────────────────────────────────────────────────────┘

main.dart
    │  runApp(ModularApp(module: AppModule(), child: AppWidget()))
    │
    ├─ AppModule → IndexModule → "/" → InitPage
    │
    ├─ AppWidget → MaterialApp.router(routerConfig: Modular.routerConfig)
    │
    ▼
InitPage._initializeApp()
    │  11 个初始化步骤完成后
    │
    │  _startDefaultPage()
    │    Modular.to.navigate("/tab/popular/")
    │
    ▼
IndexPage → ScaffoldMenu → PageView → RouterOutlet
    │
    │  Modular 匹配路由 "/tab/popular/" → PopularModule → PopularPage
    │
    ▼

┌─────────────────────────────────────────────────────────────┐
│                        UI 渲染阶段                           │
└─────────────────────────────────────────────────────────────┘

PopularPage.initState()
    │
    │  final controller = Modular.get<PopularController>()
    │  ↑ 从 DI 容器获取单例（IndexModule.binds() 中注册的）
    │
    │  controller.queryBangumiByTrend()
    │
    ▼

┌─────────────────────────────────────────────────────────────┐
│                        数据请求阶段                          │
└─────────────────────────────────────────────────────────────┘

PopularController.queryBangumiByTrend()
    │
    │  isLoadingMore = true    → Observer 监听 → UI 显示进度条
    │
    │  BangumiApi.getBangumiTrendsList(offset: 0, limit: 24)
    │
    ▼
BangumiApi.getBangumiTrendsList()
    │
    │  构造参数: { type: 2, limit: 24, offset: 0 }
    │
    │  _client.get(url, queryParameters: params)
    │
    ▼
BangumiClient.get()
    │
    │  构造 Headers:
    │    - User-Agent: 随机 UA
    │    - Referer: 空
    │    - Authorization: Bearer Token（如果需要认证）
    │
    │  DioFactory.apiDio.get(url, options: headers)
    │
    ▼

┌─────────────────────────────────────────────────────────────┐
│                        网络请求阶段                          │
└─────────────────────────────────────────────────────────────┘

DioFactory.apiDio (Dio 实例)
    │
    ├─ BaseOptions
    │    - connectTimeout: 10s
    │    - receiveTimeout: 10s
    │    - headers: { user-agent, referer }
    │
    ├─ Interceptor: BangumiMirrorInterceptor
    │    │  检查是否启用镜像
    │    │  如果启用: api.bgm.tv → api.kazumi.fyi
    │    └─ handler.next(options)
    │
    ├─ BackgroundTransformer
    │    JSON 解析在后台线程执行
    │
    ▼
HTTP GET https://api.bgm.tv/p1/subjects/trending?type=2&limit=24&offset=0
    │
    │  或（镜像模式）
    │
HTTP GET https://api.kazumi.fyi/p1/subjects/trending?type=2&limit=24&offset=0
    │
    ▼
Bangumi 服务器返回 JSON

┌─────────────────────────────────────────────────────────────┐
│                        响应处理阶段                          │
└─────────────────────────────────────────────────────────────┘

JSON 响应
    │
    │  BackgroundTransformer 解析 JSON（后台线程）
    │
    ▼
BangumiClient.get()
    │
    │  return response.data  ← 返回 Map/List
    │
    ▼
BangumiApi.getBangumiTrendsList()
    │
    │  遍历 jsonData['data']
    │    BangumiItem.fromJson(jsonItem['subject'])
    │
    │  解析每个字段：
    │    id: json['id']
    │    name: json['name']
    │    nameCn: json['name_cn'] ?? json['nameCN'] ?? json['name']
    │    images: json['images']
    │    ratingScore: json['rating']['score']
    │    ...
    │
    │  return List<BangumiItem>
    │
    ▼
PopularController.queryBangumiByTrend()
    │
    │  trendList.addAll(result)  ← 更新 MobX 状态
    │
    │  isLoadingMore = false     → Observer 监听 → UI 隐藏进度条
    │
    ▼

┌─────────────────────────────────────────────────────────────┐
│                        UI 刷新阶段                           │
└─────────────────────────────────────────────────────────────┘

Observer 监听到 trendList 变化
    │
    ▼
PopularPage.build() 重新执行
    │
    │  Observer(builder: (_) {
    │    return contentGrid(popularController.trendList);
    │  })
    │
    ▼
contentGrid()
    │
    │  SliverGrid(
    │    gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
    │      crossAxisCount: 3/5/6,  ← 自适应列数
    │    ),
    │    delegate: SliverChildBuilderDelegate(
    │      (context, index) => BangumiCardV(bangumiItem: trendList[index]),
    │    ),
    │  )
    │
    ▼
BangumiCardV 渲染每个番剧卡片
    │
    ├─ 图片: CachedNetworkImage(bangumiItem.images['large'])
    │
    ├─ 标题: Text(bangumiItem.nameCn)
    │
    └─ 评分: Text(bangumiItem.ratingScore.toString())
```

## 数据流简图

```
用户打开应用
    ↓
InitPage 初始化完成后跳转
    ↓
PopularPage.initState()
    ↓
PopularController.queryBangumiByTrend()
    ↓
BangumiApi.getBangumiTrendsList()
    ↓
BangumiClient.get()
    ↓
DioFactory.apiDio.get()
    ↓
BangumiMirrorInterceptor (URL 重写)
    ↓
HTTP 请求 → Bangumi 服务器
    ↓
返回 JSON
    ↓
BangumiItem.fromJson() 解析
    ↓
trendList.addAll() 更新状态
    ↓
Observer 自动刷新 UI
    ↓
SliverGrid 渲染番剧卡片
    ↓
用户看到番剧列表
```

## 涉及的文件

| 文件 | 职责 | 注释状态 |
|------|------|---------|
| `lib/main.dart` | 入口，初始化基础设施 | ✅ |
| `lib/app_module.dart` | 路由根，转发给 IndexModule | ✅ |
| `lib/app_widget.dart` | UI 根，MaterialApp.router | ✅ |
| `lib/pages/index_module.dart` | 路由中枢，注册依赖 | ✅ |
| `lib/pages/router.dart` | Tab 路由注册表 | ✅ |
| `lib/pages/init_page.dart` | 启动页，初始化后跳转 | ✅ |
| `lib/pages/index_page.dart` | 导航壳 | ✅ |
| `lib/pages/menu/menu.dart` | 底部栏/侧边栏 | ✅ |
| `lib/pages/popular/popular_module.dart` | 推荐页模块 | ✅ |
| `lib/pages/popular/popular_page.dart` | 推荐页 UI | ✅ |
| `lib/pages/popular/popular_controller.dart` | MobX 状态管理 | ✅ |
| `lib/request/apis/bangumi_api.dart` | API 层 | ✅ |
| `lib/request/clients/bangumi_client.dart` | HTTP 客户端 | ✅ |
| `lib/request/core/dio_factory.dart` | Dio 实例工厂 | ✅ |
| `lib/modules/bangumi/bangumi_item.dart` | 数据模型 | ✅ |

## 架构分层

```
┌─────────────────────────────────────────────────────────────┐
│  UI 层                                                       │
│  PopularPage (Widget)                                        │
│    └─ Observer 监听 PopularController                        │
├─────────────────────────────────────────────────────────────┤
│  状态管理层                                                   │
│  PopularController (MobX Store)                              │
│    └─ @observable trendList, isLoadingMore                   │
├─────────────────────────────────────────────────────────────┤
│  API 层                                                      │
│  BangumiApi (纯静态方法)                                      │
│    └─ 调用 BangumiClient，解析 JSON                          │
├─────────────────────────────────────────────────────────────┤
│  网络层                                                      │
│  BangumiClient → DioFactory → Dio                           │
│    └─ 拦截器处理镜像、认证、签名                               │
├─────────────────────────────────────────────────────────────┤
│  数据模型层                                                   │
│  BangumiItem                                                 │
│    └─ fromJson 解析 JSON，Hive 持久化                        │
└─────────────────────────────────────────────────────────────┘
```

## 其他页面的通用模式

理解了 PopularPage 的链路，其他页面基本一样：

| 页面 | Controller | API 方法 | 数据模型 |
|------|------------|----------|----------|
| 推荐页 | PopularController | getBangumiTrendsList() | BangumiItem |
| 时间表 | TimelineController | getCalendar() | BangumiItem |
| 收藏页 | CollectController | getBangumiCollectibles() | BangumiCollection |
| 搜索页 | SearchController | bangumiSearch() | BangumiItem |
| 详情页 | InfoController | getBangumiInfoByID() | BangumiItem |
| 播放页 | VideoPageController | getBangumiEpisodesByID() | EpisodeInfo |

### 通用模式总结

```
Page (Widget)
    │  initState()
    │  Modular.get<XxxController>()
    │
    ▼
Controller (MobX Store)
    │  @observable 变量
    │  调用 XxxApi 方法
    │
    ▼
Api (纯静态方法)
    │  调用 Client.get/post
    │  解析 JSON → 数据模型
    │
    ▼
Client (HTTP 封装)
    │  DioFactory.xxxDio.get/post
    │  构造 Headers
    │
    ▼
Dio (HTTP 实例)
    │  拦截器处理
    │  发送请求
    │
    ▼
服务器返回 JSON
    │
    ▼
数据模型.fromJson() 解析
    │
    ▼
Controller 更新 @observable
    │
    ▼
Observer 自动刷新 UI
```

## 学习要点

1. **分层架构**：UI → Controller → API → Client → Dio → 服务器
2. **MobX 响应式**：@observable 变量变化 → Observer 自动刷新 UI
3. **依赖注入**：Modular.get<T>() 获取单例
4. **工厂模式**：DioFactory 创建不同用途的 Dio 实例
5. **拦截器模式**：镜像重写、认证、签名
6. **数据模型**：fromJson 解析 JSON，HiveType 持久化

## 下一步

掌握了这个链路后，可以继续学习：

- **播放器**：视频播放的特殊处理（media_kit、WebView、弹幕）
- **插件系统**：XPath 规则引擎、声明式爬虫
- **同步机制**：WebDAV、Bangumi 双向同步
- **下载管理**：M3U8 分片下载、广告过滤
