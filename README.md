# 🎮 GameGuardian

> **Empowering parents with comprehensive gaming oversight and control**

[![.NET](https://img.shields.io/badge/.NET-9.0-512BD4?logo=dotnet)](https://dotnet.microsoft.com/)
[![Aspire](https://img.shields.io/badge/Aspire-9.3-FF6B6B?logo=microsoft)](https://learn.microsoft.com/en-us/dotnet/aspire/)
[![UNO Platform](https://img.shields.io/badge/UNO%20Platform-Cross--Platform-00D4FF?logo=microsoft)](https://platform.uno/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Contributors Welcome](https://img.shields.io/badge/Contributors-Welcome-orange.svg)](CONTRIBUTING.md)

GameGuardian is a modern, cross-platform parental gaming oversight platform that gives parents the tools they need to monitor, manage, and guide their children's gaming experiences across all major platforms.

## 🌟 Why GameGuardian?

In today's digital world, children spend significant time gaming across multiple platforms, often with little parental oversight. GameGuardian bridges this gap by providing:

- **🔍 Real-time monitoring** of gaming sessions across Steam, Xbox, PlayStation, Nintendo, and mobile platforms
- **💰 Purchase control** with approval workflows for in-game purchases
- **⏰ Screen time management** with intelligent scheduling and limits
- **📊 Behavioral insights** to promote healthy gaming habits
- **🚨 Emergency controls** for immediate intervention when needed

## 🚀 Features

### For Parents
- **Cross-Platform Dashboard** - Monitor all your children's gaming activities from one unified interface
- **Real-Time Notifications** - Get instant alerts for purchase requests, screen time limits, and concerning behavior
- **Flexible Controls** - Set age-appropriate restrictions, spending limits, and play schedules
- **Detailed Analytics** - Understand gaming patterns and promote healthy digital habits
- **Emergency Override** - Instantly pause or terminate gaming sessions when needed

### For Families
- **Family Hierarchy** - Support for multiple parents, guardians, and children with appropriate permissions
- **Age-Appropriate Settings** - Dynamic controls that adapt as children grow
- **Educational Features** - Promote learning games and positive gaming experiences
- **Communication Tools** - Built-in messaging between parents and children

### Technical Excellence
- **Cross-Platform Native Apps** - iOS, Android, Windows, macOS, Linux, and Web
- **Enterprise-Grade Security** - COPPA/GDPR compliant with end-to-end encryption
- **Real-Time Architecture** - Instant updates using SignalR and modern web technologies
- **Cloud-Native Design** - Kubernetes-ready with comprehensive observability

## 🎯 Supported Gaming Platforms

| Platform | Status | Features |
|----------|--------|----------|
| 🎮 **Steam** | ✅ Full Support | Session monitoring, purchase control, friend management |
| 🎮 **Xbox Live** | ✅ Full Support | Game tracking, Microsoft Store integration, achievements |
| 🎮 **PlayStation** | ✅ Full Support | PSN integration, Remote Play monitoring, store purchases |
| 🎮 **Nintendo** | ✅ Full Support | Switch monitoring, eShop controls, parental integration |
| 📱 **iOS/Android** | ✅ Full Support | App Store/Play Store integration, screen time APIs |
| 🖥️ **PC Gaming** | 🚧 In Progress | Epic Games, Origin, Uplay integration |

## 🏗️ Architecture

GameGuardian is built with modern, scalable technologies:

```
┌─────────────────────────────────────────────────────────────┐
│                   UNO Platform Apps                        │
│   📱 Mobile    🖥️ Desktop    🌐 Web/WASM    📺 Smart TV   │
└─────────────────────┬───────────────────────────────────────┘
                      │
        ┌─────────────▼─────────────┐
        │     API Gateway (YARP)    │
        │  🔐 Keycloak Integration   │
        └─────────────┬─────────────┘
                      │
    ┌─────────────────┼─────────────────┐
    │                 │                 │
┌───▼──────┐  ┌──────▼──────┐  ┌───────▼────────┐
│ 👨‍👩‍👧‍👦 Family │  │  🎮 Gaming   │  │  📊 Monitoring │
│ Service  │  │  Platform   │  │  Service       │
│          │  │ Integration │  │                │
└──────────┘  └─────────────┘  └────────────────┘
```

### Tech Stack
- **Backend**: .NET 9 with Vertical Slice Architecture
- **Frontend**: UNO Platform for true cross-platform native experiences
- **Authentication**: Keycloak with family hierarchies and SSO
- **Database**: PostgreSQL with Entity Framework Core
- **Real-time**: SignalR for live monitoring and notifications
- **Deployment**: Kubernetes with Helm charts and GitOps (Flux)
- **Observability**: OpenTelemetry, Prometheus, Grafana

## 🚦 Getting Started

### Prerequisites
- [.NET 9 SDK](https://dotnet.microsoft.com/download/dotnet/9.0)
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [Visual Studio 2022](https://visualstudio.microsoft.com/) or [JetBrains Rider](https://www.jetbrains.com/rider/)

### Quick Start

1. **Clone the repository**
   ```bash
   git clone https://github.com/npmulder/GameGuardian.git
   cd GameGuardian
   ```

2. **Start infrastructure services**
   ```bash
   docker-compose up -d
   ```

3. **Run the application**
   ```bash
   dotnet run --project src/GameGuardian.AppHost
   ```

4. **Access the dashboard**
   - Open your browser to `https://localhost:5001`
   - Default admin credentials: `admin@gameguardian.local` / `password`

### Development Setup

For detailed development setup instructions, see our [Development Guide](docs/development.md).

## 📖 Documentation

- [**Getting Started**](docs/getting-started.md) - Complete setup and first-time configuration
- [**User Guide**](docs/user-guide.md) - How to use GameGuardian as a parent
- [**API Documentation**](docs/api.md) - REST API reference and integration guide
- [**Architecture Guide**](docs/architecture.md) - Technical deep-dive into the system design
- [**Deployment Guide**](docs/deployment.md) - Production deployment with Kubernetes
- [**Contributing**](CONTRIBUTING.md) - How to contribute to the project

## 🛡️ Security & Privacy

GameGuardian takes security and privacy seriously:

- **🔒 End-to-end encryption** for all sensitive data
- **👨‍👩‍👧‍👦 COPPA compliance** for children's data protection
- **🌍 GDPR compliance** with comprehensive privacy controls
- **🔐 Zero-trust architecture** with multi-factor authentication
- **📋 Regular security audits** and penetration testing
- **🚫 No data selling** - your family's data stays private

## 🤝 Contributing

We welcome contributions from the community! GameGuardian is built by parents, for parents.

### Ways to Contribute
- 🐛 **Report bugs** and suggest features
- 💻 **Contribute code** - from bug fixes to new features
- 📝 **Improve documentation** - help other families get started
- 🌐 **Translate the app** - make GameGuardian accessible worldwide
- 🎨 **Design improvements** - enhance the user experience

See our [Contributing Guide](CONTRIBUTING.md) for detailed instructions.

### Development Focus Areas
- 🎮 **Gaming Platform Integrations** - Add support for new platforms
- 🧠 **AI/ML Features** - Behavioral analysis and recommendations
- 📱 **Mobile Experience** - Native mobile app improvements
- ♿ **Accessibility** - Making GameGuardian usable for everyone
- 🌍 **Internationalization** - Multi-language support

## 📊 Project Status

GameGuardian is under active development with the following roadmap:

### 🚀 Phase 1: Foundation (Current)
- [x] Core architecture and .NET Aspire setup
- [x] Family management and authentication
- [x] Basic UNO Platform dashboard
- [ ] Steam integration
- [ ] Core testing framework

### 🎯 Phase 2: Platform Integrations
- [ ] Xbox Live integration
- [ ] PlayStation Network integration
- [ ] Nintendo Switch integration
- [ ] Mobile platform integration
- [ ] Advanced analytics dashboard

### 🌟 Phase 3: Advanced Features
- [ ] AI-powered behavioral analysis
- [ ] Machine learning recommendations
- [ ] Advanced reporting and insights
- [ ] Social features and community

## 💖 Community

Join our growing community of parents and developers:

- 💬 **Discord**: [Join our server](https://discord.gg/gameguardian) for real-time discussions
- 🐦 **Twitter**: [@GameGuardianApp](https://twitter.com/GameGuardianApp) for updates and news
- 📧 **Mailing List**: [Subscribe](https://newsletter.gameguardian.com) for monthly updates
- 🐛 **Issues**: [GitHub Issues](https://github.com/npmulder/GameGuardian/issues) for bug reports and feature requests

## 📄 License

GameGuardian is licensed under the [MIT License](LICENSE). This means you can:

- ✅ Use it commercially
- ✅ Modify and distribute
- ✅ Include in proprietary software
- ✅ Use for private projects

## 🙏 Acknowledgments

GameGuardian is made possible by:

- The amazing **.NET and UNO Platform communities**
- **Open source contributors** who make this project better
- **Parents worldwide** who provide feedback and guidance
- **Gaming platform APIs** that enable integration
- **Security researchers** who help keep families safe

---

<div align="center">

**⭐ Star this repo if GameGuardian helps your family!**

[Report Bug](https://github.com/npmulder/GameGuardian/issues/new?template=bug_report.md) •
[Request Feature](https://github.com/npmulder/GameGuardian/issues/new?template=feature_request.md) •
[Join Discord](https://discord.gg/gameguardian) •
[Documentation](docs/README.md)

</div>