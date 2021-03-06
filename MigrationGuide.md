YTKNetwork 2.0 迁移指南
======================

YTKNetwork 2.0 所依赖的 AFNetworking 版本从 2.X 变为 3.X 版本，抛弃了旧有的以 `AFHTTPRequestOperation` 为核心的 API，采用新的基于 `NSURLSession` 的 API。本指南的目的在于帮助使用 YTKNetwork 1.X 版本的应用迁移到新的 API。

## AFHTTPRequestOperation 完全被移除

在 iOS 7 上苹果引入了 `NSURLSession` 系列 API，旨在替代 `NSURLConnection` 系列 API。在 Xcode 7 中，`NSURLConnection` API 已经正式标记为废弃的（deprecated）。AFNetworking 3 当中也放弃了基于 `NSOperation` 的请求方式，转而采用基于 `NSURLSessionTask`。因此 `YTKRequest` 中的下列属性发生了变化：

#### YTKNetwork 1.X

```Objective-C
@property (nonatomic, strong) AFHTTPRequestOperation *requestOperation;
@property (nonatomic, strong, readonly, nullable) NSError *requestOperationError;
```

#### YTKNetwork 2.X

```Objective-C
@property (nonatomic, strong, readonly) NSURLSessionTask *requestTask;
@property (nonatomic, strong, readonly, nullable) NSError *error;
```

同时，原来依赖于 `AFHTTPRequestOperation` 的这些属性，需要使用新加入的替代 API 获取：

* `request.requestOperation.response` --> `request.response`
* `request.requestOperation.request` --> `request.currentRequest` & `request.originalRequest`

由于失去了 Operation 封装类，`request.currentRequest` 和 `request.originalRequest` 属性需要在 request 进行 `start` 之后才能获取，否则为 nil。

## 响应序列化选项

YTKNetwork 2.0 中加入了新的响应序列化选项，以及对应的 `responseObject` 属性，不同的序列化选项会导致响应返回不同类型的 `responseObject`，具体对应如下：

```Objective-C
typedef NS_ENUM(NSInteger, YTKResponseSerializerType) {
    YTKResponseSerializerTypeHTTP = 0,  /// NSData
    YTKResponseSerializerTypeJSON,      /// JSON object
    YTKResponseSerializerTypeXMLParser  /// NSXMLParser
};
```

默认的序列化选项是 `YTKResponseSerializerTypeJSON`。

需要注意的是，YTKNetwork 1.0 没有对响应序列化进行控制，并且主动忽略了 AFN 返回的响应序列化错误。AFN 默认进行的是 JSON 序列化，而在 YTKNetwork 1.0 中即使序列化结果是错误的（例如 Content-Type 不正确，或者返回的是二进制数据），仍然会认为请求是成功的，并且调用 YTKRequest 的成功回调。这一行为在 YTKNetwork 2.0 当中被修正了。如果发现在升级 YTKNetwork 2.0 之后某些之前正常的请求报错，请检查是否是由于响应的类型不正确。

## URL 拼接

YTKNetwork 2.0 中 `baseUrl` 和 `requestUrl` 的拼接实现发生了变化。在 1.X 中这两个 URL 是作为字符串直接拼接起来的，而在 2.0 中改为使用 `[NSURL URLWithString:relativeToURL]` 方法进行拼接，以提升可靠性和兼容性。

这一实现可能导致和之前不同的 URL 拼接结果。当 `requestUrl` 带有 `/` 前缀，同时 `baseUrl` 包含除 Host 之外的内容， `requestUrl` 仍然会被拼接到最顶级，这一差距来源于 `NSURL` 本身的实现：

```Objective-C
NSURL *baseURL = [NSURL URLWithString:@"http://example.com/v1/"];
NSURL *resultURL = [NSURL URLWithString:@"/foo" relativeToURL:baseURL];
// resultURL : http://example.com/foo
```

