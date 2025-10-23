# SSE流式消息渲染设计方案

## 概述

SSE（Server-Sent Events）流式消息渲染是AI聊天应用的核心技术挑战之一。本文档详细设计了变高Cell的实现方案，重点解决实时内容更新、动态高度计算、性能优化等关键问题。

## 核心技术挑战

### 1. 动态高度计算
- **挑战**：内容实时增长导致Cell高度频繁变化
- **影响**：布局抖动、滚动位置跳跃、性能下降
- **解决策略**：预测性高度计算 + 平滑动画过渡

### 2. 渲染性能优化
- **挑战**：频繁的UI更新导致性能瓶颈
- **影响**：界面卡顿、电池消耗增加
- **解决策略**：批量更新 + 异步渲染 + 智能节流

### 3. 用户体验保障
- **挑战**：保持良好的阅读体验和交互响应
- **影响**：用户满意度下降
- **解决策略**：智能滚动跟随 + 视觉反馈优化

## 流式渲染架构设计

### 1. 流式数据处理管道

```swift
class StreamingDataPipeline {
    private let chunkProcessor = ChunkProcessor()
    private let contentBuffer = ContentBuffer()
    private let renderingScheduler = RenderingScheduler()
    
    func processStreamingData(_ data: Data) -> AnyPublisher<ProcessedChunk, StreamingError> {
        return Just(data)
            .flatMap { [weak self] data in
                self?.chunkProcessor.process(data) ?? Empty().eraseToAnyPublisher()
            }
            .flatMap { [weak self] chunk in
                self?.contentBuffer.buffer(chunk) ?? Empty().eraseToAnyPublisher()
            }
            .flatMap { [weak self] bufferedContent in
                self?.renderingScheduler.schedule(bufferedContent) ?? Empty().eraseToAnyPublisher()
            }
            .eraseToAnyPublisher()
    }
}

class ChunkProcessor {
    private var partialData = Data()
    
    func process(_ data: Data) -> AnyPublisher<RawChunk, StreamingError> {
        return Future { [weak self] promise in
            guard let self = self else { return }
            
            self.partialData.append(data)
            
            // 按行分割数据
            let chunks = self.extractCompleteChunks()
            
            for chunk in chunks {
                // 处理每个完整的数据块
                promise(.success(RawChunk(data: chunk)))
            }
        }
        .eraseToAnyPublisher()
    }
    
    private func extractCompleteChunks() -> [Data] {
        // 实现基于换行符的数据分割逻辑
        var chunks: [Data] = []
        // ... 分割逻辑
        return chunks
    }
}
```

### 2. 内容缓冲管理

```swift
class ContentBuffer {
    private var buffer: String = ""
    private let bufferSize: Int
    private let flushInterval: TimeInterval
    private var flushTimer: Timer?
    
    init(bufferSize: Int = 100, flushInterval: TimeInterval = 0.1) {
        self.bufferSize = bufferSize
        self.flushInterval = flushInterval
    }
    
    func buffer(_ chunk: RawChunk) -> AnyPublisher<BufferedContent, Never> {
        return Future { [weak self] promise in
            guard let self = self else { return }
            
            self.buffer += chunk.content
            
            // 检查是否需要立即刷新
            if self.shouldFlush() {
                let content = BufferedContent(
                    text: self.buffer,
                    isComplete: chunk.isComplete,
                    metadata: chunk.metadata
                )
                self.buffer = ""
                promise(.success(content))
            } else {
                // 设置定时刷新
                self.scheduleFlush { content in
                    promise(.success(content))
                }
            }
        }
        .eraseToAnyPublisher()
    }
    
    private func shouldFlush() -> Bool {
        return buffer.count >= bufferSize || 
               buffer.hasSuffix("\n") || 
               buffer.hasSuffix("。") || 
               buffer.hasSuffix("!")
    }
}
```

### 3. 渲染调度器

