# UML - Diagramas de Integración (Sequence Diagrams)

> **Objetivo**: Documentar flujos críticos de interacción entre servicios, mostrando orquestación, compensaciones y sincronización Legacy↔Nuevo.  
> **Audiencia**: Desarrolladores, Arquitectos de Soluciones, Tech Leads

---

## 🎯 Contexto de Integración

Estos diagramas muestran cómo los servicios de FinScale Evolution interactúan para resolver procesos de negocio críticos, aplicando patrones de integración como:

- **Temporal.io Saga Pattern**: Orquestación de transacciones distribuidas con compensación automática
- **Event-Driven Architecture**: Comunicación asíncrona vía Kafka para desacoplamiento
- **Change Data Capture (CDC)**: Sincronización bidireccional Legacy↔Nuevo sin dual-writes
- **Circuit Breaker + Retry**: Resiliencia ante fallos de servicios externos

### Problemas del Sistema Actual Resueltos

| Problema Legacy | Solución en Nueva Arquitectura |
|-----------------|-------------------------------|
| **Rollback manual** tras fallos en clearing | Temporal Saga con compensaciones automáticas |
| **Dual-writes** para sincronizar Legacy | CDC Adapter captura Oracle WAL, publica a Kafka |
| **Cascading failures** cuando servicio cae | Circuit Breaker detiene llamadas a servicios fallidos |
| **Timeouts bloqueantes** en fraud check | Async scoring con default score si timeout |

---

## 📐 Flujo 1: Pago Internacional con Temporal Saga (Happy Path)

**Escenario**: Cliente envía $500 USD a beneficiario en Brasil. El pago requiere:
1. Scoring de fraude (< 100ms)
2. Cotización y bloqueo FX (USD→BRL)
3. Registro inmutable en Ledger
4. Envío a red PIX (instant settlement)

**Patrón Aplicado**: **Temporal.io Saga Pattern** con compensación automática si algún paso falla.

```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant App as Mobile App
    participant Gateway as API Gateway<br/>(Kong)
    participant Payment as Payment Service<br/>(Spring WebFlux)
    participant Temporal as Temporal.io<br/>(Workflow Engine)
    participant Fraud as Fraud Service<br/>(TensorFlow)
    participant FX as FX Service<br/>(Redis Cache)
    participant Ledger as Ledger Service<br/>(TimescaleDB)
    participant Clearing as Clearing Service<br/>(PIX Adapter)
    participant PIX as PIX Network<br/>(Banco Central BR)
    participant Kafka as Kafka<br/>(Event Streaming)
    
    Cliente->>App: Iniciar pago<br/>$500 USD → BRL
    App->>Gateway: POST /api/v1/payments<br/>Authorization: Bearer {token}
    Gateway->>Payment: CreatePayment(request)
    
    Note over Payment: Validar request<br/>(schema, saldo disponible)
    
    Payment->>Temporal: StartWorkflow<br/>(PaymentSagaWorkflow)
    activate Temporal
    
    rect rgb(230, 245, 255)
        Note over Temporal: Saga Step 1: Fraud Check
        Temporal->>Fraud: CheckFraud(paymentId)
        activate Fraud
        Fraud->>Fraud: ML Scoring<br/>(50 features, 30ms)
        Fraud-->>Temporal: RiskScore(25, APPROVED)
        deactivate Fraud
        
        Note over Temporal: ✅ Score < 70 → Continue
    end
    
    rect rgb(255, 245, 230)
        Note over Temporal: Saga Step 2: FX Lock
        Temporal->>FX: LockRate(USD, BRL, 500)
        activate FX
        FX->>FX: Redis GET fx:rate:USD/BRL<br/>1 USD = 5.25 BRL
        FX->>FX: Lock TTL=5min<br/>lockId: abc123
        FX-->>Temporal: RateLock(5.25, lockId)
        deactivate FX
        
        Note over Temporal: ✅ Rate locked → Continue
    end
    
    rect rgb(230, 255, 230)
        Note over Temporal: Saga Step 3: Reserve Funds
        Temporal->>Ledger: ReserveFunds(account, 500)
        activate Ledger
        Ledger->>Ledger: Append Event:<br/>FUNDS_RESERVED
        Ledger->>Kafka: Publish LedgerEvent
        Ledger-->>Temporal: ReservationId
        deactivate Ledger
        
        Note over Temporal: ✅ Funds reserved → Continue
    end
    
    rect rgb(255, 230, 255)
        Note over Temporal: Saga Step 4: Send to Clearing
        Temporal->>Clearing: SendPayment(PIX, 2625 BRL)
        activate Clearing
        Clearing->>Clearing: Build PIX Request<br/>(JSON payload)
        Clearing->>PIX: POST /v2/pix<br/>{"amount": 2625, "key": "..."}
        activate PIX
        PIX-->>Clearing: 200 OK<br/>{"txId": "E12345", "status": "CONFIRMED"}
        deactivate PIX
        Clearing-->>Temporal: ClearingConfirmed(txId)
        deactivate Clearing
        
        Note over Temporal: ✅ PIX instant settlement → Success
    end
    
    rect rgb(230, 255, 245)
        Note over Temporal: Saga Step 5: Settle Ledger
        Temporal->>Ledger: SettleFunds(reservationId)
        activate Ledger
        Ledger->>Ledger: Append Event:<br/>FUNDS_SETTLED
        Ledger->>Kafka: Publish PaymentSettled
        Ledger-->>Temporal: Settled
        deactivate Ledger
    end
    
    Temporal-->>Payment: WorkflowCompleted(SUCCESS)
    deactivate Temporal
    
    Payment->>Kafka: Publish PaymentExecuted
    Payment-->>Gateway: 200 OK<br/>{"paymentId": "...", "status": "SETTLED"}
    Gateway-->>App: Payment success
    App-->>Cliente: ✅ Pago completado<br/>en 8 segundos
```

