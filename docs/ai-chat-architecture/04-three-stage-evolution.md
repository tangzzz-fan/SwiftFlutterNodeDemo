# AI聊天应用三阶段演进方案

## 概述

本文档详细描述了AI聊天应用从基础功能到高级渲染能力的三阶段演进过程。每个阶段都有明确的技术目标、实现方案和过渡策略，确保系统的平滑升级和功能扩展。

## 阶段一：基础CollectionView聊天界面

### 技术目标
- 实现基础的聊天消息展示
- 建立可扩展的消息类型系统
- 处理基本的布局和高度计算
- 奠定后续演进的架构基础

### 核心技术方案

#### 1. 基础消息模型设计

```swift
// 基础消息协议
protocol MessageContent {
    var id: String { get }
    var type: MessageType { get }
    var content: Any { get }
    var sender: MessageSender { get }
    var timestamp: Date { get }
    var metadata: MessageMetadata? { get }
}

// 消息类型枚举
enum MessageType: String, CaseIterable {
    case text = "text.plain"
    case image = "media.image"
    case system = "system.notification"
    
    var cellIdentifier: String {
        switch self {
        case .text: return "TextMessageCell"
        case .image: return "ImageMessageCell"
        case .system: return "SystemMessageCell"
        }
    }
    
    var estimatedHeight: CGFloat {
        switch self {
        case .text: return 60
        case .image: return 200
        case .system: return 40
        }
    }
}

// 具体消息实现
struct TextMessageContent: MessageContent {
    let id: String
    let type: MessageType = .text
    let content: Any
    let sender: MessageSender
    let timestamp: Date
    let metadata: MessageMetadata?
    
    var text: String {
        return content as? String ?? ""
    }
}
```

#### 2. CollectionView基础架构

```swift
class ChatCollectionViewController: UIViewController {
    @IBOutlet weak var collectionView: UICollectionView!
    
    private var messages: [MessageContent] = []
    private let messageManager = MessageManager()
    private let layoutManager = ChatLayoutManager()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupCollectionView()
        setupMessageHandling()
    }
    
    private func setupCollectionView() {
        // 注册Cell类型
        collectionView.register(
            UINib(nibName: "TextMessageCell", bundle: nil),
            forCellWithReuseIdentifier: "TextMessageCell"
        )
        
        // 设置布局
        let layout = ChatFlowLayout()
        layout.estimatedItemSize = UICollectionViewFlowLayout.automaticSize
        layout.minimumLineSpacing = 8
        layout.sectionInset = UIEdgeInsets(top: 16, left: 16, bottom: 16, right: 16)
        collectionView.collectionViewLayout = layout
        
        // 设置代理
        collectionView.delegate = self
        collectionView.dataSource = self
    }
    
    private func setupMessageHandling() {
        messageManager.onNewMessage = { [weak self] message in
            DispatchQueue.main.async {
                self?.addMessage(message)
            }
        }
    }
    
    private func addMessage(_ message: MessageContent) {
        messages.append(message)
        
        let indexPath = IndexPath(item: messages.count - 1, section: 0)
        collectionView.performBatchUpdates({
            collectionView.insertItems(at: [indexPath])
        }) { _ in
            self.scrollToBottom(animated: true)
        }
    }
}

// MARK: - UICollectionViewDataSource
extension ChatCollectionViewController: UICollectionViewDataSource {
    func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        return messages.count
    }
    
    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        let message = messages[indexPath.item]
        let cell = collectionView.dequeueReusableCell(
            withReuseIdentifier: message.type.cellIdentifier,
            for: indexPath
        )
        
        if let configurableCell = cell as? MessageCellConfigurable {
            configurableCell.configure(with: message, at: indexPath)
        }
        
        return cell
    }
}
```

#### 3. 自适应高度Cell实现

