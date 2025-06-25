## 🔧 Detailed Implementation Guide

### Vertical Slice Architecture Overview

**Vertical Slice Architecture** organizes code by business features rather than technical layers. Each slice contains everything needed for a specific feature: request/response models, validation, business logic, data access, and the endpoint definition.

**Benefits for GameGuardian**:
- **Feature-focused development**: Each gaming feature is self-contained
- **Team autonomy**: Different developers can work on different slices independently
- **Easier testing**: Each slice can be tested in isolation
- **Better maintainability**: Changes to family management don't affect gaming integrations
- **Faster onboarding**: New developers can understand one slice at a time

### Backend Services Implementation

#### 1. API Gateway (GameGuardian.Api)
**Purpose**: Central entry point with authentication, rate limiting, and routing

**Key Components**:
```csharp
// Program.cs - YARP configuration with Vertical Slice routing
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = keycloakConfig.Authority;
        options.RequireHttpsMetadata = true;
        options.Audience = keycloakConfig.Audience;
    });

// Rate limiting per user
builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy("PerUserPolicy", context =>
    {
        var userId = context.User?.FindFirst("sub")?.Value ?? "anonymous";
        return RateLimitPartition.CreateFixedWindow(userId, _ => 
            new FixedWindowRateLimiterOptions
            {
                Window = TimeSpan.FromMinutes(1),
                PermitLimit = 100
            });
    });
});
```

**Features**:
- JWT token validation with Keycloak
- User-specific rate limiting
- Request/response logging with OpenTelemetry
- Health checks for downstream services
- CORS configuration for web clients

#### 2. Family Service with Vertical Slices
**Purpose**: Manage family structures, relationships, and permissions

**Example Slice: Create Family**
```csharp
// Features/CreateFamily/CreateFamilyCommand.cs
public record CreateFamilyCommand(
    string FamilyName,
    string ParentFirstName,
    string ParentLastName,
    string ParentEmail,
    DateTime ParentBirthDate
) : IRequest<Result<CreateFamilyResponse>>;

public record CreateFamilyResponse(
    Guid FamilyId,
    string FamilyName,
    Guid ParentId
);

// Features/CreateFamily/CreateFamilyValidator.cs
public class CreateFamilyValidator : AbstractValidator<CreateFamilyCommand>
{
    public CreateFamilyValidator()
    {
        RuleFor(x => x.FamilyName)
            .NotEmpty()
            .MaximumLength(100);
            
        RuleFor(x => x.ParentEmail)
            .NotEmpty()
            .EmailAddress();
            
        RuleFor(x => x.ParentBirthDate)
            .Must(BeValidAdultAge)
            .WithMessage("Parent must be at least 18 years old");
    }
    
    private bool BeValidAdultAge(DateTime birthDate)
    {
        return DateTime.Today.Year - birthDate.Year >= 18;
    }
}

// Features/CreateFamily/CreateFamilyHandler.cs
public class CreateFamilyHandler : IRequestHandler<CreateFamilyCommand, Result<CreateFamilyResponse>>
{
    private readonly FamilyDbContext _context;
    private readonly IKeycloakService _keycloakService;
    private readonly ILogger<CreateFamilyHandler> _logger;
    
    public CreateFamilyHandler(
        FamilyDbContext context,
        IKeycloakService keycloakService,
        ILogger<CreateFamilyHandler> logger)
    {
        _context = context;
        _keycloakService = keycloakService;
        _logger = logger;
    }
    
    public async Task<Result<CreateFamilyResponse>> Handle(
        CreateFamilyCommand request, 
        CancellationToken cancellationToken)
    {
        using var activity = ActivitySource.StartActivity("CreateFamily.Handle");
        activity?.SetTag("family.name", request.FamilyName);
        
        try
        {
            // Create parent in Keycloak first
            var keycloakUserId = await _keycloakService.CreateParentUserAsync(
                request.ParentEmail,
                request.ParentFirstName,
                request.ParentLastName);
                
            if (keycloakUserId.IsFailure)
            {
                return Result.Failure<CreateFamilyResponse>(keycloakUserId.Error);
            }
            
            // Create family and parent in our database
            var family = new Family
            {
                Id = Guid.NewGuid(),
                Name = request.FamilyName,
                CreatedAt = DateTime.UtcNow
            };
            
            var parent = new FamilyMember
            {
                Id = Guid.NewGuid(),
                FamilyId = family.Id,
                KeycloakUserId = keycloakUserId.Value,
                Role = FamilyRole.Parent,
                FirstName = request.ParentFirstName,
                LastName = request.ParentLastName,
                Email = request.ParentEmail,
                BirthDate = request.ParentBirthDate
            };
            
            _context.Families.Add(family);
            _context.FamilyMembers.Add(parent);
            
            await _context.SaveChangesAsync(cancellationToken);
            
            _logger.LogInformation("Created family {FamilyId} with parent {ParentId}", 
                family.Id, parent.Id);
                
            return Result.Success(new CreateFamilyResponse(
                family.Id,
                family.Name,
                parent.Id));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create family {FamilyName}", request.FamilyName);
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            return Result.Failure<CreateFamilyResponse>("Failed to create family");
        }
    }
}

// Features/CreateFamily/CreateFamilyEndpoint.cs
public class CreateFamilyEndpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost("/api/families", async (
            CreateFamilyCommand command,
            ISender sender,
            CancellationToken cancellationToken) =>
        {
            var result = await sender.Send(command, cancellationToken);
            
            return result.IsSuccess
                ? Results.Created($"/api/families/{result.Value.FamilyId}", result.Value)
                : Results.BadRequest(result.Error);
        })
        .WithName("CreateFamily")
        .WithTags("Families")
        .RequireAuthorization()
        .Produces<CreateFamilyResponse>(StatusCodes.Status201Created)
        .Produces<string>(StatusCodes.Status400BadRequest);
    }
}
```

**Example Slice: Add Child to Family**
```csharp
// Features/AddChild/AddChildCommand.cs
public record AddChildCommand(
    Guid FamilyId,
    string ChildFirstName,
    string ChildLastName,
    DateTime ChildBirthDate,
    string? ChildEmail, // Optional for older children
    ChildSettingsDto Settings
) : IRequest<Result<AddChildResponse>>;

public record ChildSettingsDto(
    int MaxDailyScreenTimeMinutes,
    decimal MonthlySpendingLimit,
    List<string> AllowedContentRatings,
    bool RequireApprovalForPurchases,
    List<TimeSlotDto> AllowedPlayTimes
);

// Features/AddChild/AddChildHandler.cs
public class AddChildHandler : IRequestHandler<AddChildCommand, Result<AddChildResponse>>
{
    private readonly FamilyDbContext _context;
    private readonly IKeycloakService _keycloakService;
    private readonly ILogger<AddChildHandler> _logger;
    
    public async Task<Result<AddChildResponse>> Handle(
        AddChildCommand request,
        CancellationToken cancellationToken)
    {
        using var activity = ActivitySource.StartActivity("AddChild.Handle");
        
        // Verify family exists and user has permission
        var family = await _context.Families
            .Include(f => f.Members)
            .FirstOrDefaultAsync(f => f.Id == request.FamilyId, cancellationToken);
            
        if (family == null)
            return Result.Failure<AddChildResponse>("Family not found");
            
        // Calculate child's age for appropriate settings
        var childAge = DateTime.Today.Year - request.ChildBirthDate.Year;
        
        // Create child in Keycloak if they're old enough for their own account
        string? keycloakUserId = null;
        if (childAge >= 13 && !string.IsNullOrEmpty(request.ChildEmail))
        {
            var keycloakResult = await _keycloakService.CreateChildUserAsync(
                request.ChildEmail,
                request.ChildFirstName,
                request.ChildLastName,
                childAge);
                
            if (keycloakResult.IsFailure)
                return Result.Failure<AddChildResponse>(keycloakResult.Error);
                
            keycloakUserId = keycloakResult.Value;
        }
        
        var child = new FamilyMember
        {
            Id = Guid.NewGuid(),
            FamilyId = family.Id,
            KeycloakUserId = keycloakUserId,
            Role = FamilyRole.Child,
            FirstName = request.ChildFirstName,
            LastName = request.ChildLastName,
            Email = request.ChildEmail,
            BirthDate = request.ChildBirthDate,
            ChildSettings = new ChildSettings
            {
                MaxDailyScreenTimeMinutes = request.Settings.MaxDailyScreenTimeMinutes,
                MonthlySpendingLimit = request.Settings.MonthlySpendingLimit,
                AllowedContentRatings = request.Settings.AllowedContentRatings
                    .Select(r => Enum.Parse<ContentRating>(r)).ToList(),
                RequireApprovalForPurchases = request.Settings.RequireApprovalForPurchases,
                AllowedPlayTimes = request.Settings.AllowedPlayTimes
                    .Select(t => new TimeSlot(t.StartTime, t.EndTime, t.DaysOfWeek))
                    .ToList()
            }
        };
        
        _context.FamilyMembers.Add(child);
        await _context.SaveChangesAsync(cancellationToken);
        
        // Publish domain event for other services
        await _context.AddDomainEventAsync(new ChildAddedEvent(
            child.Id,
            family.Id,
            childAge,
            child.ChildSettings.MaxDailyScreenTimeMinutes));
        
        return Result.Success(new AddChildResponse(
            child.Id,
            child.FirstName,
            child.LastName,
            childAge));
    }
}
```