### Compensación Automática (Unhappy Path)

**Escenario**: PIX Network rechaza el pago (cuenta beneficiario cerrada).

```mermaid
sequenceDiagram
    autonumber
    participant Temporal as Temporal.io<br/>(Saga Orchestrator)
    participant Clearing as Clearing Service
    participant PIX as PIX Network
    participant Ledger as Ledger Service<br/>(Compensation)
    participant FX as FX Service<br/>(Compensation)
    participant Notification as Notification Service
    
    Note over Temporal: Ejecutando Step 4: Send to Clearing
    
    Temporal->>Clearing: SendPayment(PIX, 2625 BRL)
    activate Clearing
    Clearing->>PIX: POST /v2/pix
    activate PIX
    PIX-->>Clearing: 400 Bad Request<br/>{"error": "ACCOUNT_CLOSED"}
    deactivate PIX
    Clearing-->>Temporal: ClearingFailed(ACCOUNT_CLOSED)
    deactivate Clearing
    
    rect rgb(255, 230, 230)
        Note over Temporal: 🔴 Failure Detected<br/>Iniciando compensaciones en orden inverso
        
        Note over Temporal: Compensate Step 3: Release Funds
        Temporal->>Ledger: ReleaseFunds(reservationId)
        activate Ledger
        Ledger->>Ledger: Append Event:<br/>FUNDS_RELEASED
        Ledger-->>Temporal: Released
        deactivate Ledger
        
        Note over Temporal: Compensate Step 2: Release FX Lock
        Temporal->>FX: ReleaseLock(lockId)
        activate FX
        FX->>FX: Redis DEL fx:lock:abc123
        FX-->>Temporal: Lock released
        deactivate FX
        
        Note over Temporal: No compensation needed for Step 1 (Fraud Check)
    end
    
    Temporal->>Notification: SendFailureNotification
    activate Notification
    Notification->>Notification: Push notification:<br/>"Pago falló: Cuenta destino cerrada"
    deactivate Notification
    
    Note over Temporal: ✅ Saga Rolled Back<br/>Sistema consistente
```

**Código Temporal Workflow**:

```java
@WorkflowInterface
public interface PaymentSagaWorkflow {
    @WorkflowMethod
    PaymentResult executePayment(PaymentOrder order);
}

@ActivityInterface
public interface PaymentActivities {
    RiskScore checkFraud(String paymentId);
    RateLock lockFXRate(String from, String to, BigDecimal amount);
    void releaseFXLock(String lockId); // Compensation
    String reserveFunds(String accountId, BigDecimal amount);
    void releaseFunds(String reservationId); // Compensation
    String sendToClearing(ClearingRequest request);
    void settleFunds(String reservationId);
}

// Implementación del Workflow
public class PaymentSagaWorkflowImpl implements PaymentSagaWorkflow {
    
    private final PaymentActivities activities = Workflow.newActivityStub(
        PaymentActivities.class,
        ActivityOptions.newBuilder()
            .setStartToCloseTimeout(Duration.ofSeconds(10))
            .setRetryOptions(RetryOptions.newBuilder()
                .setMaximumAttempts(3)
                .build())
            .build()
    );
    
    @Override
    public PaymentResult executePayment(PaymentOrder order) {
        // Saga Variables
        String lockId = null;
        String reservationId = null;
        
        try {
            // Step 1: Fraud Check (no compensation needed)
            RiskScore score = activities.checkFraud(order.getPaymentId());
            if (score.getStatus() == RiskStatus.BLOCKED) {
                return PaymentResult.blocked(score);
            }
            
            // Step 2: FX Lock (compensable)
            RateLock rateLock = activities.lockFXRate(
                order.getOriginCurrency(),
                order.getDestinationCurrency(),
                order.getAmount()
            );
            lockId = rateLock.getLockId();
            
            // Step 3: Reserve Funds (compensable)
            reservationId = activities.reserveFunds(
                order.getOriginatorAccountId(),
                order.getAmount()
            );
            
            // Step 4: Send to Clearing (compensable)
            String clearingTxId = activities.sendToClearing(
                ClearingRequest.from(order, rateLock)
            );
            
            // Step 5: Settle Funds (final)
            activities.settleFunds(reservationId);
            
            return PaymentResult.success(clearingTxId);
            
        } catch (Exception e) {
            // Automatic Compensation in reverse order
            Workflow.getLogger(getClass()).error("Payment failed, compensating...", e);
            
            if (reservationId != null) {
                Saga.compensate(() -> activities.releaseFunds(reservationId));
            }
            
            if (lockId != null) {
                Saga.compensate(() -> activities.releaseFXLock(lockId));
            }
            
            return PaymentResult.failed(e.getMessage());
        }
    }
}
```