```swift
class RenderingScheduler {
    private let renderingQueue = DispatchQueue(label: "streaming.rendering", qos: .userInitiated)
    private let mainQueue = DispatchQueue.main
    private var pendingRenders: [String: PendingRender] = [:]
    
    func schedule(_ content: BufferedContent) -> AnyPublisher<ProcessedChunk, StreamingError> {
        return Future { [weak self] promise in
            self?.renderingQueue.async {
                let processedChunk = self?.processContent(content)
                
                self?.mainQueue.async {
                    if let chunk = processedChunk {
                        promise(.success(chunk))
                    } else {
                        promise(.failure(.processingFailed))
                    }
                }
            }
        }
        .eraseToAnyPublisher()
    }
    
    private func processContent(_ content: BufferedContent) -> ProcessedChunk? {
        // 内容预处理：Markdown解析、语法高亮等
        let processedText = preprocessText(content.text)
        let estimatedHeight = calculateEstimatedHeight(processedText)
        
        return ProcessedChunk(
            content: processedText,
            estimatedHeight: estimatedHeight,
            renderingHints: extractRenderingHints(content)
        )
    }
}
```

## 变高Cell实现方案

### 1. 自适应高度Cell设计

```swift
class StreamingMessageCell: UICollectionViewCell, MessageCellConfigurable {
    @IBOutlet weak var contentLabel: UILabel!
    @IBOutlet weak var containerView: UIView!
    
    private var heightConstraint: NSLayoutConstraint?
    private var contentHeightObserver: NSKeyValueObservation?
    
    var onHeightChanged: ((IndexPath, CGFloat) -> Void)?
    var indexPath: IndexPath?
    var messageRenderer: MessageRenderable?
    
    override func awakeFromNib() {
        super.awakeFromNib()
        setupDynamicHeight()
    }
    
    private func setupDynamicHeight() {
        contentLabel.numberOfLines = 0
        contentLabel.lineBreakMode = .byWordWrapping
        
        // 设置初始高度约束
        heightConstraint = containerView.heightAnchor.constraint(greaterThanOrEqualToConstant: 44)
        heightConstraint?.isActive = true
        
        // 监听内容高度变化
        contentHeightObserver = contentLabel.observe(\.bounds) { [weak self] _, _ in
            self?.updateHeightIfNeeded()
        }
    }
    
    func configure(with content: MessageContent, at indexPath: IndexPath) {
        self.indexPath = indexPath
        
        if content.type.supportedFeatures.contains(.streaming) {
            startStreamingRender(content: content)
        } else {
            renderStaticContent(content: content)
        }
    }
    
    private func startStreamingRender(content: MessageContent) {
        guard let streamingContent = content as? StreamingMessageContent else { return }
        
        streamingContent.contentStream
            .receive(on: DispatchQueue.main)
            .sink(
                receiveCompletion: { [weak self] completion in
                    self?.handleStreamingCompletion(completion)
                },
                receiveValue: { [weak self] chunk in
                    self?.updateContent(with: chunk)
                }
            )
            .store(in: &cancellables)
    }
    
    private func updateContent(with chunk: ProcessedChunk) {
        // 使用动画更新内容
        UIView.animate(
            withDuration: 0.2,
            delay: 0,
            options: [.curveEaseOut, .allowUserInteraction],
            animations: {
                self.contentLabel.text = chunk.content
                self.layoutIfNeeded()
            },
            completion: { _ in
                self.updateHeightIfNeeded()
            }
        )
    }
    
    private func updateHeightIfNeeded() {
        guard let indexPath = indexPath else { return }
        
        let newHeight = calculateRequiredHeight()
        let currentHeight = heightConstraint?.constant ?? 0
        
        if abs(newHeight - currentHeight) > 5 { // 避免微小变化
            heightConstraint?.constant = newHeight
            onHeightChanged?(indexPath, newHeight)
        }
    }
    
    private func calculateRequiredHeight() -> CGFloat {
        let labelSize = contentLabel.sizeThatFits(CGSize(
            width: contentLabel.bounds.width,
            height: .greatestFiniteMagnitude
        ))
        
        return labelSize.height + 32 // 添加padding
    }
}
```

### 2. 智能高度预测器