#### 3. Gaming Platform Integration with Vertical Slices
**Purpose**: Connect to Steam, Xbox, PlayStation, Nintendo, and mobile platforms

**Example Slice: Link Steam Account**
```csharp
// Features/LinkSteamAccount/LinkSteamAccountCommand.cs
public record LinkSteamAccountCommand(
    Guid ChildId,
    string SteamId,
    string SteamUsername
) : IRequest<Result<LinkSteamAccountResponse>>;

// Features/LinkSteamAccount/LinkSteamAccountHandler.cs
public class LinkSteamAccountHandler : IRequestHandler<LinkSteamAccountCommand, Result<LinkSteamAccountResponse>>
{
    private readonly GamingDbContext _context;
    private readonly ISteamService _steamService;
    private readonly IFamilyService _familyService;
    
    public async Task<Result<LinkSteamAccountResponse>> Handle(
        LinkSteamAccountCommand request,
        CancellationToken cancellationToken)
    {
        using var activity = ActivitySource.StartActivity("LinkSteamAccount.Handle");
        activity?.SetTag("child.id", request.ChildId.ToString());
        activity?.SetTag("steam.id", request.SteamId);
        
        // Verify child exists and user has permission to link accounts
        var child = await _familyService.GetChildAsync(request.ChildId);
        if (child == null)
            return Result.Failure<LinkSteamAccountResponse>("Child not found");
            
        // Verify Steam account exists and is valid
        var steamProfile = await _steamService.GetPlayerSummaryAsync(request.SteamId);
        if (steamProfile.IsFailure)
            return Result.Failure<LinkSteamAccountResponse>("Invalid Steam account");
            
        // Check if Steam account is already linked to another child
        var existingLink = await _context.GamingAccounts
            .FirstOrDefaultAsync(ga => ga.PlatformUserId == request.SteamId 
                && ga.Platform == GamingPlatform.Steam, cancellationToken);
                
        if (existingLink != null)
            return Result.Failure<LinkSteamAccountResponse>("Steam account already linked");
            
        var gamingAccount = new GamingAccount
        {
            Id = Guid.NewGuid(),
            ChildId = request.ChildId,
            Platform = GamingPlatform.Steam,
            PlatformUserId = request.SteamId,
            Username = request.SteamUsername,
            ProfileData = JsonSerializer.Serialize(steamProfile.Value),
            LinkedAt = DateTime.UtcNow,
            IsActive = true
        };
        
        _context.GamingAccounts.Add(gamingAccount);
        await _context.SaveChangesAsync(cancellationToken);
        
        // Start monitoring this account
        await _steamService.StartMonitoringAsync(request.SteamId, request.ChildId);
        
        return Result.Success(new LinkSteamAccountResponse(
            gamingAccount.Id,
            request.SteamId,
            request.SteamUsername));
    }
}

// Features/LinkSteamAccount/LinkSteamAccountEndpoint.cs
public class LinkSteamAccountEndpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapPost("/api/gaming/steam/link", async (
            LinkSteamAccountCommand command,
            ISender sender,
            CancellationToken cancellationToken) =>
        {
            var result = await sender.Send(command, cancellationToken);
            
            return result.IsSuccess
                ? Results.Ok(result.Value)
                : Results.BadRequest(result.Error);
        })
        .WithName("LinkSteamAccount")
        .WithTags("Gaming")
        .RequireAuthorization("ParentRole")
        .Produces<LinkSteamAccountResponse>()
        .Produces<string>(StatusCodes.Status400BadRequest);
    }
}
```

**Example Slice: Intercept Purchase**
```csharp
// Features/InterceptPurchase/InterceptPurchaseCommand.cs
public record InterceptPurchaseCommand(
    Guid ChildId,
    string Platform,
    string GameTitle,
    string ItemName,
    decimal Amount,
    string Currency,
    string ItemDescription,
    Dictionary<string, object> PlatformMetadata
) : IRequest<Result<InterceptPurchaseResponse>>;

// Features/InterceptPurchase/InterceptPurchaseHandler.cs
public class InterceptPurchaseHandler : IRequestHandler<InterceptPurchaseCommand, Result<InterceptPurchaseResponse>>
{
    private readonly GamingDbContext _context;
    private readonly IFamilyService _familyService;
    private readonly IPurchaseApprovalService _approvalService;
    private readonly IParentNotificationService _notificationService;
    
    public async Task<Result<InterceptPurchaseResponse>> Handle(
        InterceptPurchaseCommand request,
        CancellationToken cancellationToken)
    {
        using var activity = ActivitySource.StartActivity("InterceptPurchase.Handle");
        
        // Get child and their settings
        var child = await _familyService.GetChildWithSettingsAsync(request.ChildId);
        if (child == null)
            return Result.Failure<InterceptPurchaseResponse>("Child not found");
            
        // Check if purchase requires approval
        var requiresApproval = ShouldRequireApproval(child.Settings, request.Amount);
        
        if (!requiresApproval)
        {
            // Auto-approve small purchases within limits
            return Result.Success(new InterceptPurchaseResponse(
                PurchaseStatus.AutoApproved,
                null,
                "Purchase automatically approved"));
        }
        
        // Create pending approval request
        var approvalRequest = new PurchaseApproval
        {
            Id = Guid.NewGuid(),
            ChildId = request.ChildId,
            Platform = request.Platform,
            GameTitle = request.GameTitle,
            ItemName = request.ItemName,
            Amount = request.Amount,
            Currency = request.Currency,
            Description = request.ItemDescription,
            RequestedAt = DateTime.UtcNow,
            Status = ApprovalStatus.Pending,
            PlatformMetadata = JsonSerializer.Serialize(request.PlatformMetadata)
        };
        
        _context.PurchaseApprovals.Add(approvalRequest);
        await _context.SaveChangesAsync(cancellationToken);
        
        // Notify all parents in the family
        var family = await _familyService.GetFamilyAsync(child.FamilyId);
        var parents = family.Members.Where(m => m.Role == FamilyRole.Parent);
        
        foreach (var parent in parents)
        {
            await _notificationService.NotifyPurchaseApprovalRequired(
                parent.Id,
                approvalRequest);
        }
        
        return Result.Success(new InterceptPurchaseResponse(
            PurchaseStatus.PendingApproval,
            approvalRequest.Id,
            "Purchase requires parental approval"));
    }
    
    private bool ShouldRequireApproval(ChildSettings settings, decimal amount)
    {
        if (!settings.RequireApprovalForPurchases)
            return false;
            
        // Check daily spending limit
        var todaySpending = GetTodaySpending(settings.ChildId);
        if (todaySpending + amount > settings.DailySpendingLimit)
            return true;
            
        // Check individual purchase threshold
        return amount > settings.AutoApprovalThreshold;
    }
}
```

#### 4. Real-Time Monitoring Service with Vertical Slices
**Purpose**: Live gaming session tracking and parent-child communication