```swift
class TextMessageCell: UICollectionViewCell, MessageCellConfigurable {
    @IBOutlet weak var messageLabel: UILabel!
    @IBOutlet weak var containerView: UIView!
    @IBOutlet weak var timestampLabel: UILabel!
    
    private var widthConstraint: NSLayoutConstraint?
    
    override func awakeFromNib() {
        super.awakeFromNib()
        setupUI()
    }
    
    private func setupUI() {
        messageLabel.numberOfLines = 0
        messageLabel.lineBreakMode = .byWordWrapping
        
        containerView.layer.cornerRadius = 12
        containerView.backgroundColor = .systemBlue
        
        // 设置最大宽度约束
        widthConstraint = containerView.widthAnchor.constraint(lessThanOrEqualToConstant: 280)
        widthConstraint?.isActive = true
    }
    
    func configure(with content: MessageContent, at indexPath: IndexPath) {
        guard let textContent = content as? TextMessageContent else { return }
        
        messageLabel.text = textContent.text
        timestampLabel.text = formatTimestamp(textContent.timestamp)
        
        // 根据发送者调整布局
        updateLayoutForSender(textContent.sender)
        
        // 触发自动布局计算
        setNeedsLayout()
        layoutIfNeeded()
    }
    
    private func updateLayoutForSender(_ sender: MessageSender) {
        if sender.isCurrentUser {
            containerView.backgroundColor = .systemBlue
            messageLabel.textColor = .white
            // 右对齐布局
        } else {
            containerView.backgroundColor = .systemGray5
            messageLabel.textColor = .label
            // 左对齐布局
        }
    }
    
    override func preferredLayoutAttributesFitting(_ layoutAttributes: UICollectionViewLayoutAttributes) -> UICollectionViewLayoutAttributes {
        let targetSize = CGSize(width: layoutAttributes.frame.width, height: 0)
        let size = systemLayoutSizeFitting(
            targetSize,
            withHorizontalFittingPriority: .required,
            verticalFittingPriority: .fittingSizeLevel
        )
        
        var frame = layoutAttributes.frame
        frame.size.height = ceil(size.height)
        layoutAttributes.frame = frame
        
        return layoutAttributes
    }
}
```

#### 4. 布局管理器

```swift
class ChatFlowLayout: UICollectionViewFlowLayout {
    override func prepare() {
        super.prepare()
        
        guard let collectionView = collectionView else { return }
        
        // 动态调整item宽度
        let availableWidth = collectionView.bounds.width - sectionInset.left - sectionInset.right
        estimatedItemSize = CGSize(width: availableWidth, height: 60)
    }
    
    override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
        let attributes = super.layoutAttributesForElements(in: rect)
        
        // 优化布局属性
        attributes?.forEach { attribute in
            if attribute.representedElementCategory == .cell {
                optimizeLayoutAttribute(attribute)
            }
        }
        
        return attributes
    }
    
    private func optimizeLayoutAttribute(_ attribute: UICollectionViewLayoutAttributes) {
        // 确保Cell不会超出边界
        let maxWidth = collectionView?.bounds.width ?? 0
        if attribute.frame.width > maxWidth {
            attribute.frame.size.width = maxWidth - sectionInset.left - sectionInset.right
        }
    }
}
```

### 阶段一总结

**实现成果：**
- 基础聊天界面框架
- 可扩展的消息类型系统
- 自适应高度的Cell实现
- 基础的布局管理

**技术债务：**
- 仅支持纯文本消息
- 布局计算相对简单
- 缺乏复杂内容渲染能力

## 阶段二：Markdown渲染集成

### 技术目标
- 集成Swift Markdown渲染库
- 支持富文本消息展示
- 优化渲染性能
- 保持向后兼容性

### 核心技术方案

#### 1. Markdown渲染器集成

```swift
import Down

class MarkdownRenderer {
    private let downProcessor: Down
    private let renderingQueue = DispatchQueue(label: "markdown.rendering", qos: .userInitiated)
    
    init() {
        // 配置Markdown处理器
        self.downProcessor = Down(markdownString: "")
    }
    
    func renderMarkdown(_ markdown: String) -> AnyPublisher<NSAttributedString, MarkdownError> {
        return Future { [weak self] promise in
            self?.renderingQueue.async {
                do {
                    let down = Down(markdownString: markdown)
                    let attributedString = try down.toAttributedString(.default, stylesheet: self?.createStylesheet())
                    
                    DispatchQueue.main.async {
                        promise(.success(attributedString))
                    }
                } catch {
                    DispatchQueue.main.async {
                        promise(.failure(.renderingFailed(error)))
                    }
                }
            }
        }
        .eraseToAnyPublisher()
    }
    
    private func createStylesheet() -> String {
        return """
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            font-size: 16px;
            line-height: 1.5;
            color: #333;
        }
        
        code {
            background-color: #f6f8fa;
            border-radius: 3px;
            padding: 2px 4px;
            font-family: 'SF Mono', Monaco, 'Cascadia Code', monospace;
        }
        
        pre {
            background-color: #f6f8fa;
            border-radius: 6px;
            padding: 16px;
            overflow-x: auto;
        }
        
        blockquote {
            border-left: 4px solid #dfe2e5;
            padding-left: 16px;
            margin-left: 0;
            color: #6a737d;
        }
        """
    }
}
```