---

## 📐 Flujo 2: Sincronización Legacy↔Nuevo con CDC (Change Data Capture)

**Escenario**: Payment creado en nuevo sistema debe ser visible para sistema Legacy (reportes, CRM). Al mismo tiempo, pagos creados en Legacy deben propagarse al nuevo sistema.

**Patrón Aplicado**: **CDC Pattern** con Debezium capturando Oracle WAL, publicando a Kafka.

```mermaid
sequenceDiagram
    autonumber
    participant Legacy as Legacy J2EE<br/>(Monolito)
    participant Oracle as Oracle Database<br/>(CORE_SCHEMA)
    participant Debezium as CDC Adapter<br/>(Debezium)
    participant Kafka as Kafka<br/>(legacy-changes topic)
    participant Transformer as Event Transformer
    participant Payment as Payment Service<br/>(Nuevo)
    participant Ledger as Ledger Service<br/>(Nuevo)
    participant Facade as Legacy Facade<br/>(ACL)
    
    rect rgb(255, 240, 240)
        Note over Legacy,Oracle: Dirección 1: Legacy → Nuevo
        
        Legacy->>Oracle: INSERT INTO PAYMENTS<br/>(TRANSACTION_ID, AMOUNT, STATUS)
        activate Oracle
        Oracle->>Oracle: Write to WAL<br/>(Write-Ahead Log)
        deactivate Oracle
        
        Note over Debezium: Polling WAL cada 500ms
        
        Debezium->>Oracle: Read WAL changes
        Oracle-->>Debezium: ChangeEvent<br/>{op: INSERT, table: PAYMENTS, ...}
        
        Debezium->>Kafka: Publish to legacy-changes<br/>(CDC event raw)
        
        Kafka->>Transformer: Consume legacy-changes
        activate Transformer
        
        Transformer->>Transformer: Transform:<br/>Oracle columns → Domain Event
        
        Note over Transformer: LegacyPaymentDTO → PaymentCreated
        
        Transformer->>Kafka: Publish to payment-events<br/>(Domain Event)
        deactivate Transformer
        
        Kafka->>Payment: Consume payment-events
        activate Payment
        Payment->>Payment: Process event:<br/>Create payment in PostgreSQL
        Payment->>Ledger: POST /ledger/entry
        deactivate Payment
    end
    
    rect rgb(240, 255, 240)
        Note over Payment,Legacy: Dirección 2: Nuevo → Legacy
        
        Payment->>Kafka: Publish PaymentCreated
        
        Kafka->>Facade: Consume payment-events
        activate Facade
        
        Facade->>Facade: Translate:<br/>PaymentCreated → LegacyPaymentDTO
        
        Note over Facade: DDD Model → Denormalized Legacy
        
        Facade->>Legacy: POST /legacy/api/payments<br/>(HTTP Bridge)
        activate Legacy
        Legacy->>Oracle: INSERT INTO PAYMENTS<br/>(via JDBC)
        Legacy-->>Facade: 201 Created
        deactivate Legacy
        
        Facade-->>Kafka: ACK consumed
        deactivate Facade
    end
    
    Note over Legacy,Ledger: ✅ Bidirectional Sync<br/>Lag < 2 segundos
```

**Código Event Transformer**:

```java
@Service
public class LegacyEventTransformer {
    
    @KafkaListener(topics = "legacy-changes")
    public Mono<Void> handleLegacyChange(ConsumerRecord<String, ChangeEvent> record) {
        ChangeEvent change = record.value();
        
        return switch(change.getOperation()) {
            case INSERT, UPDATE -> {
                // Parse Oracle columns
                Map<String, Object> after = change.getAfter();
                
                // Transform to Domain Event
                PaymentCreated domainEvent = PaymentCreated.builder()
                    .paymentId(UUID.fromString((String) after.get("TRANSACTION_ID")))
                    .originatorAccountId((String) after.get("ORIG_ACCOUNT_NUM"))
                    .beneficiaryFullName((String) after.get("BENEF_FULL_NAME"))
                    .amountCents(((Number) after.get("AMOUNT_CENTS")).longValue())
                    .currencyCode((String) after.get("CURRENCY_CODE"))
                    .status(PaymentStatus.valueOf((String) after.get("STATUS")))
                    .occurredAt(Instant.ofEpochMilli(change.getTimestamp()))
                    .source(EventSource.LEGACY_CDC)
                    .build();
                
                // Publish to domain topic
                yield kafkaTemplate.send("payment-events", domainEvent).then();
            }
            
            case DELETE -> {
                String paymentId = (String) change.getBefore().get("TRANSACTION_ID");
                
                PaymentCancelled domainEvent = new PaymentCancelled(
                    UUID.fromString(paymentId),
                    Instant.now(),
                    EventSource.LEGACY_CDC
                );
                
                yield kafkaTemplate.send("payment-events", domainEvent).then();
            }
        };
    }
}
```