**Example Slice: Get Live Gaming Status**
```csharp
// Features/GetLiveStatus/GetLiveStatusQuery.cs
public record GetLiveStatusQuery(
    Guid FamilyId
) : IRequest<Result<LiveStatusResponse>>;

public record LiveStatusResponse(
    List<ChildLiveStatus> Children,
    FamilyOverview Overview
);

public record ChildLiveStatus(
    Guid ChildId,
    string Name,
    bool IsCurrentlyGaming,
    string? CurrentGame,
    string? Platform,
    TimeSpan SessionDuration,
    TimeSpan RemainingScreenTime,
    List<PendingApproval> PendingApprovals
);

// Features/GetLiveStatus/GetLiveStatusHandler.cs
public class GetLiveStatusHandler : IRequestHandler<GetLiveStatusQuery, Result<LiveStatusResponse>>
{
    private readonly MonitoringDbContext _context;
    private readonly IGamingPlatformService _gamingService;
    private readonly IMemoryCache _cache;
    
    public async Task<Result<LiveStatusResponse>> Handle(
        GetLiveStatusQuery request,
        CancellationToken cancellationToken)
    {
        using var activity = ActivitySource.StartActivity("GetLiveStatus.Handle");
        
        // Get family members
        var family = await _context.Families
            .Include(f => f.Members.Where(m => m.Role == FamilyRole.Child))
            .FirstOrDefaultAsync(f => f.Id == request.FamilyId, cancellationToken);
            
        if (family == null)
            return Result.Failure<LiveStatusResponse>("Family not found");
            
        var childStatuses = new List<ChildLiveStatus>();
        
        foreach (var child in family.Members)
        {
            // Check# GameGuardian Project Documentation

## 🎯 Project Overview

**GameGuardian** is a cross-platform parental gaming oversight platform that addresses the growing concern of parents having little control or understanding of their children's gaming habits and in-game spending. Built with UNO Platform for cross-platform capabilities and powered by .NET 8 backend services, it provides comprehensive monitoring, control, and analytics for family gaming activities.

### Problem Statement
- Lack of oversight in online gaming
- Parents have little control over children's gaming habits
- Limited understanding of in-game spending patterns
- Need for real-time monitoring and intervention capabilities

### Solution
A companion dashboard for parents to:
- Monitor screen time and content types
- Control in-game purchases with approval workflows
- Set play schedules and content restrictions
- Review behavioral insights and analytics
- Enable emergency controls and overrides

## 🏗️ Technical Architecture

### Platform Stack
- **Frontend**: UNO Platform (iOS, Android, Windows, macOS, Linux, WebAssembly)
- **Backend**: .NET 8 with Vertical Slice Architecture
- **Authentication**: Keycloak with family hierarchies
- **Database**: PostgreSQL with Entity Framework Core
- **Real-time**: SignalR for live monitoring
- **Message Bus**: MassTransit with RabbitMQ
- **Caching**: Redis for session and data caching
- **Observability**: OpenTelemetry, Prometheus, Grafana
- **Deployment**: Kubernetes with Helm charts
- **CI/CD**: GitHub Actions with GitOps (Flux)

### System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                UNO Platform Apps                           │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐  │
│  │   Mobile    │ │   Desktop   │ │    Web/WASM         │  │
│  │ (iOS/Droid) │ │ (Win/Mac)   │ │   (Browser)         │  │
│  └─────────────┘ └─────────────┘ └─────────────────────┘  │
└─────────────────────┬───────────────────────────────────────┘
                      │
        ┌─────────────▼─────────────┐
        │     API Gateway (YARP)    │
        │  + Keycloak Integration   │
        └─────────────┬─────────────┘
                      │
    ┌─────────────────┼─────────────────┐
    │                 │                 │
┌───▼──────┐  ┌──────▼──────┐  ┌───────▼────────┐
│ Family   │  │  Gaming     │  │  Monitoring    │
│ Service  │  │  Platform   │  │  Service       │
│ (.NET 8) │  │ Integration │  │  (.NET 8)      │
└──────────┘  │  (.NET 8)   │  └────────────────┘
              └─────────────┘
```

## 🔐 Authentication & Security

### Family Hierarchy Model
```
Family Structure:
├── Parent/Guardian Account (Primary)
│   ├── Child Account 1 (Age 8) - Restricted
│   ├── Child Account 2 (Age 14) - Moderate
│   └── Child Account 3 (Age 17) - Supervised
├── Co-Parent Account (Secondary Admin)
└── Grandparent Account (Observer Only)
```

### Keycloak Configuration
- **Multi-Realm Architecture**: Each family can have isolated authentication
- **Custom Authenticators**: Hardware fingerprinting, behavioral biometrics
- **Age-Based Policies**: Dynamic permissions based on child age
- **Gaming Platform Integration**: Steam, Xbox, PlayStation, Nintendo account linking
- **Emergency Overrides**: Parent emergency access patterns

### Security Features
- COPPA/GDPR-K compliance
- Hardware fingerprinting for anti-cheat
- Behavioral biometrics analysis
- Encrypted communication channels
- Audit logging for all activities

## 🎮 Gaming Platform Integrations

### Supported Platforms
- **Steam**: Web API, Community API, Store API
- **Xbox Live**: REST API, Microsoft Store, Game Bar
- **PlayStation**: PSN API, PlayStation Store, Remote Play
- **Nintendo**: Account API, eShop, Parental Controls
- **Mobile**: iOS App Store (StoreKit), Google Play Billing

### Integration Capabilities
- Real-time session monitoring
- Purchase interception and approval
- Content rating verification
- Account linking and verification
- Anti-cheat and security measures

## 📱 UNO Platform Features

### Cross-Platform Targets
- **Mobile**: iOS and Android with touch-optimized UI
- **Desktop**: Windows, macOS, Linux with system tray integration
- **Web**: WebAssembly with PWA capabilities
- **Future**: Smart TV apps, wearable notifications

### Key Features
- Real-time family dashboard
- Live gaming session monitoring
- Purchase approval interface
- Schedule management
- Emergency controls
- Analytics and reporting

### Platform-Specific Optimizations
- **Mobile**: Push notifications, location services, quick actions
- **Desktop**: Keyboard shortcuts, multi-window support, system tray
- **Web**: Responsive design, offline capabilities, browser notifications

## 🚀 Development Phases

### Phase 1: Foundation (Issues #1-4, #8)
**Priority: High**
- [ ] Project setup and architecture foundation
- [ ] Keycloak configuration and family authentication
- [ ] Core backend services architecture
- [ ] UNO Platform parent dashboard development
- [ ] Testing framework and quality assurance

### Phase 2: Integrations & Deployment (Issues #5, #7)
**Priority: Medium**
- [ ] Gaming platform API integrations
- [ ] Kubernetes deployment and DevOps pipeline

### Phase 3: Advanced Features (Issue #6)
**Priority: Medium**
- [ ] Advanced analytics and AI behavioral analysis

## 🛠️ Development Guidelines

### Code Standards
- Clean Architecture with CQRS patterns
- SOLID principles and dependency injection
- Comprehensive unit and integration testing
- OpenTelemetry instrumentation for observability
- Secure coding practices for child safety

### Testing Strategy
- **Unit Tests**: xUnit with Moq for mocking
- **Integration Tests**: TestContainers for real dependencies
- **Advanced Testing**: OpenTelemetry-based integration testing
- **UI Tests**: Uno.UITest for cross-platform consistency
- **Performance Tests**: Load testing with NBomber
- **Security Tests**: OWASP ZAP scanning, compliance validation

### Development Environment
- Docker containers for local development
- PostgreSQL database with Entity Framework migrations
- Redis for caching and session management
- Keycloak for authentication testing
- OpenTelemetry for local observability

## 📊 Key Metrics & Analytics

### Parent Dashboard Metrics
- Real-time gaming session status
- Daily/weekly screen time summaries
- Purchase request notifications
- Behavioral trend analysis
- Content exposure reports

### Child Safety Analytics
- Age-appropriate content filtering
- Social interaction monitoring
- Potential risk indicators
- Learning game engagement
- Physical activity correlation

### Technical Metrics
- API response times and availability
- SignalR connection stability
- Gaming platform integration health
- Security event monitoring
- Compliance audit trails

## 🎯 Success Criteria

### Functional Requirements
- [ ] Parents can monitor children's gaming across all major platforms
- [ ] Real-time purchase approval system prevents unauthorized spending
- [ ] Emergency controls can instantly terminate gaming sessions
- [ ] Analytics provide actionable insights for healthy gaming habits
- [ ] All features work consistently across mobile, desktop, and web