#### 2. 增强的消息类型系统

```swift
// 扩展消息类型
enum MessageType: String, CaseIterable {
    case text = "text.plain"
    case markdown = "text.markdown"
    case code = "code.block"
    case image = "media.image"
    case system = "system.notification"
    
    var supportedFeatures: Set<MessageFeature> {
        switch self {
        case .text:
            return [.basicFormatting]
        case .markdown:
            return [.richText, .codeHighlighting, .linkPreviews]
        case .code:
            return [.syntaxHighlighting, .copyable]
        case .image:
            return [.zoomable, .downloadable]
        case .system:
            return []
        }
    }
    
    var preferredRenderer: MessageRenderer.Type {
        switch self {
        case .text:
            return PlainTextRenderer.self
        case .markdown:
            return MarkdownRenderer.self
        case .code:
            return CodeBlockRenderer.self
        case .image:
            return ImageRenderer.self
        case .system:
            return SystemMessageRenderer.self
        }
    }
}

enum MessageFeature {
    case basicFormatting
    case richText
    case codeHighlighting
    case syntaxHighlighting
    case linkPreviews
    case zoomable
    case downloadable
    case copyable
}

// Markdown消息内容
struct MarkdownMessageContent: MessageContent {
    let id: String
    let type: MessageType = .markdown
    let content: Any
    let sender: MessageSender
    let timestamp: Date
    let metadata: MessageMetadata?
    
    var markdownText: String {
        return content as? String ?? ""
    }
    
    var renderingOptions: MarkdownRenderingOptions {
        return MarkdownRenderingOptions(
            enableCodeHighlighting: true,
            enableLinkPreviews: true,
            enableMathRendering: false
        )
    }
}

struct MarkdownRenderingOptions {
    let enableCodeHighlighting: Bool
    let enableLinkPreviews: Bool
    let enableMathRendering: Bool
}
```

#### 3. Markdown消息Cell实现