---

## 📐 Flujo 3: Detección de Fraude en Tiempo Real con Circuit Breaker

**Escenario**: Fraud Service usa modelo ML en TensorFlow Serving. Si TensorFlow cae o tarda > 100ms, usar scoring por reglas estático.

**Patrón Aplicado**: **Circuit Breaker Pattern** con fallback a reglas simples.

```mermaid
sequenceDiagram
    autonumber
    participant Payment as Payment Service
    participant Fraud as Fraud Service<br/>(Circuit Breaker)
    participant TensorFlow as TensorFlow Serving<br/>(ML Model)
    participant Cassandra as Cassandra<br/>(Fraud Scores)
    participant Kafka as Kafka<br/>(fraud-events)
    
    rect rgb(230, 255, 230)
        Note over Payment,TensorFlow: Estado: Circuit CLOSED (healthy)
        
        Payment->>Fraud: POST /fraud/score<br/>{"paymentId": "..."}
        activate Fraud
        
        Fraud->>Fraud: Extract 50 features
        
        Fraud->>TensorFlow: POST /v1/models/fraud:predict<br/>{features: [...]}
        activate TensorFlow
        TensorFlow->>TensorFlow: Inference (30ms)
        TensorFlow-->>Fraud: {"score": 25}
        deactivate TensorFlow
        
        Fraud->>Cassandra: INSERT fraud_score<br/>(async fire-and-forget)
        
        Fraud->>Kafka: Publish FraudScored
        
        Fraud-->>Payment: 200 OK<br/>{"score": 25, "status": "APPROVED"}
        deactivate Fraud
    end
    
    rect rgb(255, 230, 230)
        Note over Payment,TensorFlow: Falla: TensorFlow timeout 3 veces consecutivas
        
        Payment->>Fraud: POST /fraud/score<br/>{"paymentId": "..."}
        activate Fraud
        
        Fraud->>TensorFlow: POST /v1/models/fraud:predict
        activate TensorFlow
        Note over TensorFlow: ⏱️ Timeout 100ms
        TensorFlow--xFraud: Timeout Exception
        deactivate TensorFlow
        
        Note over Fraud: 🔴 Circuit Breaker OPENS<br/>(3 failures detected)
        
        Fraud->>Fraud: Fallback: Rule-based scoring<br/>(amount + country + velocity)
        
        Fraud->>Fraud: Calculate static score: 45
        
        Fraud->>Cassandra: INSERT fraud_score<br/>(source: FALLBACK)
        
        Fraud->>Kafka: Publish FraudScoredFallback<br/>(alert ops team)
        
        Fraud-->>Payment: 200 OK<br/>{"score": 45, "status": "APPROVED",<br/>"warning": "ML model unavailable"}
        deactivate Fraud
        
        Note over Payment: ⚠️ Payment continues<br/>(degraded mode)
    end
    
    rect rgb(255, 245, 230)
        Note over Payment,TensorFlow: Recovery: Circuit HALF_OPEN
        
        Note over Fraud: Después de 30 segundos,<br/>Circuit intenta recuperarse
        
        Payment->>Fraud: POST /fraud/score
        activate Fraud
        
        Note over Fraud: Circuit HALF_OPEN:<br/>Allow 1 request (test)
        
        Fraud->>TensorFlow: POST /v1/models/fraud:predict
        activate TensorFlow
        TensorFlow-->>Fraud: {"score": 30}
        deactivate TensorFlow
        
        Note over Fraud: ✅ Success!<br/>Circuit CLOSES (healthy again)
        
        Fraud-->>Payment: 200 OK<br/>{"score": 30, "status": "APPROVED"}
        deactivate Fraud
    end
```

**Código Circuit Breaker**:

```java
@Service
public class FraudServiceClient {
    
    private final WebClient tensorflowClient;
    private final CircuitBreakerRegistry circuitBreakerRegistry;
    
    @CircuitBreaker(name = "tensorflowML", fallbackMethod = "fallbackScoring")
    @Timeout(value = 100, unit = TimeUnit.MILLISECONDS)
    public Mono<RiskScore> scorePayment(PaymentOrder payment) {
        return tensorflowClient.post()
            .uri("/v1/models/fraud:predict")
            .bodyValue(extractFeatures(payment))
            .retrieve()
            .bodyToMono(MLPrediction.class)
            .map(prediction -> new RiskScore(
                (int) (prediction.getScore() * 100),
                prediction.getScore() > 0.7 ? RiskStatus.BLOCKED : RiskStatus.APPROVED,
                ScoringSource.ML_MODEL
            ));
    }
    
    // Fallback: Scoring por reglas estáticas
    public Mono<RiskScore> fallbackScoring(PaymentOrder payment, Throwable t) {
        log.warn("TensorFlow unavailable, using rule-based scoring", t);
        
        int score = 0;
        
        // Regla 1: Monto alto
        if (payment.getAmount().getValue().compareTo(new BigDecimal("5000")) > 0) {
            score += 30;
        }
        
        // Regla 2: País de riesgo
        if (isHighRiskCountry(payment.getBeneficiary().getCountry())) {
            score += 40;
        }
        
        // Regla 3: Horario inusual (2 AM - 5 AM)
        LocalTime now = LocalTime.now();
        if (now.isAfter(LocalTime.of(2, 0)) && now.isBefore(LocalTime.of(5, 0))) {
            score += 20;
        }
        
        return Mono.just(new RiskScore(
            score,
            score >= 70 ? RiskStatus.BLOCKED : RiskStatus.APPROVED,
            ScoringSource.FALLBACK_RULES
        ));
    }
}
```