### Technical Requirements
- [ ] 99.9% uptime for critical monitoring features
- [ ] Sub-second response times for real-time features
- [ ] Secure handling of all family and gaming data
- [ ] Scalable architecture supporting thousands of families
- [ ] Comprehensive observability for production operations

### Compliance Requirements
- [ ] Full COPPA compliance for child data protection
- [ ] GDPR-K compliance for children's privacy rights
- [ ] Gaming platform terms of service adherence
- [ ] Secure authentication and authorization
- [ ] Audit trails for all family interactions

## 🔄 Getting Started

### Prerequisites
- .NET 8 SDK
- Docker and Docker Compose
- Node.js (for UNO Platform WebAssembly)
- Visual Studio 2022 or JetBrains Rider

### Initial Setup
1. Clone the repository: `git clone https://github.com/npmulder/GameGuardian.git`
2. Run `docker-compose up -d` for local services (PostgreSQL, Redis, Keycloak)
3. Execute database migrations: `dotnet ef database update`
4. Configure Keycloak realms and clients using the provided setup scripts
5. Set up gaming platform API credentials in user secrets
6. Build and run the UNO Platform solution
7. Access the parent dashboard on your preferred platform

### Repository Structure
```
GameGuardian/
├── src/
│   ├── Backend/
│   │   ├── GameGuardian.Api/              # API Gateway with YARP
│   │   ├── GameGuardian.Family.Api/       # Family management service
│   │   │   ├── Features/                  # Vertical slices
│   │   │   │   ├── CreateFamily/          # Create family slice
│   │   │   │   │   ├── CreateFamilyCommand.cs
│   │   │   │   │   ├── CreateFamilyHandler.cs
│   │   │   │   │   ├── CreateFamilyValidator.cs
│   │   │   │   │   └── CreateFamilyEndpoint.cs
│   │   │   │   ├── AddChild/              # Add child slice
│   │   │   │   │   ├── AddChildCommand.cs
│   │   │   │   │   ├── AddChildHandler.cs
│   │   │   │   │   ├── AddChildValidator.cs
│   │   │   │   │   └── AddChildEndpoint.cs
│   │   │   │   └── GetFamilyDashboard/    # Family dashboard slice
│   │   │   │       ├── GetFamilyDashboardQuery.cs
│   │   │   │       ├── GetFamilyDashboardHandler.cs
│   │   │   │       └── GetFamilyDashboardEndpoint.cs
│   │   │   ├── Domain/                    # Shared domain models
│   │   │   │   ├── Family.cs
│   │   │   │   ├── FamilyMember.cs
│   │   │   │   └── ChildSettings.cs
│   │   │   ├── Infrastructure/            # Data access and external services
│   │   │   │   ├── FamilyDbContext.cs
│   │   │   │   └── KeycloakService.cs
│   │   │   └── Program.cs
│   │   ├── GameGuardian.Gaming.Api/       # Gaming platform integrations
│   │   │   ├── Features/
│   │   │   │   ├── LinkSteamAccount/      # Steam account linking slice
│   │   │   │   │   ├── LinkSteamAccountCommand.cs
│   │   │   │   │   ├── LinkSteamAccountHandler.cs
│   │   │   │   │   ├── LinkSteamAccountValidator.cs
│   │   │   │   │   └── LinkSteamAccountEndpoint.cs
│   │   │   │   ├── StartGamingSession/    # Session tracking slice
│   │   │   │   │   ├── StartGamingSessionCommand.cs
│   │   │   │   │   ├── StartGamingSessionHandler.cs
│   │   │   │   │   └── StartGamingSessionEndpoint.cs
│   │   │   │   └── InterceptPurchase/     # Purchase interception slice
│   │   │   │       ├── InterceptPurchaseCommand.cs
│   │   │   │       ├── InterceptPurchaseHandler.cs
│   │   │   │       └── InterceptPurchaseEndpoint.cs
│   │   │   ├── Domain/
│   │   │   │   ├── GamingAccount.cs
│   │   │   │   ├── GameSession.cs
│   │   │   │   └── PurchaseRequest.cs
│   │   │   ├── Infrastructure/
│   │   │   │   ├── SteamService.cs
│   │   │   │   ├── XboxService.cs
│   │   │   │   └── PlayStationService.cs
│   │   │   └── Program.cs
│   │   ├── GameGuardian.Monitoring.Api/   # Real-time monitoring service
│   │   │   ├── Features/
│   │   │   │   ├── GetLiveStatus/         # Live status slice
│   │   │   │   ├── SendEmergencyStop/     # Emergency controls slice
│   │   │   │   └── GetBehaviorAnalysis/   # Behavior analysis slice
│   │   │   ├── Hubs/                      # SignalR hubs
│   │   │   │   └── ParentMonitoringHub.cs
│   │   │   └── Program.cs
│   │   ├── GameGuardian.Analytics.Api/    # Behavioral analytics service
│   │   │   ├── Features/
│   │   │   │   ├── GenerateWeeklyReport/  # Weekly reports slice
│   │   │   │   ├── AnalyzeBehavior/       # AI behavior analysis slice
│   │   │   │   └── PredictRisks/          # Risk prediction slice
│   │   │   └── Program.cs
│   │   └── GameGuardian.Shared/           # Shared libraries and contracts
│   │       ├── Contracts/                 # Request/Response DTOs
│   │       ├── Common/                    # Common utilities
│   │       └── Events/                    # Domain events
│   ├── Frontend/
│   │   ├── GameGuardian.Shared/           # Shared UNO Platform code
│   │   ├── GameGuardian.Mobile/           # iOS/Android projects
│   │   ├── GameGuardian.Windows/          # Windows desktop
│   │   ├── GameGuardian.WebAssembly/      # Web application
│   │   ├── GameGuardian.macOS/            # macOS desktop
│   │   └── GameGuardian.Skia.Linux/       # Linux desktop
│   └── Infrastructure/
│       ├── Keycloak/                      # Custom authenticators and themes
│       ├── Database/                      # Migrations and seeding
│       └── Docker/                        # Container configurations
├── tests/
│   ├── GameGuardian.Tests.Unit/           # Unit tests (per slice)
│   ├── GameGuardian.Tests.Integration/    # Integration tests with TestContainers
│   ├── GameGuardian.Tests.E2E/            # End-to-end tests
│   └── GameGuardian.Tests.Performance/    # Load and performance tests
├── deploy/
│   ├── kubernetes/                        # K8s manifests and Helm charts
│   ├── flux/                              # Flux GitOps configurations
│   │   ├── clusters/                      # Cluster configurations
│   │   │   ├── production/
│   │   │   │   ├── flux-system/           # Flux controllers
│   │   │   │   ├── sources/               # Git repositories and Helm repos
│   │   │   │   ├── releases/              # Helm releases
│   │   │   │   └── kustomizations/        # Kustomize configs
│   │   │   ├── staging/
│   │   │   └── development/
│   │   ├── apps/                          # Application definitions
│   │   │   ├── gameguardian/
│   │   │   │   ├── base/                  # Base configurations
│   │   │   │   ├── overlays/              # Environment overlays
│   │   │   │   │   ├── production/
│   │   │   │   │   ├── staging/
│   │   │   │   │   └── development/
│   │   │   │   └── releases/              # Helm release configs
│   │   │   └── infrastructure/            # Infrastructure apps (Redis, PostgreSQL)
│   │   └── infrastructure/                # Platform infrastructure (monitoring, ingress)
│   ├── terraform/                         # Infrastructure as Code
│   └── github-actions/                    # CI/CD workflows
├── docs/
│   ├── architecture/                      # Technical documentation
│   ├── api/                              # API documentation
│   └── user-guides/                      # User documentation
└── scripts/
    ├── setup/                            # Development setup scripts
    └── deployment/                       # Deployment automation
```

        foreach (var child in family.Members)
        {
            // Check cache first for performance
            var cacheKey = $"child_status_{child.Id}";
            var cachedStatus = _cache.Get<ChildLiveStatus>(cacheKey);
            
            if (cachedStatus != null)
            {
                childStatuses.Add(cachedStatus);
                continue;
            }
            
            // Get current gaming sessions
            var activeSessions = await _gamingService.GetActiveSessionsAsync(child.Id);
            var currentSession = activeSessions.FirstOrDefault();
            
            // Calculate remaining screen time
            var todayScreenTime = await GetTodayScreenTimeAsync(child.Id);
            var maxScreenTime = TimeSpan.FromMinutes(child.ChildSettings.MaxDailyScreenTimeMinutes);
            var remainingTime = maxScreenTime - todayScreenTime;
            
            // Get pending approvals
            var pendingApprovals = await _context.PurchaseApprovals
                .Where(pa => pa.ChildId == child.Id && pa.Status == ApprovalStatus.Pending)
                .Select(pa => new PendingApproval(pa.Id, pa.ItemName, pa.Amount, pa.Currency))
                .ToListAsync(cancellationToken);
            
            var status = new ChildLiveStatus(
                child.Id,
                $"{child.FirstName} {child.LastName}",
                currentSession != null,
                currentSession?.GameTitle,
                currentSession?.Platform,
                currentSession?.Duration ?? TimeSpan.Zero,
                remainingTime > TimeSpan.Zero ? remainingTime : TimeSpan.Zero,
                pendingApprovals);
                
            // Cache for 30 seconds to reduce load
            _cache.Set(cacheKey, status, TimeSpan.FromSeconds(30));
            childStatuses.Add(status);
        }
        
        var overview = new FamilyOverview(
            childStatuses.Count(c => c.IsCurrentlyGaming),
            childStatuses.Sum(c => c.PendingApprovals.Count),
            childStatuses.Count(c => c.RemainingScreenTime == TimeSpan.Zero));
        
        return Result.Success(new LiveStatusResponse(childStatuses, overview));
    }
    
    private async Task<TimeSpan> GetTodayScreenTimeAsync(Guid childId)
    {
        var today = DateTime.Today;
        var sessions = await _context.GameSessions
            .Where(s => s.ChildId == childId 
                && s.StartTime >= today 
                && s.StartTime < today.AddDays(1))
            .ToListAsync();
            
        return sessions.Aggregate(TimeSpan.Zero, (total, session) => 
            total + (session.EndTime?.Subtract(session.StartTime) ?? TimeSpan.Zero));
    }
}