```swift
class HeightPredictor {
    private var historicalData: [String: [CGFloat]] = [:]
    private let maxHistorySize = 50
    
    func predictHeight(for content: String, currentHeight: CGFloat) -> CGFloat {
        let contentType = classifyContent(content)
        let history = historicalData[contentType] ?? []
        
        if history.isEmpty {
            return estimateHeightFromContent(content)
        }
        
        // 使用历史数据和内容特征预测高度
        let averageHeight = history.reduce(0, +) / CGFloat(history.count)
        let contentFactor = CGFloat(content.count) / 100.0
        
        return averageHeight * (1 + contentFactor * 0.1)
    }
    
    func recordActualHeight(_ height: CGFloat, for content: String) {
        let contentType = classifyContent(content)
        var history = historicalData[contentType] ?? []
        
        history.append(height)
        if history.count > maxHistorySize {
            history.removeFirst()
        }
        
        historicalData[contentType] = history
    }
    
    private func classifyContent(_ content: String) -> String {
        if content.contains("```") {
            return "code"
        } else if content.contains("$$") {
            return "math"
        } else if content.contains("|") && content.contains("-") {
            return "table"
        } else {
            return "text"
        }
    }
}
```

### 3. 布局动画管理器

```swift
class LayoutAnimationManager {
    private var animationQueue: [LayoutAnimation] = []
    private var isAnimating = false
    
    func animateHeightChange(
        from oldHeight: CGFloat,
        to newHeight: CGFloat,
        for indexPath: IndexPath,
        in collectionView: UICollectionView
    ) {
        let animation = LayoutAnimation(
            indexPath: indexPath,
            oldHeight: oldHeight,
            newHeight: newHeight,
            duration: calculateAnimationDuration(heightDifference: abs(newHeight - oldHeight))
        )
        
        animationQueue.append(animation)
        processAnimationQueue(in: collectionView)
    }
    
    private func processAnimationQueue(in collectionView: UICollectionView) {
        guard !isAnimating, !animationQueue.isEmpty else { return }
        
        isAnimating = true
        let animation = animationQueue.removeFirst()
        
        UIView.animate(
            withDuration: animation.duration,
            delay: 0,
            usingSpringWithDamping: 0.8,
            initialSpringVelocity: 0.5,
            options: [.curveEaseInOut, .allowUserInteraction],
            animations: {
                collectionView.performBatchUpdates({
                    // 触发布局更新
                }, completion: nil)
            },
            completion: { [weak self] _ in
                self?.isAnimating = false
                self?.processAnimationQueue(in: collectionView)
            }
        )
    }
    
    private func calculateAnimationDuration(heightDifference: CGFloat) -> TimeInterval {
        // 根据高度变化幅度调整动画时长
        let baseDuration: TimeInterval = 0.2
        let maxDuration: TimeInterval = 0.5
        
        let factor = min(heightDifference / 200.0, 1.0)
        return baseDuration + (maxDuration - baseDuration) * TimeInterval(factor)
    }
}

struct LayoutAnimation {
    let indexPath: IndexPath
    let oldHeight: CGFloat
    let newHeight: CGFloat
    let duration: TimeInterval
}
```

## 性能优化策略

### 1. 渲染节流机制

```swift
class RenderingThrottler {
    private let throttleInterval: TimeInterval
    private var lastRenderTime: Date = Date()
    private var pendingRender: (() -> Void)?
    private var throttleTimer: Timer?
    
    init(throttleInterval: TimeInterval = 0.05) {
        self.throttleInterval = throttleInterval
    }
    
    func throttleRender(_ renderBlock: @escaping () -> Void) {
        let now = Date()
        let timeSinceLastRender = now.timeIntervalSince(lastRenderTime)
        
        if timeSinceLastRender >= throttleInterval {
            // 立即执行
            renderBlock()
            lastRenderTime = now
        } else {
            // 延迟执行
            pendingRender = renderBlock
            scheduleDelayedRender()
        }
    }
    
    private func scheduleDelayedRender() {
        throttleTimer?.invalidate()
        
        let remainingTime = throttleInterval - Date().timeIntervalSince(lastRenderTime)
        
        throttleTimer = Timer.scheduledTimer(withTimeInterval: max(remainingTime, 0.01), repeats: false) { [weak self] _ in
            self?.executePendingRender()
        }
    }
    