```swift
class MarkdownMessageCell: UICollectionViewCell, MessageCellConfigurable {
    @IBOutlet weak var contentTextView: UITextView!
    @IBOutlet weak var containerView: UIView!
    @IBOutlet weak var loadingIndicator: UIActivityIndicatorView!
    
    private var cancellables = Set<AnyCancellable>()
    private let markdownRenderer = MarkdownRenderer()
    private var heightCache: [String: CGFloat] = [:]
    
    var onHeightChanged: ((IndexPath, CGFloat) -> Void)?
    var indexPath: IndexPath?
    
    override func awakeFromNib() {
        super.awakeFromNib()
        setupTextView()
    }
    
    private func setupTextView() {
        contentTextView.isEditable = false
        contentTextView.isScrollEnabled = false
        contentTextView.textContainer.lineFragmentPadding = 0
        contentTextView.textContainerInset = UIEdgeInsets(top: 12, left: 12, bottom: 12, right: 12)
        
        // 设置链接处理
        contentTextView.delegate = self
        contentTextView.dataDetectorTypes = [.link, .phoneNumber]
    }
    
    func configure(with content: MessageContent, at indexPath: IndexPath) {
        self.indexPath = indexPath
        
        guard let markdownContent = content as? MarkdownMessageContent else { return }
        
        // 检查缓存
        if let cachedHeight = heightCache[content.id] {
            renderCachedContent(markdownContent, cachedHeight: cachedHeight)
        } else {
            renderMarkdownContent(markdownContent)
        }
    }
    
    private func renderMarkdownContent(_ content: MarkdownMessageContent) {
        showLoadingState()
        
        markdownRenderer.renderMarkdown(content.markdownText)
            .receive(on: DispatchQueue.main)
            .sink(
                receiveCompletion: { [weak self] completion in
                    self?.hideLoadingState()
                    if case .failure(let error) = completion {
                        self?.showErrorState(error)
                    }
                },
                receiveValue: { [weak self] attributedString in
                    self?.displayRenderedContent(attributedString, for: content)
                }
            )
            .store(in: &cancellables)
    }
    
    private func displayRenderedContent(_ attributedString: NSAttributedString, for content: MarkdownMessageContent) {
        contentTextView.attributedText = attributedString
        
        // 计算并缓存高度
        let newHeight = calculateRequiredHeight()
        heightCache[content.id] = newHeight
        
        // 通知高度变化
        if let indexPath = indexPath {
            onHeightChanged?(indexPath, newHeight)
        }
        
        // 添加渐入动画
        UIView.animate(withDuration: 0.3) {
            self.contentTextView.alpha = 1.0
        }
    }
    
    private func calculateRequiredHeight() -> CGFloat {
        let textViewSize = contentTextView.sizeThatFits(CGSize(
            width: contentTextView.bounds.width,
            height: .greatestFiniteMagnitude
        ))
        
        return textViewSize.height + 24 // 添加容器padding
    }
    
    private func showLoadingState() {
        contentTextView.alpha = 0.5
        loadingIndicator.startAnimating()
    }
    
    private func hideLoadingState() {
        loadingIndicator.stopAnimating()
    }
    
    private func showErrorState(_ error: MarkdownError) {
        contentTextView.text = "渲染失败: \(error.localizedDescription)"
        contentTextView.textColor = .systemRed
    }
}

// MARK: - UITextViewDelegate
extension MarkdownMessageCell: UITextViewDelegate {
    func textView(_ textView: UITextView, shouldInteractWith URL: URL, in characterRange: NSRange, interaction: UITextItemInteraction) -> Bool {
        // 处理链接点击
        if interaction == .invokeDefaultAction {
            handleLinkTap(URL)
            return false
        }
        return true
    }
    
    private func handleLinkTap(_ url: URL) {
        // 实现链接处理逻辑
        NotificationCenter.default.post(
            name: .linkTapped,
            object: nil,
            userInfo: ["url": url]
        )
    }
}
```

#### 4. 渲染性能优化

```swift
class MarkdownRenderingCache {
    private let cache = NSCache<NSString, CachedMarkdownResult>()
    private let renderingQueue = DispatchQueue(label: "markdown.cache", qos: .utility)
    
    init() {
        cache.countLimit = 100
        cache.totalCostLimit = 50 * 1024 * 1024 // 50MB
    }
    
    func getCachedResult(for markdown: String) -> CachedMarkdownResult? {
        return cache.object(forKey: markdown as NSString)
    }
    
    func cacheResult(_ result: CachedMarkdownResult, for markdown: String) {
        let cost = result.attributedString.length * 2 // 估算内存占用
        cache.setObject(result, forKey: markdown as NSString, cost: cost)
    }
    
    func preloadMarkdown(_ markdowns: [String]) {
        renderingQueue.async {
            for markdown in markdowns {
                if self.getCachedResult(for: markdown) == nil {
                    // 预渲染并缓存
                    self.prerenderMarkdown(markdown)
                }
            }
        }
    }
    
    private func prerenderMarkdown(_ markdown: String) {
        do {
            let down = Down(markdownString: markdown)
            let attributedString = try down.toAttributedString()
            let result = CachedMarkdownResult(
                attributedString: attributedString,
                estimatedHeight: calculateEstimatedHeight(attributedString),
                renderingTime: Date()
            )
            cacheResult(result, for: markdown)
        } catch {
            // 预渲染失败，忽略错误
        }
    }
}

class CachedMarkdownResult: NSObject {
    let attributedString: NSAttributedString
    let estimatedHeight: CGFloat
    let renderingTime: Date
    
    init(attributedString: NSAttributedString, estimatedHeight: CGFloat, renderingTime: Date) {
        self.attributedString = attributedString
        self.estimatedHeight = estimatedHeight
        self.renderingTime = renderingTime
    }
}
```

