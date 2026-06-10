# 第5章 网络请求与 API 层

> request 目录封装了所有外部 API 调用。
> DioFactory 统一配置 4 个 HTTP 客户端实例，
> 4 个 Client 分别对接不同的服务（Bangumi、弹弹Play、插件站点、下载），
> api_endpoints.dart 集中管理所有 API 端点地址。

---

## 5.1 网络层架构

```
┌─────────────────────────────────────────────────────────────────┐
│  API 层（lib/request/apis/）                                     │
│  纯业务逻辑：构造参数 → 调用 Client → 解析 JSON → 返回数据模型    │
│                                                                 │
│  BangumiApi      → BangumiClient      → apiDio                  │
│  DanmakuApi      → DanmakuClient      → apiDio                  │
│  PluginCatalogApi → GithubClient       → githubDio               │
│  BangumiApi (爬虫) → PluginSiteClient  → pluginDio               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Client 层（lib/request/clients/）                               │
│  封装 HTTP 细节：Headers、认证、签名、错误处理                     │
│                                                                 │
│  BangumiClient      → GET/POST + Bearer Token + HMAC 签名       │
│  DanmakuClient      → GET + HMAC 签名                           │
│  PluginSiteClient   → getText/postFormText + 随机 UA             │
│  GithubClient       → getJson/getText + Accept Header            │
│  DownloadHttpClient → getStream/getPlain/download               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  基础设施层（lib/request/core/）                                  │
│  Dio 实例管理、代理、拦截器、错误映射                              │
│                                                                 │
│  DioFactory         → 4 个 Dio 实例（懒加载）                    │
│  NetworkConfig      → 超时、代理、证书配置                        │
│  NetworkErrorMapper → DioException → NetworkException            │
│  NetworkException   → 领域异常（类型化错误）                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  配置层（lib/request/config/）                                   │
│                                                                 │
│  api_endpoints.dart → 所有 API 端点地址集中管理                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.2 核心文件清单

### API 层（业务逻辑）

| 文件 | 职责 |
|------|------|
| `request/apis/bangumi_api.dart` | Bangumi API 封装（番剧信息、搜索、收藏、评论） |
| `request/apis/danmaku_api.dart` | 弹弹Play API 封装（弹幕获取） |
| `request/apis/plugin_catalog_api.dart` | 规则仓库 API 封装（GitHub） |

### Client 层（HTTP 封装）

| 文件 | 职责 |
|------|------|
| `request/clients/bangumi_client.dart` | Bangumi HTTP 客户端（认证 + 镜像签名） |
| `request/clients/danmaku_client.dart` | 弹弹Play HTTP 客户端（HMAC 签名） |
| `request/clients/plugin_site_client.dart` | 插件站点 HTTP 客户端（爬虫专用） |
| `request/clients/github_client.dart` | GitHub HTTP 客户端（检查更新） |
| `request/clients/download_http_client.dart` | 下载 HTTP 客户端（流式下载） |

### 基础设施层

| 文件 | 职责 |
|------|------|
| `request/core/dio_factory.dart` | Dio 实例工厂（4 个实例 + 镜像拦截器） |
| `request/core/network_config.dart` | 网络配置（超时、代理、证书） |
| `request/core/network_error_mapper.dart` | 错误映射（DioException → NetworkException） |
| `request/core/network_exception.dart` | 领域异常定义 |
| `request/core/dio_logger_interceptor.dart` | 日志拦截器 |

### 配置层

| 文件 | 职责 |
|------|------|
| `request/config/api_endpoints.dart` | 所有 API 端点地址集中管理 |

---

## 5.3 ★ DioFactory — 4 个 Dio 实例

> 源码：`lib/request/core/dio_factory.dart`

```dart
class DioFactory {
    // 4 个 Dio 实例（懒加载，首次访问时创建）
    static Dio? _apiDio;      // Bangumi API
    static Dio? _githubDio;   // GitHub API
    static Dio? _pluginDio;   // 插件爬虫
    static Dio? _downloadDio; // 视频下载
}
```

### 为什么需要 4 个实例？

| 实例 | 用途 | 超时 | 拦截器 | 特殊配置 |
|------|------|------|--------|----------|
| `apiDio` | Bangumi API | 12s | BangumiMirrorInterceptor | 空 Referer |
| `githubDio` | GitHub API | 12s | GithubMirrorInterceptor | Accept Header |
| `pluginDio` | 插件爬虫 | 12s | 无 | 随机 Accept-Language |
| `downloadDio` | 视频下载 | 15s/30s | 无 | 短超时 |

### 实例创建

```dart
static Dio get apiDio => _apiDio ??= _create(
    NetworkConfig.fromSettings(),           // 从设置读取代理配置
    defaultHeaders: {
        'referer': '',                       // Bangumi API 不需要 Referer
        'user-agent': getRandomUA(),         // 随机 User-Agent
    },
    interceptors: [_BangumiMirrorInterceptor()],  // 镜像拦截器
);

