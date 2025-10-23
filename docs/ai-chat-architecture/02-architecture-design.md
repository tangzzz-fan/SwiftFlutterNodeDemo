# AI聊天应用架构设计方案

## 设计原则

### SOLID原则应用

#### 单一职责原则 (SRP)
- **MessageRenderer**: 专注消息渲染逻辑
- **MessageDataSource**: 专注数据管理
- **StreamingManager**: 专注流式数据处理
- **LayoutCalculator**: 专注布局计算

#### 开闭原则 (OCP)
- 通过协议扩展支持新消息类型
- 渲染策略可插拔设计
- 无需修改核心代码即可添加新功能

#### 里氏替换原则 (LSP)
- 所有MessageCell子类可互相替换
- 渲染器实现可透明切换

#### 接口隔离原则 (ISP)
- 细粒度的协议设计
- 避免强制实现不需要的方法

#### 依赖倒置原则 (DIP)
- 依赖抽象而非具体实现
- 通过依赖注入实现解耦

## 核心架构设计

### 1. 协议驱动的类型系统

```swift
// MARK: - 核心协议定义

/// 消息内容协议
protocol MessageContent {
    var id: String { get }
    var type: MessageType { get }
    var timestamp: Date { get }
    var sender: MessageSender { get }
    var content: Any { get }
    var metadata: [String: Any] { get }
}

/// 消息类型协议
protocol MessageType {
    var identifier: String { get }
    var displayName: String { get }
    var cellClass: MessageCellConfigurable.Type { get }
    var renderingStrategy: RenderingStrategy { get }
    var estimatedHeight: CGFloat { get }
    var supportedFeatures: Set<MessageFeature> { get }
}

/// 消息渲染协议
protocol MessageRenderable {
    func render(content: MessageContent, in cell: UICollectionViewCell) -> AnyPublisher<RenderResult, RenderError>
    func calculateLayout(for content: MessageContent, constrainedTo size: CGSize) -> MessageLayout
    func prepareForReuse()
}

/// Cell配置协议
protocol MessageCellConfigurable: UICollectionViewCell {
    var messageRenderer: MessageRenderable? { get set }
    var onHeightChanged: ((IndexPath, CGFloat) -> Void)? { get set }
    var indexPath: IndexPath? { get set }
    
    func configure(with content: MessageContent, at indexPath: IndexPath)
    func updateContent(_ content: MessageContent, animated: Bool)
}

/// 流式数据协议
protocol StreamingDataSource {
    func startStreaming(for messageId: String) -> AnyPublisher<StreamingChunk, StreamingError>
    func stopStreaming(for messageId: String)
    func pauseStreaming(for messageId: String)
    func resumeStreaming(for messageId: String)
}
```

### 2. 消息类型枚举设计

```swift
enum MessageTypeIdentifier: String, CaseIterable {
    case plainText = "text.plain"
    case markdown = "text.markdown"
    case code = "code.block"
    case math = "math.latex"
    case table = "data.table"
    case image = "media.image"
    case file = "media.file"
    case audio = "media.audio"
    case video = "media.video"
    case thinkingChain = "ai.thinking_chain"
    case deepThought = "ai.deep_thought"
    case systemMessage = "system.message"
    case quotedReply = "interaction.quoted_reply"
    case actionSuggestion = "interaction.action_suggestion"
    
    var messageType: MessageType {
        switch self {
        case .plainText:
            return PlainTextMessageType()
        case .markdown:
            return MarkdownMessageType()
        case .code:
            return CodeBlockMessageType()
        case .math:
            return MathMessageType()
        case .table:
            return TableMessageType()
        case .image:
            return ImageMessageType()
        case .file:
            return FileMessageType()
        case .audio:
            return AudioMessageType()
        case .video:
            return VideoMessageType()
        case .thinkingChain:
            return ThinkingChainMessageType()
        case .deepThought:
            return DeepThoughtMessageType()
        case .systemMessage:
            return SystemMessageType()
        case .quotedReply:
            return QuotedReplyMessageType()
        case .actionSuggestion:
            return ActionSuggestionMessageType()
        }
    }
}
```

### 3. 渲染策略设计

```swift
enum RenderingStrategy {
    case native(UIView.Type)
    case webView(WebViewConfig)
    case hybrid(UIView.Type, WebViewConfig)
    case streaming(StreamingConfig)
    
    struct WebViewConfig {
        let htmlTemplate: String
        let cssStyles: String?
        let jsScripts: [String]?
        let allowsInlineMediaPlayback: Bool
        let allowsAirPlay: Bool
    }
    
    struct StreamingConfig {
        let chunkSize: Int
        let updateInterval: TimeInterval
        let animationDuration: TimeInterval
        let bufferSize: Int
    }
}

enum MessageFeature: String, CaseIterable {
    case copyable = "copyable"
    case shareable = "shareable"
    case interactive = "interactive"
    case streaming = "streaming"
    case multimedia = "multimedia"
    case expandable = "expandable"
    case quotable = "quotable"
    case translatable = "translatable"
}
```