// Features/GetLiveStatus/GetLiveStatusEndpoint.cs
public class GetLiveStatusEndpoint : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        app.MapGet("/api/monitoring/families/{familyId}/status", async (
            Guid familyId,
            ISender sender,
            CancellationToken cancellationToken) =>
        {
            var query = new GetLiveStatusQuery(familyId);
            var result = await sender.Send(query, cancellationToken);
            
            return result.IsSuccess
                ? Results.Ok(result.Value)
                : Results.BadRequest(result.Error);
        })
        .WithName("GetLiveStatus")
        .WithTags("Monitoring")
        .RequireAuthorization("ParentRole")
        .Produces<LiveStatusResponse>()
        .Produces<string>(StatusCodes.Status400BadRequest);
    }
}
```

**SignalR Hub for Real-Time Updates**
```csharp
// Hubs/ParentMonitoringHub.cs
[Authorize]
public class ParentMonitoringHub : Hub
{
    private readonly IFamilyService _familyService;
    private readonly IMonitoringService _monitoringService;
    private readonly ILogger<ParentMonitoringHub> _logger;
    
    public async Task JoinFamilyMonitoring(Guid familyId)
    {
        var parentId = Context.User.GetUserId();
        
        // Verify parent belongs to this family
        if (!await _familyService.IsParentOfFamily(parentId, familyId))
        {
            _logger.LogWarning("Unauthorized family monitoring attempt by {ParentId} for family {FamilyId}", 
                parentId, familyId);
            return;
        }
            
        await Groups.AddToGroupAsync(Context.ConnectionId, $"family_{familyId}");
        _logger.LogInformation("Parent {ParentId} joined monitoring for family {FamilyId}", 
            parentId, familyId);
    }
    
    [Authorize(Policy = "ParentRole")]
    public async Task SendEmergencyStop(Guid childId, string reason)
    {
        using var activity = ActivitySource.StartActivity("EmergencyStop.SignalR");
        activity?.SetTag("child.id", childId.ToString());
        
        var parentId = Context.User.GetUserId();
        
        // Verify parental authority
        if (!await _familyService.HasParentalAuthority(parentId, childId))
        {
            await Clients.Caller.SendAsync("Error", "Unauthorized action");
            return;
        }
        
        // Execute emergency stop
        var result = await _monitoringService.EmergencyStopAllSessions(childId, reason);
        
        if (result.IsSuccess)
        {
            // Notify all family members
            var familyId = await _familyService.GetFamilyIdForChild(childId);
            await Clients.Group($"family_{familyId}")
                .SendAsync("EmergencyStopExecuted", childId, reason, parentId);
                
            _logger.LogWarning("Emergency stop executed by {ParentId} for child {ChildId}: {Reason}",
                parentId, childId, reason);
        }
        else
        {
            await Clients.Caller.SendAsync("Error", result.Error);
        }
    }
    
    public async Task ApprovePurchase(Guid approvalId, bool approved, string? reason = null)
    {
        using var activity = ActivitySource.StartActivity("ApprovePurchase.SignalR");
        
        var parentId = Context.User.GetUserId();
        var result = await _monitoringService.ProcessPurchaseApproval(
            approvalId, parentId, approved, reason);
            
        if (result.IsSuccess)
        {
            var familyId = result.Value.FamilyId;
            await Clients.Group($"family_{familyId}")
                .SendAsync("PurchaseApprovalProcessed", result.Value);
        }
        else
        {
            await Clients.Caller.SendAsync("Error", result.Error);
        }
    }
    
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var parentId = Context.User?.GetUserId();
        _logger.LogInformation("Parent {ParentId} disconnected from monitoring", parentId);
        await base.OnDisconnectedAsync(exception);
    }
}
```

### Service Registration and Configuration
```csharp
// Program.cs - Family Service
var builder = WebApplication.CreateBuilder(args);

// Add Vertical Slice Architecture dependencies
builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));
builder.Services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
builder.Services.AddCarter();

// Add database context
builder.Services.AddDbContext<FamilyDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection"))
           .EnableDetailedErrors()
           .EnableSensitiveDataLogging(builder.Environment.IsDevelopment()));

// Add caching
builder.Services.AddStackExchangeRedisCache(options =>
    options.Configuration = builder.Configuration.GetConnectionString("Redis"));

// Add external services
builder.Services.AddScoped<IKeycloakService, KeycloakService>();
builder.Services.AddScoped<IFamilyService, FamilyService>();

// Add MassTransit for event publishing
builder.Services.AddMassTransit(x =>
{
    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host(builder.Configuration.GetConnectionString("RabbitMQ"));
        cfg.ConfigureEndpoints(context);
    });
});

// Add OpenTelemetry
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddJaegerExporter())
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddPrometheusExporter());

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

// Add Carter endpoints (includes all vertical slice endpoints)
app.MapCarter();

app.Run();
```

### Domain Models
```csharp
// Domain/Family.cs
public class Family
{
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
    public bool IsActive { get; set; } = true;
    
    // Navigation properties
    public List<FamilyMember> Members { get; set; } = new();
    public FamilySettings Settings { get; set; } = new();
    
    // Domain events (for event-driven architecture)
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    
    public void AddDomainEvent(IDomainEvent domainEvent)
    {
        _domainEvents.Add(domainEvent);
    }
    
    public void ClearDomainEvents()
    {
        _domainEvents.Clear();
    }
}

// Domain/FamilyMember.cs
public class FamilyMember
{
    public Guid Id { get; set; }
    public Guid FamilyId { get; set; }
    public string? KeycloakUserId { get; set; } // Null for young children without accounts
    public FamilyRole Role { get; set; }
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string? Email { get; set; }
    public DateTime BirthDate { get; set; }
    public DateTime CreatedAt { get; set; }
    public bool IsActive { get; set; } = true;
    
    // Navigation properties
    public Family Family { get; set; } = null!;
    public ChildSettings? ChildSettings { get; set; } // Only for children
    public List<GamingAccount> GamingAccounts { get; set; } = new();
    
