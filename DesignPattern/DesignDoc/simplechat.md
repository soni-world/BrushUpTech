# Product Requirements Document (PRD)
# Simple AI Chat Client - Free Model Rate Limiter

## 1. Executive Summary

### 1.1 Project Overview
Build a **minimal Spring Boot service** that acts as a smart wrapper around free-tier AI chat models. The service automatically selects an available model based on rate limits, allowing you to experiment with AI chat applications without worrying about quota management.

### 1.2 Core Purpose
- **Single Goal**: Provide a simple chat API that automatically uses free AI models
- **No Complex Features**: Just rate limiting and model selection
- **For Experimentation**: Perfect for small tasks, prototyping, and learning

### 1.3 Use Case
```
You: "I want to build a chatbot"
Service: "Let me handle which free AI model to use based on limits"
You: Just call POST /chat with your message
Service: Returns AI response using the best available free model
```

---

## 2. Scope - What's INCLUDED ✅

### 2.1 Core Features (Minimal Viable Product)
1. **Simple Chat API** - Single endpoint to send messages and get responses
2. **Rate Limit Tracking** - Track usage per model (per minute/day)
3. **Auto Model Selection** - Pick the best available free model
4. **Multiple Free Models** - Support 15+ free AI models (Gemini, Groq, etc.)
5. **Basic Monitoring** - Simple health check and usage stats

### 2.2 Supported Models
- **Google Gemini** (6 free models)
- **Groq/Meta-Llama** (8+ free models)
- **Others** (IBM Allam, Mistral, etc.)

---

## 3. Scope - What's EXCLUDED ❌

### 3.1 NOT Included (Keep It Simple)
- ❌ Image/Document processing (text chat only)
- ❌ Admin UI or management console
- ❌ User authentication/authorization
- ❌ Multi-tenancy or per-user limits
- ❌ Conversation history/memory
- ❌ Streaming responses
- ❌ Custom model configuration via API
- ❌ Database persistence
- ❌ Advanced monitoring/alerting
- ❌ Load balancing across instances

---

## 4. Technical Architecture (Simplified)

### 4.1 Technology Stack
- **Framework**: Spring Boot 3.x
- **Language**: Java 17+
- **Cache**: Redis (for rate limiting only)
- **Build**: Maven
- **That's it!** No database, no Kafka, no complex infrastructure

### 4.2 Core Components (Only 5!)

```
1. AIModelRegistry     → Stores model configs in memory
2. AIModelSelector     → Picks best available model
3. AIUsageTracker      → Tracks rate limits in Redis
4. ChatService         → Calls AI APIs
5. ChatController      → Single REST endpoint
```

---

## 5. API Design (Super Simple)

### 5.1 Main Endpoint - Chat

**POST /api/chat**
```json
Request:
{
  "message": "Explain quantum computing in simple terms"
}

Response:
{
  "reply": "Quantum computing is...",
  "model": "gemini-1.5-flash",
  "timestamp": "2026-01-31T10:30:00Z"
}
```

### 5.2 Optional Endpoints (Nice to Have)

**GET /api/models**
- List all available models and their limits

**GET /api/health**
- Check service health

**GET /api/stats**
- View current usage statistics

---

## 6. Data Models (Minimal)

### 6.1 ChatRequest
```java
{
  "message": "string"  // That's it!
}
```

### 6.2 ChatResponse
```java
{
  "reply": "string",
  "model": "string",
  "timestamp": "string"
}
```

### 6.3 AIModel (Internal)
```java
{
  "modelId": "string",
  "requestsPerMinute": "int",
  "requestsPerDay": "int",
  "isActive": "boolean"
}
```

---

## 7. Configuration (Environment Variables)

```bash
# Server
SERVER_PORT=8080

# Redis (for rate limiting)
REDIS_HOST=localhost
REDIS_PORT=6379

# API Keys (only what you need)
GEMINI_API_KEY=your-key-here
GROQ_API_KEY=your-key-here
```

---

## 8. Pre-configured Free Models

### 8.1 Best Free Models (Auto-configured)
```
1. gemini-1.5-flash        → 15 req/min, 1500 req/day  ⭐ BEST
2. gemini-1.5-flash-8b     → 15 req/min, 1500 req/day  ⭐ BEST
3. llama-3.1-8b-instant    → 30 req/min, 14400 req/day ⭐ BEST
4. llama3-8b-8192          → 30 req/min, 14400 req/day
5. gemini-1.5-pro          → 2 req/min, 50 req/day
6. llama-3.3-70b-versatile → 30 req/min, 1000 req/day
7. gemma2-9b-it            → 30 req/min, 14400 req/day
8. deepseek-r1-distill     → 30 req/min, 1000 req/day
```

**Total Daily Capacity**: ~50,000+ free requests per day!

---

## 9. Project Structure (Minimal)

```
simple-ai-chat-client/
├── src/main/java/com/yourname/aichat/
│   ├── controller/
│   │   └── ChatController.java          ← Single REST controller
│   ├── service/
│   │   ├── ChatService.java             ← Main chat logic
│   │   └── CacheService.java            ← Redis wrapper
│   ├── registry/
│   │   └── AIModelRegistry.java         ← Model storage
│   ├── selector/
│   │   └── AIModelSelector.java         ← Model selection
│   ├── tracker/
│   │   └── AIUsageTracker.java          ← Rate limiting
│   ├── provider/
│   │   ├── AIProvider.java              ← Interface
│   │   ├── GeminiProvider.java          ← Gemini API
│   │   └── GroqProvider.java            ← Groq API
│   ├── dto/
│   │   ├── ChatRequest.java
│   │   ├── ChatResponse.java
│   │   └── AIModel.java
│   └── config/
│       ├── AIModelConfig.java           ← Model initialization
│       └── RedisConfig.java             ← Redis setup
├── src/main/resources/
│   └── application.yaml
├── pom.xml
└── README.md
```