static Dio get pluginDio => _pluginDio ??= _create(
    NetworkConfig.fromSettings(),
    defaultHeaders: {
        'user-agent': getRandomUA(),
        'accept-language': getRandomAcceptedLanguage(),  // 随机语言
    },
);

static Dio get downloadDio => _downloadDio ??= _create(
    NetworkConfig.fromSettings(
        connectTimeout: Duration(seconds: 15),
        receiveTimeout: Duration(seconds: 30),  // 下载需要更长超时
    ),
    defaultHeaders: { 'user-agent': getRandomUA() },
);
```

### 核心创建方法

```dart
static Dio _create(NetworkConfig config, {...}) {
    final dio = Dio(BaseOptions(
        connectTimeout: config.connectTimeout,
        receiveTimeout: config.receiveTimeout,
        headers: defaultHeaders,
        validateStatus: (status) => status != null && status >= 200 && status < 300,
    ));
    dio.httpClientAdapter = config.createAdapter();  // 代理配置
    dio.transformer = BackgroundTransformer();        // JSON 后台线程解析
    dio.interceptors.addAll(interceptors);            // 添加拦截器
    return dio;
}
```

---

## 5.4 镜像拦截器

### BangumiMirrorInterceptor

```dart
class _BangumiMirrorInterceptor extends Interceptor {
    @override
    void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
        // 1. 检查是否启用镜像
        if (!GStorage.getSetting(SettingsKeys.enableBangumiProxy)) {
            handler.next(options);  // 未启用，直接放行
            return;
        }

        // 2. 检查是否是可镜像的域名
        if (!_mirrorableHosts.contains(uri.host)) {
            handler.next(options);  // 不是，直接放行
            return;
        }

        // 3. 重写 URL：api.bgm.tv → api.kazumi.fyi
        options.path = 'https://api.kazumi.fyi' + uri.path + '?' + uri.query;
        handler.next(options);
    }
}
```

**可镜像的域名**：`api.bgm.tv`、`next.bgm.tv`

**不支持镜像的端点**（涉及用户隐私）：`/v0/me`（用户信息）、`/v0/users/-/collections`（收藏同步）

### GithubMirrorInterceptor

```dart
class _GithubMirrorInterceptor extends Interceptor {
    @override
    void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
        // 1. 检查是否启用 Git 镜像
        if (!GStorage.getSetting(SettingsKeys.enableGitProxy)) {
            handler.next(options);
            return;
        }

        // 2. 重写 URL：在原始 URL 前加上镜像地址
        options.path = 'https://ghfast.top/' + uri.toString();
        handler.next(options);
    }
}
```

**可镜像的域名**：`api.github.com`、`github.com`、`raw.githubusercontent.com` 等

---

## 5.5 Client 层详解

### BangumiClient — Bangumi HTTP 客户端

> 源码：`lib/request/clients/bangumi_client.dart`

```dart
class BangumiClient {
    static final BangumiClient instance = BangumiClient._();  // 单例

    Future<dynamic> get(String url, {bool requiresAuth = false, ...}) async {
        final response = await DioFactory.apiDio.get(url,
            options: Options(headers: _headers(requiresAuth: requiresAuth, ...)),
        );
        return response.data;
    }

    Map<String, dynamic> _headers({...}) {
        final headers = <String, dynamic>{...bangumiHTTPHeader};

        // 认证：Bearer Token
        if ((requiresAuth || bangumiSyncEnable) && token.isNotEmpty) {
            headers['Authorization'] = 'Bearer $token';
        }

        // 镜像签名：HMAC（保护端点）
        if (_shouldSignProtectedMirrorRequest(url, method)) {
            headers['X-AppId'] = bangumiMirrorCredentials['id'];
            headers['X-Timestamp'] = timestamp;
            headers['X-Signature'] = generateBangumiMirrorSearchSignature(...);
        }

        return headers;
    }
}
```

### DanmakuClient — 弹弹Play HTTP 客户端

> 源码：`lib/request/clients/danmaku_client.dart`

```dart
class DanmakuClient {
    Future<dynamic> get(String url, {...}) async {
        final requestHeaders = <String, dynamic>{
            'user-agent': getRandomUA(),
            'X-Auth': 1,
            'X-AppId': dandanCredentials['id'],           // 弹弹Play AppID
            'X-Timestamp': timestamp,
            'X-Signature': generateDandanSignature(...),   // HMAC 签名
        };
        final response = await DioFactory.apiDio.get(url, options: Options(headers: requestHeaders));
        return response.data;
    }
}
```

**所有弹弹Play API 都需要 HMAC 签名认证。**

### PluginSiteClient — 插件站点 HTTP 客户端

> 源码：`lib/request/clients/plugin_site_client.dart`

```dart
class PluginSiteClient {
    // 获取网页 HTML（GET）
    Future<String> getText(String url, {Map<String, dynamic> headers = const {}}) async {
        final response = await DioFactory.pluginDio.get<String>(url,
            options: Options(responseType: ResponseType.plain, headers: _headers(headers)),
        );
        return response.data ?? '';
    }