### 阶段二总结

**实现成果：**
- 完整的Markdown渲染支持
- 性能优化的渲染缓存
- 增强的消息类型系统
- 链接交互处理

**技术债务：**
- 复杂内容渲染仍有限制
- 缺乏数学公式支持
- 表格渲染效果有限

## 阶段三：WebView增强渲染

### 技术目标
- 集成WKWebView实现复杂内容渲染
- 支持数学公式、图表、交互式内容
- 实现高性能的WebView复用机制
- 完善的动态高度计算和流畅滚动

### 核心技术方案

#### 1. WebView渲染引擎

```swift
class WebViewRenderingEngine {
    private let webViewPool = WebViewPool()
    private let contentProcessor = WebContentProcessor()
    private let heightCalculator = WebViewHeightCalculator()
    
    func renderContent(_ content: WebRenderableContent) -> AnyPublisher<WebViewRenderResult, WebRenderingError> {
        return webViewPool.acquireWebView()
            .flatMap { [weak self] webView in
                self?.processAndRender(content, in: webView) ?? 
                Fail(error: WebRenderingError.webViewUnavailable).eraseToAnyPublisher()
            }
            .eraseToAnyPublisher()
    }
    
    private func processAndRender(_ content: WebRenderableContent, in webView: WKWebView) -> AnyPublisher<WebViewRenderResult, WebRenderingError> {
        return contentProcessor.processContent(content)
            .flatMap { [weak self] processedHTML in
                self?.loadHTMLInWebView(processedHTML, webView: webView) ?? 
                Fail(error: WebRenderingError.processingFailed).eraseToAnyPublisher()
            }
            .flatMap { [weak self] webView in
                self?.heightCalculator.calculateHeight(for: webView) ?? 
                Fail(error: WebRenderingError.heightCalculationFailed).eraseToAnyPublisher()
            }
            .map { height in
                WebViewRenderResult(webView: webView, contentHeight: height)
            }
            .eraseToAnyPublisher()
    }
}

class WebContentProcessor {
    private let mathJaxProcessor = MathJaxProcessor()
    private let codeHighlighter = CodeHighlighter()
    private let chartRenderer = ChartRenderer()
    
    func processContent(_ content: WebRenderableContent) -> AnyPublisher<String, WebRenderingError> {
        return Future { [weak self] promise in
            guard let self = self else { return }
            
            var html = self.createBaseHTML()
            
            switch content.type {
            case .markdown:
                html += self.processMarkdown(content.content)
            case .mathFormula:
                html += self.mathJaxProcessor.processMath(content.content)
            case .chart:
                html += self.chartRenderer.renderChart(content.content)
            case .interactive:
                html += self.processInteractiveContent(content.content)
            }
            
            html += self.createHTMLFooter()
            promise(.success(html))
        }
        .eraseToAnyPublisher()
    }
    
    private func createBaseHTML() -> String {
        return """
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="utf-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
            <style>
                body {
                    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
                    font-size: 16px;
                    line-height: 1.6;
                    margin: 0;
                    padding: 16px;
                    word-wrap: break-word;
                }
                
                .math-container {
                    overflow-x: auto;
                    margin: 16px 0;
                }
                
                .code-block {
                    background-color: #f6f8fa;
                    border-radius: 6px;
                    padding: 16px;
                    overflow-x: auto;
                    margin: 16px 0;
                }
                
                .chart-container {
                    margin: 16px 0;
                    text-align: center;
                }
            </style>
            <script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
            <script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
            <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
        </head>
        <body>
        """
    }
}
```

#### 2. WebView池管理