### 4. 消息管理器设计

```swift
class MessageManager: ObservableObject {
    @Published private(set) var messages: [MessageContent] = []
    
    private let dataSource: MessageDataSource
    private let streamingManager: StreamingManager
    private let cacheManager: MessageCacheManager
    
    init(dataSource: MessageDataSource, 
         streamingManager: StreamingManager,
         cacheManager: MessageCacheManager) {
        self.dataSource = dataSource
        self.streamingManager = streamingManager
        self.cacheManager = cacheManager
    }
    
    // MARK: - 消息操作
    func addMessage(_ content: MessageContent) {
        messages.append(content)
        cacheManager.cache(content)
    }
    
    func updateMessage(id: String, with newContent: MessageContent) {
        guard let index = messages.firstIndex(where: { $0.id == id }) else { return }
        messages[index] = newContent
        cacheManager.updateCache(for: id, content: newContent)
    }
    
    func startStreamingMessage(id: String) -> AnyPublisher<StreamingChunk, StreamingError> {
        return streamingManager.startStreaming(for: id)
    }
}
```

### 5. 布局计算系统

```swift
class MessageLayoutCalculator {
    private var layoutCache: [String: MessageLayout] = [:]
    private let calculationQueue = DispatchQueue(label: "layout.calculation", qos: .userInitiated)
    
    func calculateLayout(for content: MessageContent, 
                        constrainedTo size: CGSize) -> AnyPublisher<MessageLayout, Never> {
        let cacheKey = "\(content.id)-\(size.width)"
        
        if let cached = layoutCache[cacheKey] {
            return Just(cached).eraseToAnyPublisher()
        }
        
        return Future { [weak self] promise in
            self?.calculationQueue.async {
                let layout = self?.performLayoutCalculation(content: content, size: size) ?? .default
                self?.layoutCache[cacheKey] = layout
                
                DispatchQueue.main.async {
                    promise(.success(layout))
                }
            }
        }
        .eraseToAnyPublisher()
    }
    
    private func performLayoutCalculation(content: MessageContent, size: CGSize) -> MessageLayout {
        let renderer = content.type.cellClass.init().messageRenderer
        return renderer?.calculateLayout(for: content, constrainedTo: size) ?? .default
    }
}

struct MessageLayout {
    let contentSize: CGSize
    let margins: UIEdgeInsets
    let spacing: CGFloat
    let estimatedHeight: CGFloat
    
    static let `default` = MessageLayout(
        contentSize: CGSize(width: 300, height: 100),
        margins: UIEdgeInsets(top: 8, left: 16, bottom: 8, right: 16),
        spacing: 8,
        estimatedHeight: 116
    )
}
```

### 6. 流式渲染管理器

```swift
class StreamingRenderingManager {
    private var activeStreams: [String: StreamingSession] = [:]
    private let renderingQueue = DispatchQueue(label: "streaming.rendering", qos: .userInitiated)
    
    func startStreamingRender(for messageId: String, 
                             content: MessageContent,
                             in cell: MessageCellConfigurable) -> AnyPublisher<RenderProgress, RenderError> {
        
        let session = StreamingSession(messageId: messageId, content: content, cell: cell)
        activeStreams[messageId] = session
        
        return session.startRendering()
            .handleEvents(receiveCompletion: { [weak self] _ in
                self?.activeStreams.removeValue(forKey: messageId)
            })
            .eraseToAnyPublisher()
    }
    
    func stopStreamingRender(for messageId: String) {
        activeStreams[messageId]?.stop()
        activeStreams.removeValue(forKey: messageId)
    }
}

class StreamingSession {
    private let messageId: String
    private let content: MessageContent
    private weak var cell: MessageCellConfigurable?
    private var cancellables = Set<AnyCancellable>()
    private let progressSubject = PassthroughSubject<RenderProgress, RenderError>()
    
    init(messageId: String, content: MessageContent, cell: MessageCellConfigurable) {
        self.messageId = messageId
        self.content = content
        self.cell = cell
    }
    
    func startRendering() -> AnyPublisher<RenderProgress, RenderError> {
        // 实现流式渲染逻辑
        return progressSubject.eraseToAnyPublisher()
    }
    
    func stop() {
        cancellables.removeAll()
        progressSubject.send(completion: .finished)
    }
}
```

## 性能优化架构

### 1. 缓存系统设计