**Configuración Resilience4j**:

```yaml
resilience4j:
  circuitbreaker:
    instances:
      tensorflowML:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 30s
        failureRateThreshold: 50
        eventConsumerBufferSize: 10
```

---

## 📐 Flujo 4: Clearing Multi-Red con Retry Exponential Backoff

**Escenario**: Clearing Service envía pago a SWIFT. Red SWIFT temporalmente congestionada. Implementar reintentos inteligentes.

```mermaid
sequenceDiagram
    autonumber
    participant Payment as Payment Service
    participant Clearing as Clearing Service<br/>(Retry Engine)
    participant SWIFT as SWIFT Network
    participant DLQ as Dead Letter Queue<br/>(PostgreSQL)
    participant Ops as Ops Team<br/>(Slack Alert)
    
    Payment->>Clearing: POST /clearing/swift<br/>{"paymentId": "...", "amount": 1000}
    activate Clearing
    
    rect rgb(255, 240, 240)
        Note over Clearing: Intento 1 (immediate)
        
        Clearing->>SWIFT: pacs.008 (ISO 20022)
        activate SWIFT
        SWIFT--xClearing: 503 Service Unavailable<br/>(Red congestionada)
        deactivate SWIFT
        
        Note over Clearing: ⏱️ Wait 2 seconds (2^1)
    end
    
    rect rgb(255, 245, 240)
        Note over Clearing: Intento 2 (after 2s)
        
        Clearing->>SWIFT: pacs.008 (retry)
        activate SWIFT
        SWIFT--xClearing: 503 Service Unavailable
        deactivate SWIFT
        
        Note over Clearing: ⏱️ Wait 4 seconds (2^2)
    end
    
    rect rgb(255, 250, 240)
        Note over Clearing: Intento 3 (after 4s)
        
        Clearing->>SWIFT: pacs.008 (retry)
        activate SWIFT
        SWIFT--xClearing: Timeout (30s)
        deactivate SWIFT
        
        Note over Clearing: ⏱️ Wait 8 seconds (2^3)
    end
    
    rect rgb(255, 230, 230)
        Note over Clearing: 🔴 Max retries exhausted (3)
        
        Clearing->>DLQ: INSERT dead_letter<br/>{"payment_id", "error", "retries": 3}
        activate DLQ
        DLQ-->>Clearing: Saved
        deactivate DLQ
        
        Clearing->>Ops: POST /slack/webhook<br/>{"text": "SWIFT payment failed after 3 retries"}
        activate Ops
        Ops->>Ops: Alerta en canal #payments-ops
        deactivate Ops
        
        Clearing-->>Payment: 502 Bad Gateway<br/>{"error": "Clearing failed", "payment_id": "..."}
        deactivate Clearing
    end
    
    Note over Payment: Payment marcado como FAILED<br/>Saga compensation triggered
```

**Código Retry Engine**:

```java
@Service
public class ClearingRetryEngine {
    
    private final SWIFTAdapter swiftAdapter;
    private final DeadLetterQueueRepository dlqRepository;
    
    @Retryable(
        value = {ClearingNetworkException.class, TimeoutException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 2000, multiplier = 2, maxDelay = 10000)
    )
    public Mono<ClearingResponse> sendWithRetry(PaymentInstruction instruction) {
        return swiftAdapter.send(instruction)
            .doOnError(error -> 
                log.warn("Clearing attempt failed for paymentId={}, will retry",
                    instruction.getPaymentId(), error)
            )
            .timeout(Duration.ofSeconds(30));
    }
    
    @Recover
    public Mono<ClearingResponse> recoverFromFailure(
        Exception ex,
        PaymentInstruction instruction
    ) {
        log.error("All retry attempts exhausted for paymentId={}",
            instruction.getPaymentId(), ex);
        
        // Save to Dead Letter Queue
        DeadLetterMessage dlm = DeadLetterMessage.builder()
            .paymentId(instruction.getPaymentId())
            .serviceName("SWIFT")
            .errorMessage(ex.getMessage())
            .payload(instruction)
            .retriesExhausted(3)
            .createdAt(Instant.now())
            .build();
        
        return dlqRepository.save(dlm)
            .then(Mono.error(new ClearingFailedException(
                "Clearing failed after 3 retries: " + ex.getMessage()
            )));
    }
}
```

---

## 🎯 Resumen de Patrones de Integración Aplicados