```swift
class WebViewPool {
    private var availableWebViews: [WKWebView] = []
    private var inUseWebViews: Set<WKWebView> = []
    private let maxPoolSize = 5
    private let creationQueue = DispatchQueue(label: "webview.creation", qos: .userInitiated)
    
    func acquireWebView() -> AnyPublisher<WKWebView, WebViewPoolError> {
        return Future { [weak self] promise in
            self?.creationQueue.async {
                if let webView = self?.getAvailableWebView() {
                    DispatchQueue.main.async {
                        promise(.success(webView))
                    }
                } else {
                    self?.createNewWebView { webView in
                        DispatchQueue.main.async {
                            promise(.success(webView))
                        }
                    }
                }
            }
        }
        .eraseToAnyPublisher()
    }
    
    func releaseWebView(_ webView: WKWebView) {
        creationQueue.async {
            self.inUseWebViews.remove(webView)
            
            // 清理WebView状态
            webView.stopLoading()
            webView.loadHTMLString("", baseURL: nil)
            
            if self.availableWebViews.count < self.maxPoolSize {
                self.availableWebViews.append(webView)
            } else {
                // 销毁多余的WebView
                DispatchQueue.main.async {
                    webView.removeFromSuperview()
                }
            }
        }
    }
    
    private func getAvailableWebView() -> WKWebView? {
        if !availableWebViews.isEmpty {
            let webView = availableWebViews.removeFirst()
            inUseWebViews.insert(webView)
            return webView
        }
        return nil
    }
    
    private func createNewWebView(completion: @escaping (WKWebView) -> Void) {
        DispatchQueue.main.async {
            let configuration = WKWebViewConfiguration()
            configuration.allowsInlineMediaPlayback = true
            configuration.mediaTypesRequiringUserActionForPlayback = []
            
            let webView = WKWebView(frame: .zero, configuration: configuration)
            webView.scrollView.isScrollEnabled = false
            webView.isOpaque = false
            webView.backgroundColor = .clear
            
            self.inUseWebViews.insert(webView)
            completion(webView)
        }
    }
}
```

#### 3. 高性能WebView消息Cell

```swift
class WebViewMessageCell: UICollectionViewCell, MessageCellConfigurable {
    private var webView: WKWebView?
    private var webViewContainer: UIView!
    private var heightConstraint: NSLayoutConstraint?
    private var loadingIndicator: UIActivityIndicatorView!
    
    private let renderingEngine = WebViewRenderingEngine()
    private var cancellables = Set<AnyCancellable>()
    
    var onHeightChanged: ((IndexPath, CGFloat) -> Void)?
    var indexPath: IndexPath?
    
    override func awakeFromNib() {
        super.awakeFromNib()
        setupUI()
    }
    
    private func setupUI() {
        // 创建WebView容器
        webViewContainer = UIView()
        webViewContainer.translatesAutoresizingMaskIntoConstraints = false
        contentView.addSubview(webViewContainer)
        
        // 创建加载指示器
        loadingIndicator = UIActivityIndicatorView(style: .medium)
        loadingIndicator.translatesAutoresizingMaskIntoConstraints = false
        contentView.addSubview(loadingIndicator)
        
        // 设置约束
        NSLayoutConstraint.activate([
            webViewContainer.topAnchor.constraint(equalTo: contentView.topAnchor, constant: 8),
            webViewContainer.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: 16),
            webViewContainer.trailingAnchor.constraint(equalTo: contentView.trailingAnchor, constant: -16),
            webViewContainer.bottomAnchor.constraint(equalTo: contentView.bottomAnchor, constant: -8),
            
            loadingIndicator.centerXAnchor.constraint(equalTo: contentView.centerXAnchor),
            loadingIndicator.centerYAnchor.constraint(equalTo: contentView.centerYAnchor)
        ])
        
        // 初始高度约束
        heightConstraint = webViewContainer.heightAnchor.constraint(equalToConstant: 100)
        heightConstraint?.isActive = true
    }
    
    func configure(with content: MessageContent, at indexPath: IndexPath) {
        self.indexPath = indexPath
        
        guard let webContent = content as? WebRenderableContent else { return }
        
        showLoadingState()
        renderWebContent(webContent)
    }
    
    private func renderWebContent(_ content: WebRenderableContent) {
        renderingEngine.renderContent(content)
            .receive(on: DispatchQueue.main)
            .sink(
                receiveCompletion: { [weak self] completion in
                    self?.hideLoadingState()
                    if case .failure(let error) = completion {
                        self?.showErrorState(error)
                    }
                },
                receiveValue: { [weak self] result in
                    self?.displayWebViewResult(result)
                }
            )
            .store(in: &cancellables)
    }
    
    private func displayWebViewResult(_ result: WebViewRenderResult) {
        // 移除旧的WebView
        webView?.removeFromSuperview()
        
        // 设置新的WebView
        webView = result.webView
        webView?.translatesAutoresizingMaskIntoConstraints = false
        webViewContainer.addSubview(result.webView)
        
        // 设置WebView约束
        NSLayoutConstraint.activate([
            result.webView.topAnchor.constraint(equalTo: webViewContainer.topAnchor),
            result.webView.leadingAnchor.constraint(equalTo: webViewContainer.leadingAnchor),
            result.webView.trailingAnchor.constraint(equalTo: webViewContainer.trailingAnchor),
            result.webView.bottomAnchor.constraint(equalTo: webViewContainer.bottomAnchor)
        ])
        
        // 更新高度
        updateHeight(result.contentHeight)
        
        // 添加渐入动画
        result.webView.alpha = 0
        UIView.animate(withDuration: 0.3) {
            result.webView.alpha = 1
        }
    }
    
    private func updateHeight(_ newHeight: CGFloat) {
        guard let indexPath = indexPath else { return }
        
        heightConstraint?.constant = newHeight
        
        UIView.animate(
            withDuration: 0.3,
            delay: 0,
            usingSpringWithDamping: 0.8,
            initialSpringVelocity: 0.5,
            options: [.curveEaseInOut],
            animations: {
                self.layoutIfNeeded()
            },
            completion: { _ in
                self.onHeightChanged?(indexPath, newHeight + 16) // 添加padding
            }
        )
    }
    
    override func prepareForReuse() {
        super.prepareForReuse()
        
        // 释放WebView回池中
        if let webView = webView {
            WebViewPool.shared.releaseWebView(webView)
            self.webView = nil
        }
        
        cancellables.removeAll()
        hideLoadingState()
    }
    
    private func showLoadingState() {
        loadingIndicator.startAnimating()
        webViewContainer.alpha = 0.5
    }
    
    private func hideLoadingState() {
        loadingIndicator.stopAnimating()
        webViewContainer.alpha = 1.0
    }
}
```

