# Business Audit Events System & AI Integration
## Complete Technical Documentation

---

## Table of Contents
1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Business Audit Events Flow](#business-audit-events-flow)
4. [AI Integration](#ai-integration)
5. [Data Models](#data-models)
6. [API Endpoints](#api-endpoints)
7. [Use Cases](#use-cases)

---

## 1. Overview

### 1.1 What are Business Audit Events?

**Business Audit Events** are high-level application events that track **business actions** performed by users in the system. Unlike database-level audit logs (which track table changes), business audit events capture:

- **User Actions**: What the user did (e.g., "Create Booking", "Update Vehicle", "Search Availability")
- **Resources**: What was affected (e.g., Vehicle ID, Booking ID)
- **Context**: HTTP request details, execution time, success/failure
- **Traceability**: Trace IDs to correlate related events

### 1.2 Purpose

- **Compliance**: Track who did what, when, and why
- **Security**: Detect suspicious activities
- **Analytics**: Understand user behavior patterns
- **Debugging**: Trace request flows across microservices
- **Audit Trail**: Maintain complete history of business operations

---

## 2. System Architecture

### 2.1 Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│  (Controllers, Services - Business Logic Execution)          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Publishes Event
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Kafka Topic                               │
│         "business_audit_events" (2 Clusters)                 │
│  - Main Cluster: Primary Kafka                              │
│  - Default Cluster: Fallback/Secondary Kafka                │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Consumes Event
                     ▼
┌─────────────────────────────────────────────────────────────┐
│            BusinessAuditEventListener                        │
│  - Listens to both Kafka clusters                           │
│  - Processes incoming events                                │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Processes
                     ▼
┌─────────────────────────────────────────────────────────────┐
│          BusinessAuditEventService                           │
│  - Transforms DTO to Entity                                 │
│  - Saves to database                                        │
│  - Provides query methods                                   │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Persists
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              MySQL Database                                  │
│         Table: business_audit_events                         │
│  - Stores all audit events                                  │
│  - Indexed for fast queries                                 │
└─────────────────────────────────────────────────────────────┘
                     │
                     │ Queries
                     ▼
┌─────────────────────────────────────────────────────────────┐
│          BusinessAuditController                             │
│  - REST APIs for querying audit events                      │
│  - Filtering, pagination, search                            │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Key Technologies

- **Kafka**: Event streaming platform (dual cluster support)
- **Spring Kafka**: Kafka integration
- **JPA/Hibernate**: Database persistence
- **MySQL**: Relational database storage
- **Spring Data JPA Specifications**: Dynamic query building

---

## 3. Business Audit Events Flow

### 3.1 Event Creation & Publishing

**Step 1: Business Action Occurs**
```java
// Example: User creates a booking
// Somewhere in the application, an event is created
BusinessAuditLogInfo event = new BusinessAuditLogInfo();
event.setAuditId(UUID.randomUUID().toString());
event.setUserId("user123");
event.setAction("Create Booking");
event.setResourceType("Booking");
event.setResourceValue("BKG-12345");
event.setDescription("User created a new booking");
event.setTimestamp(LocalDateTime.now());
event.setSuccess(true);
// ... set other fields
```

**Step 2: Publish to Kafka**
```java
// Published to Kafka topic: "business_audit_events"
kafkaTemplate.send("business_audit_events", event);
```

### 3.2 Event Consumption & Processing

**Step 3: Kafka Listener Receives Event**

<augment_code_snippet path="src/main/java/com/seera/lumi/yaqeen/listeners/BusinessAuditEventListener.java" mode="EXCERPT">
````java
@KafkaListener(
    topics = {"${kafka.topic.business.audit.events:business_audit_events}"},
    groupId = "business-audit-event-listener")
public void handleBusinessAuditEvent(@Payload BusinessAuditLogInfo event) {
  log.info("Received business audit event: {}", event);
  businessAuditEventService.processBusinessAuditEvent(event);
}
````
</augment_code_snippet>

**Step 4: Service Processes and Saves Event**

<augment_code_snippet path="src/main/java/com/seera/lumi/yaqeen/service/BusinessAuditEventService.java" mode="EXCERPT">
````java
public void processBusinessAuditEvent(BusinessAuditLogInfo event) {
  BusinessAuditEvent auditEvent = new BusinessAuditEvent();

  // Map DTO to Entity
  auditEvent.setAuditId(event.getAuditId());
  auditEvent.setUserId(event.getUserId());
  auditEvent.setAction(event.getAction());
  // ... set all fields

  // Save to database
  businessAuditEventRepository.saveAndFlush(auditEvent);
}
````
</augment_code_snippet>

**Step 5: Event Stored in Database**
```sql
INSERT INTO business_audit_events (
  audit_id, user_id, action, resource_type, resource_value,
  timestamp, success, execution_time_ms, ...
) VALUES (...);
```

### 3.3 Dual Kafka Cluster Support

The system supports **two Kafka clusters** for high availability:

1. **Main Cluster**: Primary Kafka cluster
   - Topic: `${kafka.topic.business.audit.events}`
   - Listener: `business-audit-event-listener`

2. **Default Cluster**: Fallback/Secondary Kafka cluster
   - Topic: `${kafka.topic.business.audit.events.default}`
   - Listener: `business-audit-event-listener-default`

**Why Two Clusters?**
- **High Availability**: If main cluster fails, events go to default cluster
- **Multi-Region**: Different clusters for different regions
- **Load Distribution**: Distribute load across clusters

---

## 4. AI Integration

### 4.1 Where AI is Used

**AI is NOT directly used in Business Audit Events**, but it IS used in the related **Database Audit Events** system for generating human-readable summaries of database changes.

### 4.2 AI in Database Audit Events (AuditLogService)

**Use Case**: Generate human-readable descriptions of database changes

**Example Scenario**:
```
Database Change:
- Table: bookings
- Operation: UPDATE
- Before: {"status": "pending", "amount": 1000}
- After: {"status": "confirmed", "amount": 1200}

AI-Generated Summary:
"Booking status changed from 'pending' to 'confirmed' and amount increased from 1000 to 1200"
```

### 4.3 AI Implementation in Audit System

<augment_code_snippet path="src/main/java/com/seera/lumi/yaqeen/service/AuditLogService.java" mode="EXCERPT">
````java
private String generateDiffText(AuditEvent auditEvent, AuditEvent lastSnapshot) {
  String prompt = String.format("""
      Generate a concise summary of database changes.

      Metadata:
      User: %s
      Database: %s
      Table: %s

      Before
      %s

      After
      %s
      """,
      auditEvent.getUpdatedBy(),
      auditEvent.getDatabase(),
      auditEvent.getTable(),
      lastSnapshot.getSnapshot(),
      auditEvent.getSnapshot());

  BaseResponse response = aiService.generateContent(prompt, modelSelector.selectBestModel());
  return response.getMessage();
}
````
</augment_code_snippet>

### 4.4 AI Flow for Audit Remarks

```
1. Database Change Detected
   ↓
2. Fetch Previous Snapshot
   ↓
3. Build AI Prompt (Before/After comparison)
   ↓
4. Select Best Available AI Model (Gemini/Groq)
   ↓
5. Call AI API with Prompt
   ↓
6. AI Generates Human-Readable Summary
   ↓
7. Save Summary as "Remarks" in Audit Event
```

### 4.5 AI in Inspection Report Validation

**Another AI Use Case**: Validating vehicle inspection reports

<augment_code_snippet path="src/main/java/com/seera/lumi/yaqeen/listeners/ValidateInspectionReportListener.java" mode="EXCERPT">
````java
// AI analyzes inspection images
PromptRequestDTO request = new PromptRequestDTO(prompt, imageUrl);
BaseResponse aiResponse = aiController.generateWithImageUrl(request).getBody();

// Extract odometer reading from AI response
Integer extractedOdometerReading = extractOdometerReading(aiResponse.getMessage());
````
</augment_code_snippet>

**AI Tasks**:
- Analyze vehicle inspection images
- Extract odometer readings
- Validate inspection data
- Detect anomalies

---

## 5. Data Models

### 5.1 BusinessAuditLogInfo (DTO - Kafka Message)

```java
{
  "auditId": "uuid",              // Unique event ID
  "userId": "user123",            // Who performed the action
  "action": "Create Booking",     // What action was performed
  "resourceType": "Booking",      // Type of resource affected
  "resourceValue": "BKG-12345",   // Specific resource identifier
  "description": "...",           // Human-readable description
  "methodName": "createBooking",  // Java method name
  "className": "BookingService",  // Java class name
  "parameters": {...},            // Method parameters (JSON)
  "result": {...},                // Method result (JSON)
  "traceId": "trace-123",         // Distributed tracing ID
  "clientId": "web-app",          // Client application ID
  "timestamp": "2026-01-31 10:30:00",
  "httpMethod": "POST",           // HTTP method
  "requestUri": "/api/bookings",  // Request URI
  "userAgent": "Mozilla/5.0...",  // User agent
  "ipAddress": "192.168.1.1",     // Client IP
  "success": true,                // Success/failure flag
  "errorMessage": null,           // Error message if failed
  "executionTimeMs": 250          // Execution time in milliseconds
}
```

### 5.2 BusinessAuditEvent (Entity - Database)

**Table**: `business_audit_events`

| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT | Primary key (auto-increment) |
| audit_id | VARCHAR | Unique audit event ID |
| user_id | VARCHAR | User who performed action |
| action | VARCHAR | Action name |
| resource_type | VARCHAR | Resource type |
| resource_value | VARCHAR | Resource identifier |
| description | TEXT | Description |
| method_name | VARCHAR | Java method |
| class_name | VARCHAR | Java class |
| parameters | TEXT | JSON parameters |
| result | TEXT | JSON result |
| trace_id | VARCHAR | Trace ID |
| client_id | VARCHAR | Client ID |
| timestamp | DATETIME | Event timestamp |
| http_method | VARCHAR | HTTP method |
| request_uri | VARCHAR | Request URI |
| user_agent | VARCHAR | User agent |
| ip_address | VARCHAR | IP address |
| success | BOOLEAN | Success flag |
| error_message | TEXT | Error message |
| execution_time_ms | BIGINT | Execution time |
| created_on | DATETIME | Record creation time |
| updated_on | DATETIME | Record update time |

---

## 6. API Endpoints

### 6.1 Get Recent Audit Events

**GET** `/api/v1/business-audit/recent`

**Query Parameters**:
- `userId` (optional): Filter by user
- `action` (optional): Filter by action
- `resourceType` (optional): Filter by resource type
- `resourceValue` (optional): Filter by resource value
- `clientId` (optional): Filter by client
- `success` (optional): Filter by success/failure
- `startTime` (optional): Start time (yyyy-MM-dd HH:mm:ss)
- `endTime` (optional): End time (yyyy-MM-dd HH:mm:ss)
- `page` (default: 0): Page number
- `pageSize` (default: 10): Page size

**Response**:
```json
{
  "content": [
    {
      "auditId": "uuid",
      "userId": "user123",
      "action": "Create Booking",
      "resourceType": "Booking",
      "resourceValue": "BKG-12345",
      "timestamp": "2026-01-31T10:30:00",
      "success": true,
      "executionTimeMs": 250
    }
  ],
  "totalElements": 100,
  "totalPages": 10,
  "number": 0,
  "size": 10
}
```

### 6.2 Get Audit History

**GET** `/api/v1/business-audit/history`

**Required Parameters**:
- `userId`: User ID
- `action`: Action name
- `resourceType`: Resource type

**Optional Parameters**:
- `resourceValue`, `clientId`, `startTime`, `endTime`, `success`
- `page`, `pageSize`

**Response**: Same as `/recent` but sorted in ascending order (oldest first)

### 6.3 Get Filter Options

**GET** `/api/v1/business-audit/filters`

**Purpose**: Get unique values for dropdown filters

**Response**:
```json
{
  "userIds": ["user1", "user2", "user3"],
  "actions": ["Create Booking", "Update Vehicle", "Search Availability"],
  "resourceTypes": ["Booking", "Vehicle", "Customer"],
  "resourceValues": ["BKG-123", "VEH-456"],
  "clientIds": ["web-app", "mobile-app"]
}
```

### 6.4 Search by Trace ID or Audit ID

**GET** `/api/v1/business-audit/search`

**Query Parameters**:
- `traceId` (optional): Find all events with same trace ID
- `auditId` (optional): Find specific event by audit ID
- Other filters: `userId`, `action`, `resourceType`, etc.

**Use Case**: Trace a complete request flow across microservices

---

## 7. Use Cases

### 7.1 Security Monitoring

**Scenario**: Detect suspicious login attempts

```sql
SELECT * FROM business_audit_events
WHERE action = 'User Login'
  AND success = false
  AND user_id = 'admin'
  AND timestamp > NOW() - INTERVAL 1 HOUR
GROUP BY ip_address
HAVING COUNT(*) > 5;
```

### 7.2 Compliance Reporting

**Scenario**: Generate monthly report of all booking modifications

```sql
SELECT user_id, COUNT(*) as modification_count
FROM business_audit_events
WHERE action LIKE '%Booking%'
  AND resource_type = 'Booking'
  AND timestamp BETWEEN '2026-01-01' AND '2026-01-31'
GROUP BY user_id
ORDER BY modification_count DESC;
```

### 7.3 Performance Analysis

**Scenario**: Find slow operations

```sql
SELECT action, AVG(execution_time_ms) as avg_time, COUNT(*) as count
FROM business_audit_events
WHERE timestamp > NOW() - INTERVAL 24 HOUR
GROUP BY action
HAVING avg_time > 1000
ORDER BY avg_time DESC;
```

### 7.4 Distributed Tracing

**Scenario**: Trace a booking creation flow

```
Request Flow (Same Trace ID):
1. [Web App] Create Booking Request → trace-123
2. [Booking Service] Validate Customer → trace-123
3. [Vehicle Service] Check Availability → trace-123
4. [Payment Service] Process Payment → trace-123
5. [Notification Service] Send Confirmation → trace-123
```

Query:
```sql
SELECT * FROM business_audit_events
WHERE trace_id = 'trace-123'
ORDER BY timestamp ASC;
```

### 7.5 User Activity Timeline

**Scenario**: View all actions by a specific user

```
GET /api/v1/business-audit/history?userId=user123&page=0&pageSize=50
```

Response shows chronological timeline of user's actions.

---

## 8. Advanced Features

### 8.1 Custom Repository Methods

The system uses **JPA Specifications** for dynamic query building:

<augment_code_snippet path="src/main/java/com/seera/lumi/yaqeen/repository/BusinessAuditEventRepositoryCustom.java" mode="EXCERPT">
````java
public interface BusinessAuditEventRepositoryCustom {
  List<String> findDistinctUserIds(...);
  List<String> findDistinctActions(...);
  List<String> findDistinctResourceTypes(...);
  List<String> findDistinctResourceValues(...);
  List<String> findDistinctClientIds(...);
}
````
</augment_code_snippet>

**Purpose**: Efficiently fetch unique values for filter dropdowns

### 8.2 Cleanup Mechanism

**Automatic cleanup of unwanted audit events**:

```java
@Transactional
public int cleanupUnwantedAuditEvents() {
  List<String> actionsToDelete = List.of(
    "Search Live Vehicle Availability",
    "Get Inspection Report",
    "Search Ready Vehicle"
  );
  return businessAuditEventRepository.deleteByActionIn(actionsToDelete);
}
```

**Why?** Some high-frequency read operations don't need long-term audit trails.

### 8.3 Pagination & Sorting

All query endpoints support:
- **Pagination**: `page` and `pageSize` parameters
- **Sorting**: Default DESC by timestamp (recent first) or ASC (history)
- **Filtering**: Multiple filter combinations

---

## 9. Comparison: Business Audit vs Database Audit

| Aspect | Business Audit Events | Database Audit Events |
|--------|----------------------|----------------------|
| **Level** | Application/Business | Database/Technical |
| **Captures** | User actions | Table changes |
| **Example** | "User created booking" | "INSERT into bookings table" |
| **Granularity** | High-level | Low-level |
| **AI Usage** | ❌ No | ✅ Yes (diff summaries) |
| **Volume** | Lower | Higher |
| **Retention** | Longer (years) | Shorter (months) |
| **Use Case** | Compliance, analytics | Debugging, recovery |

---

## 10. AI Integration Summary

### 10.1 Where AI is Used in the System

1. **Database Audit Remarks** (AuditLogService)
   - Generates human-readable summaries of database changes
   - Compares before/after snapshots
   - Uses free AI models (Gemini, Groq)

2. **Inspection Report Validation** (ValidateInspectionReportListener)
   - Analyzes vehicle inspection images
   - Extracts odometer readings
   - Validates inspection data

3. **NOT Used in Business Audit Events**
   - Business audit events are already human-readable
   - No need for AI-generated summaries

### 10.2 AI Model Selection

The system uses the **AIModelSelector** to automatically choose the best available free AI model:

```
Selection Criteria:
1. Model is active
2. Within rate limits (per minute/day)
3. Highest success rate
4. Lowest current utilization

Available Models:
- Gemini 1.5 Flash (15 req/min, 1500 req/day)
- Llama 3.1 8B (30 req/min, 14400 req/day)
- And 30+ other free models
```

---

## 11. Configuration

### 11.1 Kafka Topics

```yaml
kafka:
  topic:
    business.audit.events: business_audit_events
```

### 11.2 Kafka Listeners

```yaml
kafka:
  listen:
    auto.start: true
    concurrency: 1
```

### 11.3 Database

```sql
CREATE TABLE business_audit_events (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  audit_id VARCHAR(255) NOT NULL UNIQUE,
  user_id VARCHAR(255),
  action VARCHAR(255),
  resource_type VARCHAR(255),
  resource_value VARCHAR(255),
  -- ... other columns
  INDEX idx_user_id (user_id),
  INDEX idx_action (action),
  INDEX idx_resource_type (resource_type),
  INDEX idx_timestamp (timestamp),
  INDEX idx_trace_id (trace_id)
);
```

---

## 12. Best Practices

### 12.1 Event Publishing

✅ **DO**:
- Always set `auditId` (unique UUID)
- Include `traceId` for distributed tracing
- Set `timestamp` accurately
- Mark `success` flag correctly
- Include meaningful `description`

❌ **DON'T**:
- Publish sensitive data (passwords, tokens)
- Publish high-frequency read operations
- Skip error handling

### 12.2 Querying

✅ **DO**:
- Use pagination for large result sets
- Add time range filters
- Use indexes (userId, action, timestamp)
- Cache filter options

❌ **DON'T**:
- Query without time limits
- Fetch all records at once
- Ignore performance implications

---

## 13. Monitoring & Alerts

### 13.1 Key Metrics

- **Event Processing Rate**: Events/second
- **Kafka Lag**: Consumer lag
- **Database Write Performance**: Insert time
- **Failed Events**: Error rate
- **Storage Growth**: Table size

### 13.2 Alerts

- Kafka consumer lag > 1000 messages
- Event processing failure rate > 1%
- Database write time > 500ms
- Storage growth > 10GB/day

---

## 14. Future Enhancements

### 14.1 Potential AI Integrations

1. **Anomaly Detection**
   - Use AI to detect unusual user behavior patterns
   - Alert on suspicious activities

2. **Predictive Analytics**
   - Predict system load based on audit patterns
   - Forecast resource usage

3. **Natural Language Queries**
   - "Show me all failed bookings by user123 last week"
   - AI translates to SQL query

4. **Automated Insights**
   - AI generates daily/weekly audit summaries
   - Highlights important trends

### 14.2 Other Enhancements

- Real-time dashboards
- Elasticsearch integration for full-text search
- Data archival to cold storage
- GDPR compliance (data anonymization)

---

## 15. Conclusion

The **Business Audit Events** system provides comprehensive tracking of business operations with:

- ✅ Kafka-based event streaming
- ✅ Dual cluster support for high availability
- ✅ Rich filtering and querying capabilities
- ✅ Distributed tracing support
- ✅ Performance monitoring
- ✅ Compliance-ready audit trails

**AI Integration** is used in related systems (database audits, inspection validation) but not directly in business audit events, as they are already human-readable.

---

**Document Version**: 1.0
**Last Updated**: 2026-01-31
**Author**: Technical Documentation Team