| Patrón | Servicios | Problema Resuelto | Beneficio |
|--------|-----------|-------------------|-----------|
| **Saga Pattern (Temporal.io)** | Payment → Fraud → FX → Ledger → Clearing | Transacciones distribuidas sin 2PC | Rollback automático, observabilidad total |
| **Change Data Capture (CDC)** | Legacy Oracle ↔ Kafka ↔ New Services | Sincronización bidireccional sin dual-writes | Zero-downtime migration, lag < 2s |
| **Circuit Breaker** | Fraud Service ↔ TensorFlow | Cascading failures cuando ML cae | Degradación controlada con fallback |
| **Exponential Backoff** | Clearing ↔ SWIFT/PIX | Red externa temporalmente congestionada | Reduce carga en red, evita thundering herd |
| **Dead Letter Queue** | Clearing failed payments | Pérdida de transacciones tras fallos | Intervención manual, audit trail |
| **Event-Driven** | Todos los servicios vía Kafka | Acoplamiento temporal | Desacoplamiento, eventual consistency |
| **Request-Reply** | Payment → Fraud (sync) | Necesidad de respuesta inmediata | Latencia baja, decisión bloqueante |
| **Fire-and-Forget** | Fraud → Cassandra (async) | Writes no deben bloquear payment path | Throughput 50K writes/s |

---

## 📊 Métricas de Integración (SLAs)

| Integración | Latencia Target | Timeout | Retry Strategy | Fallback |
|-------------|----------------|---------|----------------|----------|
| **Payment → Fraud** | p95 < 50ms | 100ms | 3x exponential | Rule-based score |
| **Payment → FX** | p95 < 200ms | 5s | 2x linear | Cached rate (stale < 1min) |
| **Payment → Ledger** | p95 < 100ms | 10s | No retry | Saga compensation |
| **Clearing → SWIFT** | p95 < 5s | 30s | 3x exponential (2s, 4s, 8s) | Dead Letter Queue |
| **Clearing → PIX** | p95 < 2s | 15s | 3x exponential | Dead Letter Queue |
| **CDC Adapter → Kafka** | lag < 2s | N/A | N/A | Alert if lag > 10s |

---