    // 提交表单（POST）
    Future<String> postFormText(String url, {Object? data, ...}) async {
        final response = await DioFactory.pluginDio.post<String>(url,
            data: data,
            options: Options(responseType: ResponseType.plain, headers: _headers({...})),
        );
        return response.data ?? '';
    }

    // 默认 Headers：随机 UA + 随机语言
    Map<String, dynamic> _headers(Map<String, dynamic> headers) {
        return {
            'user-agent': getRandomUA(),
            'Accept-Language': getRandomAcceptedLanguage(),
            'Connection': 'keep-alive',
            ...headers,
        };
    }
}
```

**返回 HTML 字符串（不是 JSON），由 Plugin 类用 XPath 解析。**

### DownloadHttpClient — 下载 HTTP 客户端

> 源码：`lib/request/clients/download_http_client.dart`

```dart
class DownloadHttpClient {
    // 流式获取（用于直接文件下载）
    Future<Response<ResponseBody>> getStream(String url, ...) async {
        return await DioFactory.downloadDio.get<ResponseBody>(url,
            options: Options(responseType: ResponseType.stream, ...),
        );
    }

    // 获取纯文本（用于 M3U8 内容）
    Future<String> getPlain(String url, ...) async {
        final response = await DioFactory.downloadDio.get<String>(url,
            options: Options(responseType: ResponseType.plain, ...),
        );
        return response.data ?? '';
    }

    // 下载文件到本地
    Future<void> download(String url, String savePath, ...) async {
        await DioFactory.downloadDio.download(url, savePath, ...);
    }
}
```

### GithubClient — GitHub HTTP 客户端

> 源码：`lib/request/clients/github_client.dart`

```dart
class GithubClient {
    // 获取最新 Release 信息
    Future<Map<String, dynamic>> latestRelease() async {
        final data = await getJson(ApiEndpoints.latestApp);
        return Map<String, dynamic>.from(data);
    }

    // 获取最新版本号
    Future<String> latestVersion() async {
        final data = await latestRelease();
        return data['tag_name']?.toString() ?? ApiEndpoints.version;
    }
}
```

---

## 5.6 NetworkConfig — 网络配置

> 源码：`lib/request/core/network_config.dart`

```dart
class NetworkConfig {
    final Duration connectTimeout;    // 连接超时（默认 12s）
    final Duration receiveTimeout;    // 接收超时（默认 12s）
    final Duration? sendTimeout;      // 发送超时
    final String? proxyHost;          // 代理主机
    final int? proxyPort;             // 代理端口
    final bool allowBadCertificates;  // 允许自签名证书
    final bool enableLog;             // 启用日志

    // 从用户设置读取代理配置
    static NetworkConfig fromSettings({...}) {
        final proxyEnable = GStorage.getSetting(SettingsKeys.proxyEnable);
        if (proxyEnable) {
            final proxyUrl = GStorage.getSetting(SettingsKeys.proxyUrl);
            final parsed = ProxyUtils.parseProxyUrl(proxyUrl);
            return NetworkConfig(proxyHost: parsed.$1, proxyPort: parsed.$2, ...);
        }
        return NetworkConfig(...);
    }

    // 创建 HTTP 客户端适配器（配置代理）
    IOHttpClientAdapter createAdapter() {
        return IOHttpClientAdapter(createHttpClient: () {
            final client = HttpClient();
            if (hasProxy) {
                client.findProxy = (_) => 'PROXY $proxyHost:$proxyPort';
            }
            return client;
        });
    }
}
```

---

## 5.7 NetworkErrorMapper — 错误映射

> 源码：`lib/request/core/network_error_mapper.dart`

```dart
class NetworkErrorMapper {
    static Future<NetworkException> mapException(DioException error) async {
        switch (error.type) {
            case DioExceptionType.badCertificate:
                return NetworkException(type: badCertificate, message: '证书有误！');
            case DioExceptionType.badResponse:
                return NetworkException(type: badResponse, message: '服务器异常');
            case DioExceptionType.cancel:
                return NetworkException(type: cancel, message: '请求已被取消');
            case DioExceptionType.connectionError:
                return NetworkException(type: connectionError, message: '连接错误');
            case DioExceptionType.connectionTimeout:
                return NetworkException(type: connectionTimeout, message: '连接超时');
            case DioExceptionType.receiveTimeout:
                return NetworkException(type: receiveTimeout, message: '响应超时');
            case DioExceptionType.sendTimeout:
                return NetworkException(type: sendTimeout, message: '发送超时');
            case DioExceptionType.unknown:
                final connection = await _connectionLabel();  // 检测网络类型
                return NetworkException(type: unknown, message: '$connection 网络异常');
        }
    }
}
```

**将 Dio 的底层异常转换为领域异常，UI 层只处理 NetworkException。**

---

## 5.8 API 端点集中管理

> 源码：`lib/request/config/api_endpoints.dart`

```dart
class ApiEndpoints {
    // 域名
    static const String bangumiAPIDomain = 'https://api.bgm.tv';
    static const String bangumiAPINextDomain = 'https://next.bgm.tv';
    static const String bangumiMirrorDomain = 'https://api.kazumi.fyi';
    static const String dandanAPIDomain = 'https://api.dandanplay.net';
    static const String pluginShop = 'https://raw.githubusercontent.com/Predidit/KazumiRules/main/';