    private func executePendingRender() {
        pendingRender?()
        pendingRender = nil
        lastRenderTime = Date()
    }
}
```

### 2. 内存压力管理

```swift
class StreamingMemoryManager {
    private let maxBufferSize: Int = 1024 * 1024 // 1MB
    private var currentBufferSize: Int = 0
    private var contentBuffers: [String: String] = [:]
    
    func addContent(_ content: String, for messageId: String) {
        let contentSize = content.utf8.count
        
        // 检查内存压力
        if currentBufferSize + contentSize > maxBufferSize {
            performMemoryCleanup()
        }
        
        contentBuffers[messageId] = content
        currentBufferSize += contentSize
    }
    
    private func performMemoryCleanup() {
        // 清理最旧的内容
        let sortedBuffers = contentBuffers.sorted { $0.key < $1.key }
        let toRemove = sortedBuffers.prefix(sortedBuffers.count / 2)
        
        for (messageId, content) in toRemove {
            currentBufferSize -= content.utf8.count
            contentBuffers.removeValue(forKey: messageId)
        }
    }
}
```

### 3. 滚动位置管理

```swift
class ScrollPositionManager {
    private weak var collectionView: UICollectionView?
    private var isUserScrolling = false
    private var shouldFollowNewContent = true
    
    init(collectionView: UICollectionView) {
        self.collectionView = collectionView
        setupScrollObserver()
    }
    
    private func setupScrollObserver() {
        collectionView?.panGestureRecognizer.addTarget(self, action: #selector(handlePanGesture(_:)))
    }
    
    @objc private func handlePanGesture(_ gesture: UIPanGestureRecognizer) {
        switch gesture.state {
        case .began:
            isUserScrolling = true
            shouldFollowNewContent = false
        case .ended, .cancelled:
            isUserScrolling = false
            // 检查是否接近底部，决定是否恢复自动跟随
            checkShouldResumeFollowing()
        default:
            break
        }
    }
    
    private func checkShouldResumeFollowing() {
        guard let collectionView = collectionView else { return }
        
        let contentHeight = collectionView.contentSize.height
        let visibleHeight = collectionView.bounds.height
        let currentOffset = collectionView.contentOffset.y
        
        // 如果滚动到接近底部，恢复自动跟随
        if contentHeight - (currentOffset + visibleHeight) < 100 {
            shouldFollowNewContent = true
        }
    }
    
    func handleNewContent(at indexPath: IndexPath) {
        guard shouldFollowNewContent, !isUserScrolling else { return }
        
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
            self.scrollToBottom(animated: true)
        }
    }
    
    private func scrollToBottom(animated: Bool) {
        guard let collectionView = collectionView else { return }
        
        let numberOfSections = collectionView.numberOfSections
        guard numberOfSections > 0 else { return }
        
        let numberOfItems = collectionView.numberOfItems(inSection: numberOfSections - 1)
        guard numberOfItems > 0 else { return }
        
        let lastIndexPath = IndexPath(item: numberOfItems - 1, section: numberOfSections - 1)
        collectionView.scrollToItem(at: lastIndexPath, at: .bottom, animated: animated)
    }
}
```

## 错误处理和恢复

### 1. 流式传输错误处理

```swift
class StreamingErrorHandler {
    private let maxRetryAttempts = 3
    private var retryAttempts: [String: Int] = [:]
    
    func handleStreamingError(_ error: StreamingError, for messageId: String) -> AnyPublisher<RecoveryAction, Never> {
        let currentAttempts = retryAttempts[messageId] ?? 0
        
        switch error {
        case .connectionLost:
            if currentAttempts < maxRetryAttempts {
                return attemptReconnection(messageId: messageId)
            } else {
                return Just(.showErrorState("连接已断开")).eraseToAnyPublisher()
            }
            
        case .dataCorrupted:
            return Just(.requestResend(messageId)).eraseToAnyPublisher()
            
        case .timeout:
            return Just(.showTimeoutError).eraseToAnyPublisher()
        }
    }
    