```swift
protocol CacheManager {
    func cache<T>(_ object: T, for key: String) where T: Codable
    func retrieve<T>(_ type: T.Type, for key: String) -> T? where T: Codable
    func remove(for key: String)
    func clearAll()
}

class MessageCacheManager: CacheManager {
    private let memoryCache = NSCache<NSString, AnyObject>()
    private let diskCache: DiskCache
    private let cacheQueue = DispatchQueue(label: "cache.queue", qos: .utility)
    
    // 多级缓存实现
    func cache<T>(_ object: T, for key: String) where T: Codable {
        cacheQueue.async {
            // 内存缓存
            self.memoryCache.setObject(object as AnyObject, forKey: key as NSString)
            
            // 磁盘缓存
            self.diskCache.store(object, for: key)
        }
    }
}
```

### 2. 预渲染系统

```swift
class PreRenderingManager {
    private let preRenderQueue = DispatchQueue(label: "prerender.queue", qos: .background)
    private var preRenderedContent: [String: PreRenderedContent] = [:]
    
    func preRender(messages: [MessageContent]) {
        preRenderQueue.async {
            for message in messages {
                self.preRenderMessage(message)
            }
        }
    }
    
    private func preRenderMessage(_ message: MessageContent) {
        let renderer = message.type.cellClass.init().messageRenderer
        // 在后台预渲染内容
    }
}
```

### 3. 内存管理策略

```swift
class MemoryManager {
    private let memoryPressureSource: DispatchSourceMemoryPressure
    
    init() {
        memoryPressureSource = DispatchSource.makeMemoryPressureSource(eventMask: .all, queue: .main)
        setupMemoryPressureHandling()
    }
    
    private func setupMemoryPressureHandling() {
        memoryPressureSource.setEventHandler { [weak self] in
            self?.handleMemoryPressure()
        }
        memoryPressureSource.resume()
    }
    
    private func handleMemoryPressure() {
        // 清理缓存、释放非必要资源
        MessageCacheManager.shared.clearMemoryCache()
        ImageCache.shared.clearMemoryCache()
        PreRenderingManager.shared.clearPreRenderedContent()
    }
}
```

## 错误处理架构

### 1. 错误类型定义

```swift
enum MessageError: Error, LocalizedError {
    case renderingFailed(String)
    case layoutCalculationFailed(String)
    case streamingInterrupted(String)
    case networkError(Error)
    case cacheError(Error)
    
    var errorDescription: String? {
        switch self {
        case .renderingFailed(let message):
            return "渲染失败: \(message)"
        case .layoutCalculationFailed(let message):
            return "布局计算失败: \(message)"
        case .streamingInterrupted(let message):
            return "流式传输中断: \(message)"
        case .networkError(let error):
            return "网络错误: \(error.localizedDescription)"
        case .cacheError(let error):
            return "缓存错误: \(error.localizedDescription)"
        }
    }
}
```

### 2. 错误恢复机制

```swift
class ErrorRecoveryManager {
    func handleRenderingError(_ error: MessageError, for messageId: String) -> AnyPublisher<RecoveryAction, Never> {
        switch error {
        case .renderingFailed:
            return attemptFallbackRendering(messageId: messageId)
        case .streamingInterrupted:
            return attemptStreamingReconnection(messageId: messageId)
        case .networkError:
            return scheduleRetry(messageId: messageId)
        default:
            return Just(.showErrorMessage(error.localizedDescription)).eraseToAnyPublisher()
        }
    }
    
    private func attemptFallbackRendering(messageId: String) -> AnyPublisher<RecoveryAction, Never> {
        // 降级到简单文本渲染
        return Just(.fallbackToPlainText).eraseToAnyPublisher()
    }
}

enum RecoveryAction {
    case retry
    case fallbackToPlainText
    case showErrorMessage(String)
    case skipMessage
}
```

## 测试架构

### 1. 单元测试支持

```swift
protocol MessageRendererTestable {
    func renderForTesting(content: MessageContent) -> RenderResult
    func calculateLayoutForTesting(content: MessageContent, size: CGSize) -> MessageLayout
}

class MockMessageRenderer: MessageRenderable, MessageRendererTestable {
    // 测试专用的渲染器实现
}
```

### 2. 性能测试框架

```swift
class PerformanceTestSuite {
    func testRenderingPerformance(messageCount: Int) {
        measure {
            // 渲染性能测试
        }
    }
    
    func testMemoryUsage(messageCount: Int) {
        // 内存使用测试
    }
    
    func testScrollingPerformance() {
        // 滚动性能测试
    }
}
```

## 总结

本架构设计基于SOLID原则，采用协议驱动的设计模式，实现了：

1. **高扩展性**：通过协议和枚举设计，支持新消息类型的无缝添加
2. **高性能**：多级缓存、预渲染、异步计算等优化策略
3. **高可靠性**：完善的错误处理和恢复机制
4. **高可测试性**：清晰的依赖注入和模拟对象支持

该架构为后续的三阶段实现提供了坚实的基础，确保每个阶段都能在不破坏现有功能的前提下平滑演进。