**Total Files**: ~15 Java files (very manageable!)

---

## 10. Implementation Steps (Quick Start)

### Step 1: Create Spring Boot Project
```bash
# Use Spring Initializr
Dependencies: Web, Redis, Lombok
```

### Step 2: Copy Core Files from Reference
```bash
# Copy these 7 files from lumi-core-yaqeen-business:
1. AIModelRegistry.java
2. AIModelSelector.java
3. AIModelUsageTracker.java
4. GeminiProvider.java
5. GroqProvider.java
6. AIProvider.java (interface)
7. AIModel.java (DTO)
```

### Step 3: Create Simple ChatController
```java
@RestController
@RequestMapping("/api")
public class ChatController {
    
    @PostMapping("/chat")
    public ChatResponse chat(@RequestBody ChatRequest request) {
        // 1. Select best model
        // 2. Call AI API
        // 3. Return response
    }
}
```

### Step 4: Configure Models
```java
@PostConstruct
public void initModels() {
    // Register 8-10 best free models
    // Copy from AIModelConfig.java
}
```

### Step 5: Run!
```bash
mvn spring-boot:run
```

---

## 11. Usage Examples

### Example 1: Simple Chat
```bash
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What is AI?"}'
```

### Example 2: Check Available Models
```bash
curl http://localhost:8080/api/models
```

### Example 3: View Usage Stats
```bash
curl http://localhost:8080/api/stats
```

---

## 12. Dependencies (Minimal)

```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Redis for rate limiting -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    
    <!-- Lombok (less boilerplate) -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
    
    <!-- That's it! -->
</dependencies>
```

---

## 13. How It Works (Simple Flow)

```
1. You send: POST /api/chat {"message": "Hello"}
2. Service checks: Which models have available quota?
3. Service selects: Best model (e.g., gemini-1.5-flash)
4. Service calls: Gemini API with your message
5. Service tracks: Increment usage counter in Redis
6. You receive: {"reply": "Hi there!", "model": "gemini-1.5-flash"}
```

**If model hits limit**: Automatically tries next available model!

---

## 14. Rate Limiting Logic (Simple)

```java
// For each model, track in Redis:
model:gemini-1.5-flash:requests_per_minute:1738320600000 = 5
model:gemini-1.5-flash:requests_per_day:1738320600000 = 120

// Before using model, check:
if (minuteCount < 15 && dayCount < 1500) {
    // Use this model
} else {
    // Try next model
}
```

---

## 15. Success Criteria (Keep It Simple)

### Launch Checklist
- ✅ Can send chat message and get response
- ✅ Automatically switches models when limits hit
- ✅ Tracks usage in Redis
- ✅ Works with 8+ free models
- ✅ Can run locally with Docker Redis
- ✅ Basic error handling
- ✅ README with examples

**That's it!** No complex requirements.

---

## 16. Timeline (Fast!)

- **Day 1**: Project setup, copy core files
- **Day 2**: Implement ChatController and ChatService
- **Day 3**: Testing and documentation
- **Day 4**: Polish and deploy

**Total Time**: 4 days (or 1-2 days if focused)

---

## 17. Future Enhancements (Optional)

### Phase 2 (If Needed)
- Conversation history (store last 10 messages)
- Streaming responses (SSE)
- Simple web UI for testing
- Docker Compose for easy deployment

### Phase 3 (Maybe)
- User API keys for rate limiting per user
- Prompt templates
- Cost tracking

**But for now**: Keep it simple!

---

## 18. Key Differences from Full PRD

| Feature | Full Gateway | Simple Chat Client |
|---------|-------------|-------------------|
| Purpose | Production service | Experimentation |
| Endpoints | 10+ APIs | 1-3 APIs |
| Features | Images, docs, admin | Text chat only |
| Auth | JWT/OAuth | None |
| Monitoring | Full observability | Basic health check |
| Deployment | K8s, multi-env | Local/Docker |
| Complexity | High | Low |
| Setup Time | 4 weeks | 4 days |

---

## 19. Quick Reference

### Start Redis
```bash
docker run -d -p 6379:6379 redis:alpine
```

### Run Service
```bash
mvn spring-boot:run
```

### Test Chat
```bash
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Tell me a joke"}'
```

### Check Health
```bash
curl http://localhost:8080/api/health
```

---

## 20. Summary

**What You Get**:
- ✅ Simple chat API
- ✅ 50,000+ free requests/day across multiple models
- ✅ Automatic rate limit management
- ✅ No quota worries
- ✅ Perfect for experiments

**What You Don't Get**:
- ❌ Complex features
- ❌ Production-grade infrastructure
- ❌ Advanced monitoring

**Perfect For**:
- 🧪 AI experimentation
- 📚 Learning AI APIs
- 🚀 Quick prototypes
- 💡 Small projects

---

**Document Version**: 1.0 (Simplified)  
**Last Updated**: 2026-01-31  
**Status**: Ready for Quick Implementation  
**Estimated Setup Time**: 1-4 days