    // Computed properties
    public int Age => DateTime.Today.Year - BirthDate.Year;
    public string FullName => $"{FirstName} {LastName}";
    public bool IsMinor => Age < 18;
}

// Domain/ChildSettings.cs
public class ChildSettings
{
    public Guid Id { get; set; }
    public Guid ChildId { get; set; }
    public int MaxDailyScreenTimeMinutes { get; set; }
    public decimal MonthlySpendingLimit { get; set; }
    public decimal DailySpendingLimit { get; set; }
    public decimal AutoApprovalThreshold { get; set; }
    public bool RequireApprovalForPurchases { get; set; }
    public bool AllowOnlineInteraction { get; set; }
    public List<ContentRating> AllowedContentRatings { get; set; } = new();
    public List<TimeSlot> AllowedPlayTimes { get; set; } = new();
    public List<string> BlockedGames { get; set; } = new();
    public List<string> BlockedCategories { get; set; } = new();
    
    // Navigation property
    public FamilyMember Child { get; set; } = null!;
}
```

### Shared Contracts and Events
```csharp
// Shared/Contracts/Results.cs
public class Result<T>
{
    public bool IsSuccess { get; private set; }
    public bool IsFailure => !IsSuccess;
    public T Value { get; private set; } = default!;
    public string Error { get; private set; } = string.Empty;
    
    private Result(bool isSuccess, T value, string error)
    {
        IsSuccess = isSuccess;
        Value = value;
        Error = error;
    }
    
    public static Result<T> Success(T value) => new(true, value, string.Empty);
    public static Result<T> Failure(string error) => new(false, default!, error);
}

// Shared/Events/ChildAddedEvent.cs
public record ChildAddedEvent(
    Guid ChildId,
    Guid FamilyId,
    int ChildAge,
    int MaxDailyScreenTimeMinutes
) : IDomainEvent;

// Shared/Events/GamingSessionStartedEvent.cs
public record GamingSessionStartedEvent(
    Guid SessionId,
    Guid ChildId,
    string GameTitle,
    string Platform,
    DateTime StartTime
) : IDomainEvent;

// Shared/Events/PurchaseInterceptedEvent.cs
public record PurchaseInterceptedEvent(
    Guid ApprovalId,
    Guid ChildId,
    string GameTitle,
    decimal Amount,
    string Currency,
    bool RequiresApproval
) : IDomainEvent;
```

## 🧪 Testing Strategy Implementation

### Vertical Slice Testing Approach
```csharp
// Tests/Unit/Features/CreateFamily/CreateFamilyHandlerTests.cs
public class CreateFamilyHandlerTests
{
    private readonly FamilyDbContext _context;
    private readonly Mock<IKeycloakService> _keycloakServiceMock;
    private readonly Mock<ILogger<CreateFamilyHandler>> _loggerMock;
    private readonly CreateFamilyHandler _handler;
    
    public CreateFamilyHandlerTests()
    {
        var options = new DbContextOptionsBuilder<FamilyDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString())
            .Options;
        _context = new FamilyDbContext(options);
        
        _keycloakServiceMock = new Mock<IKeycloakService>();
        _loggerMock = new Mock<ILogger<CreateFamilyHandler>>();
        
        _handler = new CreateFamilyHandler(_context, _keycloakServiceMock.Object, _loggerMock.Object);
    }
    
    [Fact]
    public async Task Handle_ValidCommand_ShouldCreateFamilyAndParent()
    {
        // Arrange
        var command = new CreateFamilyCommand(
            "Test Family",
            "John",
            "Doe",
            "john.doe@email.com",
            DateTime.Now.AddYears(-30));
            
        _keycloakServiceMock
            .Setup(x => x.CreateParentUserAsync(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()))
            .ReturnsAsync(Result.Success("keycloak-user-id"));
        
        // Act
        var result = await _handler.Handle(command, CancellationToken.None);
        
        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.FamilyName.Should().Be("Test Family");
        
        var family = await _context.Families.FirstOrDefaultAsync();
        family.Should().NotBeNull();
        family!.Name.Should().Be("Test Family");
        
        var parent = await _context.FamilyMembers.FirstOrDefaultAsync();
        parent.Should().NotBeNull();
        parent!.Role.Should().Be(FamilyRole.Parent);
        parent.Email.Should().Be("john.doe@email.com");
    }
    
    [Fact]
    public async Task Handle_KeycloakFailure_ShouldReturnFailure()
    {
        // Arrange
        var command = new CreateFamilyCommand(
            "Test Family",
            "John",
            "Doe",
            "john.doe@email.com",
            DateTime.Now.AddYears(-30));
            
        _keycloakServiceMock
            .Setup(x => x.CreateParentUserAsync(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()))
            .ReturnsAsync(Result.Failure<string>("Keycloak error"));
        
        // Act
        var result = await _handler.Handle(command, CancellationToken.None);
        
        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Be("Keycloak error");
        
        var familyCount = await _context.Families.CountAsync();
        familyCount.Should().Be(0);
    }
}
```

### Integration Testing with Vertical Slices
```csharp
// Tests/Integration/Features/CreateFamily/CreateFamilyIntegrationTests.cs
public class CreateFamilyIntegrationTests : IClassFixture<GameGuardianWebApplicationFactory>
{
    private readonly GameGuardianWebApplicationFactory _factory;
    private readonly HttpClient _client;
    
    public CreateFamilyIntegrationTests(GameGuardianWebApplicationFactory factory)
    {
        _factory = factory;
        _client = _factory.CreateClient();
    }
    
    [Fact]
    public async Task CreateFamily_ValidRequest_ShouldCreateFamilySuccessfully()
    {
        // Arrange
        using var activity = TestActivitySource.StartActivity("CreateFamily_Integration_Test");
        
        var command = new CreateFamilyCommand(
            "Integration Test Family",
            "Jane",
            "Smith",
            "jane.smith@email.com",
            DateTime.Now.AddYears(-25));
        
        // Act
        var response = await _client.PostAsJsonAsync("/api/families", command);
        
        // Assert
        response.Should().BeSuccessful();
        
        var result = await response.Content.ReadFromJsonAsync<CreateFamilyResponse>();
        result.Should().NotBeNull();
        result!.FamilyName.Should().Be("Integration Test Family");
        
        // Verify in database
        using var scope = _factory.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<FamilyDbContext>();
        
        var family = await context.Families
            .Include(f => f.Members)
            .FirstOrDefaultAsync(f => f.Id == result.FamilyId);
            
        family.Should().NotBeNull();
        family!.Members.Should().HaveCount(1);
        family.Members.First().Role.Should().Be(FamilyRole.Parent);
        
        // Verify distributed tracing
        var traces = await GetDistributedTracesAsync(activity.TraceId);
        traces.Should().ContainSpan("CreateFamily.Handle")
              .And.ContainSpan("Keycloak.CreateUser")
              .And.ContainSpan("Database.SaveChanges");
    }
}
```

### Code Standards
- Vertical Slice Architecture with feature-focused organization
- SOLID principles and dependency injection
- Comprehensive unit and integration testing per slice
- OpenTelemetry instrumentation for observability
- Secure coding practices for child safety

### Testing Strategy
- **Unit Tests**: Test each slice handler independently with mocked dependencies
- **Integration Tests**: Test complete slices with TestContainers for real dependencies
- **Advanced Testing**: OpenTelemetry-based integration testing with trace verification
- **UI Tests**: Uno.UITest for cross-platform consistency
- **Performance Tests**: Load testing with NBomber per critical slice
- **Security Tests**: OWASP ZAP scanning, compliance validation per endpoint

### Development Environment
- Docker containers for local development
- PostgreSQL database with Entity Framework migrations
- Redis for caching and session management
- Keycloak for authentication testing
- OpenTelemetry for local observability

### UNO Platform Implementation

#### Shared Application Structure
```csharp
// App.xaml.cs - Application lifecycle
public partial class App : Application
{
    public static IHost? Host { get; private set; }
    
