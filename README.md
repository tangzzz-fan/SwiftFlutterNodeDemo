# FlutterPluginDemo

一个综合性的Flutter插件开发演示项目，包含多个模块和完整的技术文档。

## 项目结构

```
FlutterPluginDemo/
├── backend_module/          # 后端服务模块 (Node.js + TypeScript)
├── flutter_module/          # Flutter应用模块
├── swiftui_module/          # SwiftUI/React Native模块
└── docs/                    # 技术文档
    ├── ai-chat-architecture/        # AI聊天架构设计
    ├── flutter-best-practices/     # Flutter最佳实践
    └── hybrid-mall-dynamic-delivery/ # 混合商城动态交付方案
```

## 模块说明

### Backend Module
- 基于Node.js + TypeScript的后端服务
- 提供用户认证、设备管理等API
- 支持WebSocket实时通信

### Flutter Module  
- Flutter应用主模块
- 包含BMS实时监控、数据可视化等功能
- 使用Riverpod 3.x进行状态管理

### SwiftUI Module
- 原生iOS/React Native混合开发模块
- 演示跨平台开发最佳实践

### Docs
- **AI Chat Architecture**: AI聊天系统架构设计文档
- **Flutter Best Practices**: Flutter开发最佳实践指南
- **Hybrid Mall Dynamic Delivery**: 混合商城动态交付技术方案

## 快速开始

1. 克隆项目
```bash
git clone <repository-url>
cd FlutterPluginDemo
```

2. 启动各个模块
```bash
# 启动后端服务
cd backend_module && npm install && npm start

# 启动Flutter应用
cd flutter_module && flutter pub get && flutter run

# 启动SwiftUI模块
cd swiftui_module && npm install && npm run ios
```

## 技术栈

- **Frontend**: Flutter, SwiftUI, React Native
- **Backend**: Node.js, TypeScript, Express
- **Database**: 支持多种数据库
- **State Management**: Riverpod 3.x
- **Real-time**: WebSocket, SSE
- **Documentation**: Markdown

## 贡献指南

欢迎提交Issue和Pull Request来改进项目。

## 许可证

MIT License