#### 4. 智能内容路由

```swift
class ContentRenderingRouter {
    private let complexityAnalyzer = ContentComplexityAnalyzer()
    private let performanceMonitor = RenderingPerformanceMonitor()
    
    func determineRenderingStrategy(for content: MessageContent) -> RenderingStrategy {
        let complexity = complexityAnalyzer.analyzeComplexity(content)
        let deviceCapability = assessDeviceCapability()
        let currentLoad = performanceMonitor.getCurrentLoad()
        
        return selectOptimalStrategy(
            complexity: complexity,
            deviceCapability: deviceCapability,
            currentLoad: currentLoad
        )
    }
    
    private func selectOptimalStrategy(
        complexity: ContentComplexity,
        deviceCapability: DeviceCapability,
        currentLoad: SystemLoad
    ) -> RenderingStrategy {
        
        switch (complexity, deviceCapability, currentLoad) {
        case (.simple, _, _):
            return .nativeText
            
        case (.moderate, .high, .low), (.moderate, .high, .medium):
            return .markdown
            
        case (.complex, .high, .low):
            return .webView
            
        case (.complex, .medium, _), (.complex, .high, .high):
            return .hybridRendering
            
        default:
            return .fallbackText
        }
    }
}

enum RenderingStrategy {
    case nativeText
    case markdown
    case webView
    case hybridRendering
    case fallbackText
    
    var cellType: MessageCellConfigurable.Type {
        switch self {
        case .nativeText:
            return TextMessageCell.self
        case .markdown:
            return MarkdownMessageCell.self
        case .webView:
            return WebViewMessageCell.self
        case .hybridRendering:
            return HybridRenderingCell.self
        case .fallbackText:
            return FallbackTextCell.self
        }
    }
}
```

### 阶段三总结

**实现成果：**
- 完整的WebView渲染引擎
- 高性能的WebView池管理
- 智能的内容渲染路由
- 流畅的动态高度计算
- 完善的错误处理和降级机制

**技术优势：**
- 支持复杂内容渲染（数学公式、图表、交互式内容）
- 优秀的性能表现和用户体验
- 高度可扩展的架构设计
- 完善的缓存和优化机制