    protected override void OnLaunched(LaunchActivatedEventArgs args)
    {
        // Configure dependency injection
        var builder = Host.CreateApplicationBuilder();
        
        builder.Services.AddSingleton<IApiService, ApiService>();
        builder.Services.AddSingleton<ISignalRService, SignalRService>();
        builder.Services.AddSingleton<IAuthenticationService, AuthenticationService>();
        
        // Configure HTTP client
        builder.Services.AddHttpClient("GameGuardianApi", client =>
        {
            client.BaseAddress = new Uri("https://api.gameguardian.com/");
        });
        
        Host = builder.Build();
        
        var window = new MainWindow();
        window.Activate();
    }
}
```

#### Cross-Platform Service Implementation
```csharp
public class SignalRService : ISignalRService
{
    private HubConnection? _hubConnection;
    
    public async Task ConnectAsync()
    {
        var accessToken = await _authService.GetAccessTokenAsync();
        
        _hubConnection = new HubConnectionBuilder()
            .WithUrl("https://api.gameguardian.com/hubs/parent", options =>
            {
                options.AccessTokenProvider = () => Task.FromResult(accessToken);
            })
            .WithAutomaticReconnect()
            .Build();
            
        // Platform-specific notification setup
#if __MOBILE__
        _hubConnection.On<PurchaseRequest>("PurchaseApprovalRequired", async (request) =>
        {
            await ShowMobileNotification(request);
        });
#elif __DESKTOP__
        _hubConnection.On<PurchaseRequest>("PurchaseApprovalRequired", async (request) =>
        {
            ShowDesktopToastNotification(request);
        });
#elif __WASM__
        _hubConnection.On<PurchaseRequest>("PurchaseApprovalRequired", async (request) =>
        {
            await ShowBrowserNotification(request);
        });
#endif
        
        await _hubConnection.StartAsync();
    }
}
```

## 🧪 Testing Strategy Implementation

### Advanced Integration Testing Framework
```csharp
public class GameGuardianIntegrationTest : IAsyncLifetime
{
    private readonly WebApplicationFactory<Program> _factory;
    private readonly PostgreSqlContainer _dbContainer;
    private readonly RedisContainer _redisContainer;
    
    public GameGuardianIntegrationTest()
    {
        _dbContainer = new PostgreSqlBuilder()
            .WithDatabase("gameguardian_test")
            .WithUsername("test")
            .WithPassword("test")
            .Build();
            
        _redisContainer = new RedisBuilder()
            .Build();
            
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureTestServices(services =>
                {
                    // Replace with test containers
                    services.AddDbContext<GameGuardianDbContext>(options =>
                        options.UseNpgsql(_dbContainer.GetConnectionString()));
                    
                    services.AddStackExchangeRedisCache(options =>
                        options.Configuration = _redisContainer.GetConnectionString());
                });
            });
    }
    
    [Fact]
    public async Task Should_Track_Gaming_Session_With_Distributed_Tracing()
    {
        // Arrange
        using var activity = TestActivitySource.StartActivity("integration_test");
        var client = _factory.CreateClient();
        
        // Act - Start gaming session
        var sessionRequest = new StartSessionRequest
        {
            ChildId = "test-child-id",
            GameTitle = "Test Game",
            Platform = "Steam"
        };
        
        var response = await client.PostAsJsonAsync("/api/sessions/start", sessionRequest);
        
        // Assert - Verify trace spans were created
        response.Should().BeSuccessful();
        
        var traces = await GetDistributedTracesAsync(activity.TraceId);
        traces.Should().ContainSpan("session.start")
                     .And.ContainSpan("database.insert")
                     .And.ContainSpan("signalr.broadcast");
    }
}
```

### OpenTelemetry Test Utilities
```csharp
public static class OpenTelemetryTestExtensions
{
    public static async Task<List<ActivitySpan>> GetDistributedTracesAsync(string traceId)
    {
        // Integration with Jaeger test container for trace verification
        var jaegerClient = new JaegerClient(_jaegerContainer.GetConnectionString());
        return await jaegerClient.GetTraceSpansAsync(traceId);
    }
    
    public static ActivitySpanAssertions Should(this List<ActivitySpan> spans)
    {
        return new ActivitySpanAssertions(spans);
    }
}

public class ActivitySpanAssertions
{
    private readonly List<ActivitySpan> _spans;
    
    public ActivitySpanAssertions ContainSpan(string operationName)
    {
        _spans.Should().Contain(s => s.OperationName == operationName,
            $"Expected to find span with operation name '{operationName}'");
        return this;
    }
}
```

## 🚀 Deployment Configuration

### Flux GitOps Structure
```yaml
# flux/clusters/production/flux-system/gotk-sync.yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: gameguardian-gitops
  namespace: flux-system
spec:
  interval: 1m
  ref:
    branch: main
  url: https://github.com/npmulder/GameGuardian
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: gameguardian-apps
  namespace: flux-system
spec:
  interval: 10m
  path: "./deploy/flux/apps"
  prune: true
  sourceRef:
    kind: GitRepository
    name: gameguardian-gitops
  validation: client
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: family-service
      namespace: gameguardian
    - apiVersion: apps/v1
      kind: Deployment
      name: gaming-service
      namespace: gameguardian
```

### Helm Release Configuration
```yaml
# flux/apps/gameguardian/releases/family-service.yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: family-service
  namespace: gameguardian
spec:
  interval: 5m
  chart:
    spec:
      chart: ./deploy/kubernetes/charts/family-service
      sourceRef:
        kind: GitRepository
        name: gameguardian-gitops
        namespace: flux-system
  values:
    image:
      repository: ghcr.io/npmulder/gameguardian/family-service
      tag: "1.0.0" # {"$imagepolicy": "gameguardian:family-service:tag"}
    replicaCount: 3
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
    database:
      connectionString: "postgresql://user:pass@postgres:5432/gameguardian"
    keycloak:
      url: "https://auth.gameguardian.com"
      realm: "gameguardian"
  postRenderers:
    - kustomize:
        patches:
          - target:
              kind: Deployment
              name: family-service
            patch: |
              - op: add
                path: /spec/template/metadata/annotations/fluxcd.io~1automated
                value: "true"
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageRepository
metadata:
  name: family-service
  namespace: gameguardian
spec:
  image: ghcr.io/npmulder/gameguardian/family-service
  interval: 1m
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: family-service
  namespace: gameguardian
spec:
  imageRepositoryRef:
    name: family-service
  policy:
    semver:
      range: ">=1.0.0"
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: family-service
  namespace: gameguardian
spec:
  interval: 30m
  sourceRef:
    kind: GitRepository
    name: gameguardian-gitops
    namespace: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
      messageTemplate: |
        Automated image update

        Automation name: {{ .AutomationObject }}

        Files:
        {{ range $filename, $_ := .Updated.Files -}}
        - {{ $filename }}
        {{ end -}}

        Objects:
        {{ range $resource, $_ := .Updated.Objects -}}
        - {{ $resource.Kind }} {{ $resource.Name }}
        {{ end -}}
    push:
      branch: main
  update:
    path: "./deploy/flux/apps"
    strategy: Setters
```

### Progressive Delivery with Flagger
```yaml
# flux/apps/gameguardian/releases/canary-family-service.yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: family-service
  namespace: gameguardian
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: family-service
  progressDeadlineSeconds: 60
  service:
    port: 80
    targetPort: 8080
    gateways:
    - gameguardian-gateway.istio-system.svc.cluster.local
    hosts:
    - api.gameguardian.com
    trafficPolicy:
      tls:
        mode: DISABLE
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 30s
    webhooks:
    - name: acceptance-test
      type: pre-rollout
      url: http://flagger-loadtester.test/
      timeout: 30s
      metadata:
        type: bash
        cmd: "curl -sd 'test' http://family-service-canary.gameguardian/api/health | grep OK"
    - name: load-test
      url: http://flagger-loadtester.test/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 http://family-service-canary.gameguardian/api/health"
```

### Multi-Environment Configuration
```yaml
# flux/apps/gameguardian/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: gameguardian-prod

resources:
  - ../../base
  - namespace.yaml
  - ingress.yaml
  - monitoring.yaml