**Fecha de Propuesta**: 24 de diciembre de 2025  
**Referencias**: [C3-Componentes.md](../3.1-C4-Model/C3-Componentes.md), [Despliegue.md](Despliegue.md)
```mermaid
sequenceDiagram
    actor Cliente
    participant MobileApp as Mobile App
    participant Gateway as API Gateway
    participant Payment as Payment Service
    participant Fraud as Fraud Service
    participant FX as FX Service
    participant Ledger as Ledger Service
    participant Clearing as Clearing Service
    participant Kafka
    participant DB as PostgreSQL

    Cliente->>MobileApp: Iniciar pago<br/>(500 EUR → 550 USD)
    MobileApp->>Gateway: POST /api/v1/payments
    activate Gateway
    
    Gateway->>Payment: CreatePayment(request)
    activate Payment
    
    Note over Payment: Paso 1: Crear orden
    Payment->>Payment: Validar request
    Payment->>DB: INSERT payment_order<br/>(status=DRAFT)
    Payment->>Kafka: Publish PaymentCreated
    
    Note over Payment,Fraud: Paso 2: Scoring de fraude
    Payment->>Fraud: POST /fraud/score
    activate Fraud
    Fraud->>Fraud: Análisis ML (50ms)
    Fraud-->>Payment: RiskScore(25, APPROVED)
    deactivate Fraud
    
    Payment->>DB: UPDATE status=VALIDATED
    Payment->>Kafka: Publish PaymentValidated
    
    Note over Payment,FX: Paso 3: Lock de FX
    Payment->>FX: POST /fx/lock
    activate FX
    FX->>FX: Lock rate<br/>(1.10 EUR/USD, TTL=5min)
    FX-->>Payment: FXLock(1.10, expires=...)
    deactivate FX
    
    Payment->>DB: UPDATE fx_locked=true
    Payment-->>Gateway: 201 Created<br/>{paymentId, status}
    deactivate Payment
    
    Gateway-->>MobileApp: Payment created
    deactivate Gateway
    MobileApp-->>Cliente: Pago creado,<br/>confirme ejecución
    
    Cliente->>MobileApp: Confirmar ejecución
    MobileApp->>Gateway: POST /payments/{id}/execute
    activate Gateway
    
    Gateway->>Payment: ExecutePayment(id)
    activate Payment
    
    Payment->>DB: SELECT payment_order
    Payment->>Payment: Validar estado=VALIDATED
    
    Note over Payment,Ledger: Paso 4: Registrar en Ledger
    Payment->>Ledger: POST /ledger/debit
    activate Ledger
    Ledger->>DB: INSERT ledger_event<br/>(DEBIT, 500 EUR)
    Ledger->>Kafka: Publish LedgerEntryCreated
    Ledger-->>Payment: Ledger Entry ID
    deactivate Ledger
    
    Note over Payment,Clearing: Paso 5: Enviar a clearing
    Payment->>Clearing: POST /clearing/swift
    activate Clearing
    Clearing->>Clearing: Convert to ISO 20022
    Clearing->>Clearing: Send to SWIFT Network
    Clearing-->>Payment: SWIFT Message ACK
    deactivate Clearing
    
    Payment->>DB: UPDATE status=SENT
    Payment->>Kafka: Publish PaymentExecuted
    Payment-->>Gateway: 202 Accepted
    deactivate Payment
    
    Gateway-->>MobileApp: Ejecución iniciada
    deactivate Gateway
    MobileApp-->>Cliente: Pago enviado,<br/>estimado 1-2 días
    
    Note over Kafka,Clearing: Paso 6: Settlement asíncrono
    Kafka->>Clearing: Consume SWIFT Confirmation
    activate Clearing
    Clearing->>Payment: POST /payments/{id}/settle
    activate Payment
    
    Payment->>DB: UPDATE status=SETTLED
    Payment->>Kafka: Publish PaymentSettled
    
    Payment->>Ledger: POST /ledger/settle
    activate Ledger
    Ledger->>DB: INSERT ledger_event<br/>(SETTLEMENT)
    Ledger->>Kafka: Publish LedgerSettled
    deactivate Ledger
    
    deactivate Payment
    Clearing->>MobileApp: Push notification
    deactivate Clearing
    
    MobileApp->>Cliente: Pago completado ✅
```
```

### Especificación Técnica

| Paso | Servicio | Operación | Timeout | Retry | Efecto |
|------|----------|-----------|---------|-------|--------|
| 1 | Payment | Crear orden | 100ms | No | Idempotente (UUID) |
| 2 | Fraud | Scoring | 50ms | Sí (3x) | Circuit Breaker |
| 3 | FX | Lock rate | 200ms | Sí (2x) | TTL 5 min |
| 4 | Ledger | Registro | 100ms | No | Event Sourced |
| 5 | Clearing | SWIFT send | 5s | Sí (5x exp backoff) | Async confirmación |
| 6 | Ledger | Settlement | 100ms | No | Eventual consistency |

**Garantías**:
- **Atomicidad**: Saga Pattern (compensación si falla paso 5)
- **Idempotencia**: UUID en headers (`Idempotency-Key`)
- **Orden**: Kafka partitions por `originatorId`

---

## 🚨 Flujo 2: Detección de Fraude con Bloqueo

### Diagrama de Secuencia

```mermaid
sequenceDiagram
    actor Cliente
    participant MobileApp as Mobile App
    participant Payment as Payment Service
    participant Fraud as Fraud Service
    participant Notif as Notification Service
    participant Compliance as Compliance Service
    participant Kafka

    Cliente->>MobileApp: Crear pago<br/>(10,000 USD a país riesgo)
    MobileApp->>Payment: POST /payments
    activate Payment
    
    Payment->>Fraud: POST /fraud/score
    activate Fraud
    
    Note over Fraud: Características:<br/>- Monto alto<br/>- País de riesgo<br/>- Horario inusual<br/>- IP nueva
    
    Fraud->>Fraud: Modelo ML predice<br/>RiskScore = 85
    Fraud->>Kafka: Publish FraudDetected<br/>(score=85, severity=HIGH)
    Fraud-->>Payment: RiskScore(85, BLOCKED)
    deactivate Fraud
    
    Payment->>Payment: Cambiar estado a BLOCKED
    Payment->>Kafka: Publish PaymentBlocked
    Payment-->>MobileApp: 400 Bad Request<br/>{"error": "Payment blocked"}
    deactivate Payment
    
    MobileApp-->>Cliente: ❌ Pago bloqueado<br/>por seguridad
    
    Note over Kafka,Compliance: Procesamiento asíncrono
    
    Kafka->>Notif: Consume PaymentBlocked
    activate Notif
    Notif->>Cliente: Email + SMS:<br/>"Detectamos actividad sospechosa"
    deactivate Notif
    
    Kafka->>Compliance: Consume FraudDetected
    activate Compliance
    Compliance->>Compliance: Crear caso SAR<br/>(Suspicious Activity Report)
    Compliance->>Compliance: Asignar a analista
    Compliance->>MobileApp: Notificar a Fraud Analyst
    deactivate Compliance
```

### Lógica de Scoring

```java
// En Fraud Service
public class FraudScoringModel {
    