## 演进过渡策略

### 1. 平滑升级机制

```swift
class VersionMigrationManager {
    private let currentVersion: AppVersion
    private let migrationStrategies: [MigrationStrategy]
    
    func performMigration(from oldVersion: AppVersion, to newVersion: AppVersion) -> AnyPublisher<MigrationResult, MigrationError> {
        let applicableStrategies = migrationStrategies.filter { strategy in
            strategy.canMigrate(from: oldVersion, to: newVersion)
        }
        
        return Publishers.Sequence(sequence: applicableStrategies)
            .flatMap { strategy in
                strategy.performMigration()
            }
            .collect()
            .map { results in
                MigrationResult(completedMigrations: results)
            }
            .eraseToAnyPublisher()
    }
}

protocol MigrationStrategy {
    func canMigrate(from oldVersion: AppVersion, to newVersion: AppVersion) -> Bool
    func performMigration() -> AnyPublisher<MigrationStepResult, MigrationError>
}

class Stage1ToStage2Migration: MigrationStrategy {
    func performMigration() -> AnyPublisher<MigrationStepResult, MigrationError> {
        return Future { promise in
            // 迁移消息类型定义
            self.migrateMessageTypes()
            
            // 更新Cell注册
            self.updateCellRegistrations()
            
            // 迁移现有消息数据
            self.migrateExistingMessages()
            
            promise(.success(.stage1ToStage2Completed))
        }
        .eraseToAnyPublisher()
    }
}
```

### 2. 功能特性开关

```swift
class FeatureToggleManager {
    private var toggles: [String: Bool] = [:]
    
    func isEnabled(_ feature: Feature) -> Bool {
        return toggles[feature.key] ?? feature.defaultValue
    }
    
    func enable(_ feature: Feature) {
        toggles[feature.key] = true
        notifyFeatureChange(feature, enabled: true)
    }
    
    func disable(_ feature: Feature) {
        toggles[feature.key] = false
        notifyFeatureChange(feature, enabled: false)
    }
}

enum Feature {
    case markdownRendering
    case webViewRendering
    case mathFormulas
    case chartSupport
    case interactiveContent
    
    var key: String {
        switch self {
        case .markdownRendering: return "markdown_rendering"
        case .webViewRendering: return "webview_rendering"
        case .mathFormulas: return "math_formulas"
        case .chartSupport: return "chart_support"
        case .interactiveContent: return "interactive_content"
        }
    }
    
    var defaultValue: Bool {
        switch self {
        case .markdownRendering: return true
        case .webViewRendering: return false // 默认关闭，需要手动开启
        case .mathFormulas: return false
        case .chartSupport: return false
        case .interactiveContent: return false
        }
    }
}
```

### 3. 性能监控和自动降级

```swift
class PerformanceDegradationManager {
    private let performanceThresholds = PerformanceThresholds()
    private let degradationStrategies: [DegradationStrategy] = [
        DisableAnimationsDegradation(),
        ReduceRenderingQualityDegradation(),
        FallbackToSimpleRenderingDegradation()
    ]
    
    func monitorAndDegrade() {
        Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
            self?.checkPerformanceAndDegrade()
        }
    }
    
    private func checkPerformanceAndDegrade() {
        let currentMetrics = gatherPerformanceMetrics()
        
        if currentMetrics.memoryUsage > performanceThresholds.maxMemoryUsage {
            applyDegradation(.memoryPressure)
        }
        
        if currentMetrics.frameRate < performanceThresholds.minFrameRate {
            applyDegradation(.lowFrameRate)
        }
        
        if currentMetrics.renderingTime > performanceThresholds.maxRenderingTime {
            applyDegradation(.slowRendering)
        }
    }
}
```

## 总结

本三阶段演进方案提供了从基础聊天功能到高级渲染能力的完整升级路径：

1. **阶段一**：建立坚实的基础架构，支持基本的聊天功能
2. **阶段二**：引入Markdown渲染，提升富文本展示能力
3. **阶段三**：集成WebView，实现复杂内容的完美渲染

每个阶段都有明确的技术目标、实现方案和过渡策略，确保系统能够平滑演进，同时保持高性能和良好的用户体验。