patches:
  - target:
      kind: HelmRelease
      name: family-service
    patch: |
      - op: replace
        path: /spec/values/replicaCount
        value: 5
      - op: replace
        path: /spec/values/resources/requests/memory
        value: "512Mi"
      - op: replace
        path: /spec/values/resources/limits/memory
        value: "1Gi"
      - op: add
        path: /spec/values/autoscaling
        value:
          enabled: true
          minReplicas: 3
          maxReplicas: 10
          targetCPUUtilizationPercentage: 70

  - target:
      kind: HelmRelease
      name: gaming-service
    patch: |
      - op: replace
        path: /spec/values/replicaCount
        value: 3
      - op: add
        path: /spec/values/gaming/platforms
        value:
          steam:
            apiKey: "${STEAM_API_KEY_PROD}"
          xbox:
            clientId: "${XBOX_CLIENT_ID_PROD}"
            clientSecret: "${XBOX_CLIENT_SECRET_PROD}"

configMapGenerator:
  - name: environment-config
    literals:
      - ENVIRONMENT=production
      - LOG_LEVEL=info
      - METRICS_ENABLED=true

secretGenerator:
  - name: database-secrets
    literals:
      - CONNECTION_STRING=postgresql://prod_user:${DB_PASSWORD}@postgres-prod:5432/gameguardian_prod
```

### Notification Configuration
```yaml
# flux/clusters/production/notifications/alerts.yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Provider
metadata:
  name: slack
  namespace: flux-system
spec:
  type: slack
  channel: gameguardian-deployments
  secretRef:
    name: slack-webhook-url
---
apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Alert
metadata:
  name: gameguardian-apps
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: info
  eventSources:
    - kind: Kustomization
      name: '*'
    - kind: HelmRelease
      name: '*'
  inclusionList:
    - ".*gameguardian.*"
  summary: "GameGuardian Deployment Status"
  eventMetadata:
    env: "production"
    cluster: "gameguardian-prod"
---
apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Provider
metadata:
  name: webhook
  namespace: flux-system
spec:
  type: generic
  address: https://api.gameguardian.com/webhooks/flux-events
  secretRef:
    name: webhook-token
---
apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Alert
metadata:
  name: gameguardian-failures
  namespace: flux-system
spec:
  providerRef:
    name: webhook
  eventSeverity: error
  eventSources:
    - kind: Kustomization
      name: '*'
    - kind: HelmRelease
      name: '*'
    - kind: Canary
      name: '*'
  inclusionList:
    - ".*gameguardian.*"
  summary: "GameGuardian Deployment Failure"
```

### Flux Bootstrap Script
```bash
#!/bin/bash
# scripts/setup/bootstrap-flux.sh

set -e

CLUSTER_NAME=${1:-"gameguardian-prod"}
GITHUB_USER=${2:-"npmulder"}
GITHUB_REPO=${3:-"GameGuardian"}

echo "Bootstrapping Flux for cluster: $CLUSTER_NAME"

# Check if kubectl context is set
if ! kubectl config current-context | grep -q "$CLUSTER_NAME"; then
    echo "Error: Please set kubectl context to $CLUSTER_NAME"
    exit 1
fi

# Install Flux CLI if not present
if ! command -v flux &> /dev/null; then
    echo "Installing Flux CLI..."
    curl -s https://fluxcd.io/install.sh | sudo bash
fi

# Check GitHub token
if [ -z "$GITHUB_TOKEN" ]; then
    echo "Error: GITHUB_TOKEN environment variable must be set"
    exit 1
fi

# Bootstrap Flux
echo "Bootstrapping Flux with GitHub integration..."
flux bootstrap github \
  --owner="$GITHUB_USER" \
  --repository="$GITHUB_REPO" \
  --branch=main \
  --path="./deploy/flux/clusters/$CLUSTER_NAME" \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller \
  --read-write-key

# Wait for Flux to be ready
echo "Waiting for Flux controllers to be ready..."
kubectl wait --for=condition=ready pod -l app=source-controller -n flux-system --timeout=300s
kubectl wait --for=condition=ready pod -l app=kustomize-controller -n flux-system --timeout=300s
kubectl wait --for=condition=ready pod -l app=helm-controller -n flux-system --timeout=300s

# Install Flagger for progressive delivery
echo "Installing Flagger for canary deployments..."
kubectl apply -k github.com/fluxcd/flagger//kustomize/istio

# Create monitoring namespace and install Prometheus if needed
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -

echo "Flux bootstrap completed successfully!"
echo "You can now push your Flux configurations to trigger deployments."
```

### GitHub Actions Integration
```yaml
# .github/workflows/flux-validation.yaml
name: Flux Validation

on:
  pull_request:
    paths:
      - 'deploy/flux/**'
      - 'deploy/kubernetes/**'

jobs:
  validate-flux:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Flux CLI
        uses: fluxcd/flux2/action@main

      - name: Validate Flux configurations
        run: |
          find deploy/flux -name "*.yaml" -type f | xargs -I {} flux validate --path {}

      - name: Validate Kustomizations
        run: |
          for overlay in deploy/flux/apps/gameguardian/overlays/*/; do
            echo "Validating overlay: $overlay"
            kustomize build "$overlay" | flux validate --stdin
          done

      - name: Security scan
        run: |
          # Install kubesec
          curl -sSL https://github.com/controlplaneio/kubesec/releases/latest/download/kubesec_linux_amd64.tar.gz | tar -xvz
          
          # Scan Kubernetes manifests
          find deploy/kubernetes -name "*.yaml" -type f | xargs ./kubesec scan

  drift-detection:
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule'
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Check cluster drift
        run: |
          # This would connect to your cluster and check for drift
          # Implementation depends on your cluster access setup
          echo "Checking for configuration drift..."
          flux diff --path ./deploy/flux/clusters/production
```

## 📈 Monitoring and Observability

### Custom Metrics Configuration
```csharp
// Metrics.cs - Application-specific metrics
public static class GameGuardianMetrics
{
    private static readonly Counter ActiveSessions = Metrics
        .CreateCounter("gameguardian_active_sessions_total", 
                      "Total number of active gaming sessions",
                      new[] { "platform", "age_group" });
                      
    private static readonly Histogram PurchaseApprovalTime = Metrics
        .CreateHistogram("gameguardian_purchase_approval_duration_seconds",
                        "Time taken for purchase approval workflow");
                        
    private static readonly Gauge FamiliesMonitored = Metrics
        .CreateGauge("gameguardian_families_monitored",
                    "Number of families currently being monitored");
                    
    public static void RecordActiveSession(string platform, int childAge)
    {
        var ageGroup = childAge switch
        {
            < 10 => "young",
            < 15 => "teen",
            _ => "older"
        };
        
        ActiveSessions.WithLabels(platform, ageGroup).Inc();
    }
}
```

### Grafana Dashboard Configuration
```json
{
  "dashboard": {
    "title": "GameGuardian - Family Monitoring",
    "panels": [
      {
        "title": "Active Gaming Sessions",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(gameguardian_active_sessions_total)",
            "legendFormat": "Total Active Sessions"
          }
        ]
      },
      {
        "title": "Purchase Approval Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, gameguardian_purchase_approval_duration_seconds)",
            "legendFormat": "95th percentile"
          }
        ]
      }
    ]
  }
}
```

## 🎯 Next Steps for Implementation

### Immediate Actions (Week 1-2)
1. **Set up development environment**: Clone repo, run Docker containers
2. **Implement basic project structure**: Create .NET solutions and UNO Platform projects
3. **Configure Keycloak**: Set up realms, clients, and basic authentication flows
4. **Create database models**: Define Entity Framework entities and migrations
5. **Implement basic API endpoints**: Family management and authentication

### Short-term Goals (Week 3-6)
1. **UNO Platform dashboard**: Build cross-platform parent interface
2. **Real-time features**: Implement SignalR hubs for live monitoring
3. **Gaming platform integrations**: Start with Steam API integration
4. **Testing framework**: Set up advanced integration testing with OpenTelemetry
5. **Basic deployment**: Deploy to development Kubernetes cluster

### Medium-term Objectives (Month 2-3)
1. **Complete platform integrations**: Xbox, PlayStation, Nintendo, mobile
2. **Advanced analytics**: Implement behavioral analysis and reporting
3. **Production deployment**: Full Kubernetes setup with monitoring
4. **Performance optimization**: Load testing and optimization
5. **Security hardening**: Complete security audit and compliance validation

This documentation provides a comprehensive foundation for building GameGuardian. The project showcases advanced .NET development practices, cross-platform development with UNO Platform, sophisticated authentication with Keycloak, and enterprise-grade observability and deployment practices.