    // Bangumi API 路径
    static const String bangumiRankSearch = '/v0/search/subjects?limit={0}&offset={1}';
    static const String bangumiInfoByID = '/v0/subjects/{0}';
    static const String bangumiCalendar = '/p1/calendar';
    static const String bangumiTrendsNext = '/p1/trending/subjects';

    // 弹弹Play API 路径
    static const String dandanAPIComment = '/api/v2/comment/';
    static const String dandanAPISearch = '/api/v2/search/anime';

    // URL 参数替换工具
    static String formatUrl(String url, List<dynamic> params) {
        for (int i = 0; i < params.length; i++) {
            url = url.replaceAll('{$i}', params[i].toString());
        }
        return url;
    }
}
```

---

## 5.9 完整请求链路示例

### 搜索番剧

```
BangumiApi.getBangumiList(tag: "日本动画")
    ↓ 构造参数
BangumiClient.post(url, data: params)
    ↓ 构造 Headers（User-Agent + 镜像签名）
DioFactory.apiDio.post(url, data: params, options: Options(headers: ...))
    ↓ BangumiMirrorInterceptor
    → api.bgm.tv → api.kazumi.fyi（如果启用镜像）
    ↓ HTTP 请求
Bangumi 服务器 → JSON 响应
    ↓
BangumiClient 返回 response.data（Map/List）
    ↓
BangumiApi 解析 JSON → List<BangumiItem>
```

### 弹幕获取

```
DanmakuApi.getDanDanmaku(bangumiID, episode)
    ↓ 构造 URL
DanmakuClient.get(url)
    ↓ 构造 Headers（User-Agent + HMAC 签名）
DioFactory.apiDio.get(url, options: Options(headers: ...))
    ↓ HTTP 请求
弹弹Play 服务器 → JSON 响应
    ↓
DanmakuApi 解析 JSON → List<DanmakuEntry>
```

### 插件爬虫

```
Plugin.queryBangumi("进击的巨人")
    ↓ 构造搜索 URL
PluginSiteClient.getText(url)
    ↓ 构造 Headers（随机 UA + 随机语言）
DioFactory.pluginDio.get<String>(url, responseType: ResponseType.plain)
    ↓ HTTP 请求
动漫网站 → HTML 响应
    ↓
PluginSiteClient 返回 HTML 字符串
    ↓
Plugin 用 XPath 解析 → PluginSearchResponse
```

---

## 5.10 请求头随机化

> 源码：`lib/utils/http_headers.dart`

```dart
// 随机 User-Agent（模拟不同浏览器）
String getRandomUA() { ... }

// 随机 Accept-Language（模拟不同地区用户）
String getRandomAcceptedLanguage() { ... }
```

**用途：防止被目标网站识别为爬虫。每次请求使用不同的 UA 和语言。**

---

## 5.11 与 Java/Vue 项目的对比

| 概念 | Kazumi (Flutter) | Spring Boot | Vue (Axios) |
|------|------------------|-------------|-------------|
| HTTP 客户端 | Dio | RestTemplate/WebClient | Axios |
| 实例管理 | DioFactory（4 个实例） | @Bean 配置 | Axios.create() |
| 拦截器 | Interceptor 类 | HandlerInterceptor | Axios.interceptors |
| 代理 | IOHttpClientAdapter | JVM 代理参数 | proxy 配置 |
| 错误处理 | NetworkErrorMapper | @ExceptionHandler | 响应拦截器 |
| 镜像 | URL 重写拦截器 | 无 | 无 |
| 端点管理 | api_endpoints.dart（常量） | @Value / 配置文件 | .env / 常量 |

---

## 下一步

网络请求与 API 层到此结束。下一步：

- **PopularPage 完整链路**：从搜索请求到 UI 渲染的完整过程