    private func attemptReconnection(messageId: String) -> AnyPublisher<RecoveryAction, Never> {
        retryAttempts[messageId] = (retryAttempts[messageId] ?? 0) + 1
        
        return Timer.publish(every: 2.0, on: .main, in: .common)
            .autoconnect()
            .first()
            .map { _ in .retry }
            .eraseToAnyPublisher()
    }
}

enum RecoveryAction {
    case retry
    case requestResend(String)
    case showErrorState(String)
    case showTimeoutError
    case fallbackToStaticContent
}
```

### 2. 渲染失败降级

```swift
class RenderingFallbackManager {
    func handleRenderingFailure(content: MessageContent) -> MessageContent {
        // 降级到纯文本显示
        let fallbackContent = PlainTextMessageContent(
            id: content.id,
            text: extractPlainText(from: content),
            sender: content.sender,
            timestamp: content.timestamp
        )
        
        return fallbackContent
    }
    
    private func extractPlainText(from content: MessageContent) -> String {
        // 从复杂内容中提取纯文本
        switch content.type.identifier {
        case "text.markdown":
            return stripMarkdown(content.content as? String ?? "")
        case "code.block":
            return extractCodeText(content.content)
        default:
            return content.content as? String ?? "内容加载失败"
        }
    }
}
```

## 用户体验优化

### 1. 加载状态指示

```swift
class StreamingLoadingIndicator: UIView {
    private let dotsView = UIStackView()
    private var animationTimer: Timer?
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        setupUI()
    }
    
    private func setupUI() {
        dotsView.axis = .horizontal
        dotsView.spacing = 4
        dotsView.distribution = .equalSpacing
        
        for _ in 0..<3 {
            let dot = UIView()
            dot.backgroundColor = .systemBlue
            dot.layer.cornerRadius = 3
            dot.widthAnchor.constraint(equalToConstant: 6).isActive = true
            dot.heightAnchor.constraint(equalToConstant: 6).isActive = true
            dotsView.addArrangedSubview(dot)
        }
        
        addSubview(dotsView)
        dotsView.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            dotsView.centerXAnchor.constraint(equalTo: centerXAnchor),
            dotsView.centerYAnchor.constraint(equalTo: centerYAnchor)
        ])
    }
    
    func startAnimating() {
        animationTimer = Timer.scheduledTimer(withTimeInterval: 0.5, repeats: true) { [weak self] _ in
            self?.animateDots()
        }
    }
    
    func stopAnimating() {
        animationTimer?.invalidate()
        animationTimer = nil
    }
    
    private func animateDots() {
        for (index, dot) in dotsView.arrangedSubviews.enumerated() {
            UIView.animate(
                withDuration: 0.3,
                delay: TimeInterval(index) * 0.1,
                options: [.curveEaseInOut],
                animations: {
                    dot.alpha = dot.alpha == 1.0 ? 0.3 : 1.0
                }
            )
        }
    }
}
```

### 2. 打字机效果优化

```swift
class TypewriterEffectManager {
    private let charactersPerSecond: Double = 50
    private var displayTimer: Timer?
    
    func displayText(_ text: String, in label: UILabel, completion: @escaping () -> Void) {
        label.text = ""
        
        let characters = Array(text)
        var currentIndex = 0
        
        let interval = 1.0 / charactersPerSecond
        
        displayTimer = Timer.scheduledTimer(withTimeInterval: interval, repeats: true) { timer in
            if currentIndex < characters.count {
                label.text = String(characters[0...currentIndex])
                currentIndex += 1
            } else {
                timer.invalidate()
                completion()
            }
        }
    }
    
    func stopEffect() {
        displayTimer?.invalidate()
        displayTimer = nil
    }
}
```

## 总结

本设计方案通过以下关键技术实现了高质量的SSE流式消息渲染：

1. **分层架构**：数据处理、内容缓冲、渲染调度的清晰分离
2. **智能优化**：高度预测、渲染节流、内存管理的综合优化
3. **用户体验**：平滑动画、智能滚动、状态反馈的全面考虑
4. **错误处理**：完善的错误恢复和降级机制

该方案为实现流畅、稳定的AI聊天体验提供了完整的技术保障。