    public RiskScore score(PaymentOrder payment) {
        int score = 0;
        
        // Regla 1: Monto
        if (payment.getAmount().getValue() > 5000) {
            score += 20;
        }
        
        // Regla 2: País de destino
        if (isHighRiskCountry(payment.getBeneficiary().getCountry())) {
            score += 30;
        }
        
        // Regla 3: Velocidad (10+ pagos en 1 hora)
        long recentPayments = paymentRepository
            .countByOriginatorInLastHour(payment.getOriginatorId());
        if (recentPayments >= 10) {
            score += 25;
        }
        
        // Regla 4: Modelo ML (TensorFlow)
        double mlScore = mlModel.predict(extractFeatures(payment));
        score += (int) (mlScore * 25);
        
        return new RiskScore(
            score, 
            score >= 70 ? RiskStatus.BLOCKED : RiskStatus.APPROVED
        );
    }
}
```

---

## 🔄 Flujo 3: Reconciliación Nocturna con Saga

### Diagrama de Secuencia

```mermaid
sequenceDiagram
    participant Cron as Cron Job
    participant Recon as Reconciliation Service
    participant Clearing as Clearing Service
    participant Ledger as Ledger Service
    participant Notif as Notification Service
    participant Kafka
    participant DB as PostgreSQL

    Cron->>Recon: Trigger diario (03:00 UTC)
    activate Recon
    
    Note over Recon,Clearing: Paso 1: Obtener movimientos SWIFT
    Recon->>Clearing: GET /clearing/swift/movements?date=yesterday
    activate Clearing
    Clearing->>Clearing: Descargar MT940<br/>(SWIFT statement)
    Clearing-->>Recon: SwiftMovements[]<br/>(1,523 transacciones)
    deactivate Clearing
    
    Note over Recon,Ledger: Paso 2: Obtener ledger interno
    Recon->>Ledger: GET /ledger/entries?date=yesterday
    activate Ledger
    Ledger->>DB: SELECT * FROM ledger_events
    Ledger-->>Recon: LedgerEntries[]<br/>(1,520 transacciones)
    deactivate Ledger
    
    Note over Recon: Paso 3: Comparar
    Recon->>Recon: Reconciliar:<br/>- SWIFT: 1,523<br/>- Ledger: 1,520<br/>- Diferencias: 3
    Recon->>Recon: Identificar discrepancias:<br/>- TX-001: En SWIFT, no en Ledger<br/>- TX-002: En Ledger, no en SWIFT<br/>- TX-003: Montos difieren
    
    Note over Recon,Ledger: Paso 4: Saga de compensación
    
    alt TX en SWIFT pero no en Ledger
        Recon->>Ledger: POST /ledger/adjust<br/>(crear entrada faltante)
        activate Ledger
        Ledger->>DB: INSERT ledger_event<br/>(ADJUSTMENT)
        Ledger->>Kafka: Publish LedgerAdjusted
        deactivate Ledger
    else TX en Ledger pero no en SWIFT
        Recon->>Clearing: POST /clearing/investigate
        activate Clearing
        Clearing->>Clearing: Crear caso de investigación
        Clearing-->>Recon: Case ID
        deactivate Clearing
    else Montos difieren
        Recon->>Recon: Marcar para revisión manual
    end
    
    Note over Recon,Kafka: Paso 5: Notificar resultados
    Recon->>Kafka: Publish ReconciliationCompleted<br/>(matched=1520, discrepancies=3)
    
    Kafka->>Notif: Consume ReconciliationCompleted
    activate Notif
    Notif->>Notif: Enviar reporte a Treasury
    deactivate Notif
    
    Recon->>DB: INSERT reconciliation_report
    deactivate Recon
```

### Saga Pattern

```java
@Service
public class ReconciliationSaga {
    
    public ReconciliationResult execute(LocalDate date) {
        // 1. Iniciar saga
        SagaContext context = new SagaContext();
        
        try {
            // Step 1: Obtener movimientos
            List<SwiftMovement> swiftData = clearingService.getMovements(date);
            context.recordStep("SWIFT_FETCH", swiftData);
            
            List<LedgerEntry> ledgerData = ledgerService.getEntries(date);
            context.recordStep("LEDGER_FETCH", ledgerData);
            
            // Step 2: Reconciliar
            ReconciliationReport report = reconcile(swiftData, ledgerData);
            context.recordStep("RECONCILE", report);
            
            // Step 3: Ajustar discrepancias
            for (Discrepancy disc : report.getDiscrepancies()) {
                adjustDiscrepancy(disc, context);
            }
            
            // Success
            return ReconciliationResult.success(report);
            
        } catch (Exception e) {
            // Compensación: Rollback de ajustes
            compensate(context);
            return ReconciliationResult.failure(e);
        }
    }
    
    private void compensate(SagaContext context) {
        // Revertir en orden inverso
        context.getCompletedSteps().reversed().forEach(step -> {
            switch (step.getName()) {
                case "LEDGER_ADJUST":
                    ledgerService.revertAdjustment(step.getData());
                    break;
                case "CLEARING_UPDATE":
                    clearingService.revertUpdate(step.getData());
                    break;
            }
        });
    }
}
```

---

## 🎯 Patrones de Integración Aplicados

| Patrón | Uso | Beneficio |
|--------|-----|-----------|
| **Request-Reply** | Payment → Fraud | Sincrónico, bajo latencia |
| **Fire-and-Forget** | Payment → Kafka | Desacoplamiento |
| **Saga** | Reconciliación | Consistencia distribuida |
| **Circuit Breaker** | Fraud, FX | Resiliencia ante fallos |
| **Idempotency** | Todos los POSTs | Reintentos seguros |
| **Event Sourcing** | Ledger | Auditoría completa |

---

**Próximo Paso**: → `../3.3-Patrones-Tacticas.md`

---

**Última actualización**: 7 de diciembre de 2025