因此请注意如果 `baseUrl` 包含 Path 那么 `requestUrl` 的前缀不要在加入 `/` ，以免产生错误的的拼接结果。

## 下载请求

原来基于 `AFDownloadRequestOperation` 的下载请求改为使用系统自己的 `NSURLSessionDownloadTask`。当 `YTKRequest` 的 `resumableDownloadPath` 属性不为 nil 的情况下，会调用 `NSURLSessionDownloadTask` 进行下载，下载完成后文件会自动保存到给出的路径，无需再进行存储操作。

对于下载请求来说，响应属性的获取行为如下：

* `responseData`：不能获取
* `responseString`：不能获取
* `responseObject`：为 NSURL，是下载文件在本地所存储的路径。

如果在使用 YTKNetwork 1.X 的情况下，采用存储 `responseData` 的方式进行下载，那么无需进行改动。不过对于下载完整文件的情况，建议迁移到新的采用 `NSURLSessionDownloadTask` 的 API。

同时，获取下载进度的 API 也发生了变化，旧的 block 类型被抛弃，新的 block 类型取自 AFN 3.X：

```Objective-C
typedef void (^AFURLSessionTaskProgressBlock)(NSProgress *);
```

可以通过 `NSProgress` 的 `totalUnitCount` 和 `completedUnitCount` 获取下载进度有关的信息。

## YTKNetworkPrivate 不再暴露

`YTKNetworkPrivate.h` 将会成为私有头文件，所有依赖于此头文件的方法将不再可用。

## Cache API 更新

`YTKRequest` 类当中的 Cache 有关接口发生改变，不发送请求的情况下获取 Cache 的下列接口被去除：

* `- (id)cacheJson`
* `- (BOOL)isCacheVersionExpired;`

新的替代接口为：

* `- (BOOL)loadCacheWithError:(NSError **)error`

这个接口可以用于在不发送请求的情况下，直接读取磁盘缓存，返回值表示获取成功与否，如果获取失败，error 会返回错误的具体信息。读取缓存成功后，可以直接通过 `responseObject`，`responseData` 等属性获取数据。

用于将一个请求的响应写到另一个请求的缓存中的接口，也发生了变化：

#### YTKNetwork 1.X

`- (void)saveJsonResponseToCacheFile:(id)jsonResponse`

#### YTKNetwork 2.X

`- (void)saveResponseDataToCacheFile:(NSData *)data`

YTKNetwork 2.0 中加入了用于控制是否进行异步写缓存的接口：

`- (BOOL)writeCacheAsynchronously`

默认返回 `YES`，即使用异步方式写缓存，以提高性能。如果需要关闭此功能，可以在子类中覆盖这个方法并返回 `NO`。

## 响应前向处理

与 `- (void)requestCompleteFilter` 和 `- (void)requestFailedFilter` 对应， YTKNetwork 2.0 中加入了用于在响应结束，但是切换回主线程之前执行操作的函数 `- (void)requestCompletePreprocessor` 和 `- (void)requestFailedPreprocessor`，在这里执行的操作，可以避免阻塞主线程。

## 命名变更

YTKNetwork 2.0 中部分函数命名发生了变更：

* `[YTKNetworkAgent sharedInstance]` -> `[YTKNetworkAgent sharedAgent]`
* `[YTKChainRequestAgent sharedInstance]` -> `[YTKChainRequestAgent sharedAgent]`
* `[YTKBatchRequestAgent sharedInstance]` -> `[YTKBatchRequestAgent sharedAgent]`
* `[YTKNetworkConfig sharedInstance]` -> `[YTKNetworkConfig sharedConfig]`

部分枚举的命名也发生了变化：

* `YTKRequestMethodGet` -> `YTKRequestMethodGET`
* `YTKRequestMethodPost` -> `YTKRequestMethodPOST`
* ...

同时 `ChainCallback` 类型被重命名为 `YTKChainCallback`：

```typedef void (^YTKChainCallback)(YTKChainRequest *chainRequest, YTKBaseRequest *baseRequest);```

