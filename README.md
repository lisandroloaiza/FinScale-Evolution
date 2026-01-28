# FinScale Evolution - Arquitectura de Modernización

> **Metodología**: Why Driven Design (WhyDD) - 6 Fases de Análisis Estratégico y Técnico  
> **Objetivo**: Escalar de 2K TPS → 1M TPS | 99.5% → 99.999% disponibilidad | Time-to-Market de 4 meses → 2 semanas

---

## 📋 Tabla de Contenidos

- [Contexto del Problema](#contexto-del-problema)
- [Arquitectura de la Solución](#arquitectura-de-la-solución)
- [Navegación por Fases](#navegación-por-fases)
- [Decisiones Clave](#decisiones-clave)
- [Stack Tecnológico](#stack-tecnológico)
- [Equipos y Organización](#equipos-y-organización)

---

## 🎯 Contexto del Problema

**FinScale** es una plataforma fintech que procesa transferencias internacionales P2P. El sistema legacy presenta limitaciones críticas:

### Sistema Actual (AS-IS)
- **Monolito J2EE**: 1.2M líneas de código Java 8
- **Base de Datos**: Oracle centralizada (shared database pattern)
- **Lógica de Negocio**: 40% en stored procedures PL/SQL
- **Capacidad**: 2,000 TPS máximo
- **Disponibilidad**: 99.5% (43.8h downtime/año)
- **Time-to-Market**: 4 meses por feature
- **Despliegues**: Ventana de 6 horas cada 3 meses

### Drivers de Negocio
1. **Crecimiento explosivo**: Escalar a 1,000,000 TPS (500x)
2. **Disponibilidad crítica**: 99.999% (5.26 min downtime/año)
3. **Agilidad**: Reducir time-to-market de 4 meses → 2 semanas
4. **Regulatorio**: Cumplimiento PCI-DSS, GDPR, SOC2
5. **Global**: Soporte multi-región (US-EAST, EU-WEST, LATAM)

---

## 🏗️ Arquitectura de la Solución

### Paradigma Objetivo (TO-BE)
- **Arquitectura**: Event-Driven Microservices con DDD
- **10 Bounded Contexts**: 3 CORE + 6 SUPPORTING + 1 GENERIC
- **Stack Cloud-Native**: Spring Boot 3.2 WebFlux + Kubernetes (EKS 1.28)
- **Datos Poliglota**: PostgreSQL 15, Cassandra 4.1, Redis 7.2, TimescaleDB
- **Event Streaming**: Apache Kafka 3.5 (MSK) con CQRS/Event Sourcing
- **Resiliencia**: Circuit Breaker, Rate Limiting, Bulkhead, Retry con Backoff
- **Observabilidad**: OpenTelemetry + Grafana Stack

### Estrategia de Migración
- **Patrón**: Strangler Fig Pattern
- **Duración**: 15 meses en 3 fases
- **Sincronización**: Debezium CDC bidireccional (zero downtime)
- **Priorización**: Core Domain Chart (Ledger 0.92, Fraud 0.82, Payment 0.75)

---

## 📚 Navegación por Fases

### Fase 1: Entendimiento del Negocio 🔍
> **Objetivo**: Visualizar procesos críticos, identificar drivers y mapear capacidades de negocio

| Documento | Descripción | Contenido Clave |
|-----------|-------------|-----------------|
| [1.1 Domain Storytelling](01-Entendimiento-Negocio/1.1-Domain-Storytelling.md) | Narrativas visuales de 3 procesos críticos con diagramas Mermaid | P2P Transfer (28 interacciones), Reconciliación Batch (6h ventana), Fraude Real-Time (<100ms SLA) |
| [1.2 Drivers de Arquitectura](01-Entendimiento-Negocio/1.2-Drivers-Arquitectura.md) | 6 drivers principales traducidos desde objetivos estratégicos | Escalabilidad (1M TPS), Disponibilidad (24/7), Time-to-Market (2 semanas), Resiliencia, Modernización, Cumplimiento |
| [1.3 Business Capabilities](01-Entendimiento-Negocio/1.3-Business-Capabilities.md) | Mapa jerárquico de capacidades de negocio | 4 áreas: Customer Experience & Growth, Payment Processing & Execution, Risk & Compliance, Financial Accounting & Reconciliation |

### Fase 2: Diseño Estratégico (DDD) 🎨
> **Objetivo**: Aplicar Domain-Driven Design para identificar bounded contexts y subdominios

| Documento | Descripción | Contenido Clave |
|-----------|-------------|-----------------|
| [2.1 Core Domain Chart](02-Diseño-Estrategico/2.1-Core-Domain-Chart.md) | Clasificación de subdominios por complejidad y diferenciación | 3 CORE: General Ledger (0.92), Fraud Detection (0.82), Payment Execution (0.75) |
| [2.2 Bounded Contexts](02-Diseño-Estrategico/2.2-Bounded-Contexts.md) | 10 bounded contexts organizados por complejidad | 3 CORE (Payment Execution, General Ledger, Fraud Detection), 4 SUPPORTING Alta Complejidad (Reconciliation, Clearing & Settlement, Treasury & FX, Regulatory Reporting), 2 SUPPORTING Baja Complejidad (Customer Management, Screening & Compliance), 1 GENERIC (Identity & Access) |
| [2.3 Context Map](02-Diseño-Estrategico/2.3-Context-Map.md) | Relaciones entre contextos con patrones DDD | 10 relaciones mapeadas (Partnership, Customer-Supplier, ACL, OHS, Conformist) |
| [2.4 Modelo de Dominio](02-Diseño-Estrategico/2.4-Modelo-Dominio.md) | Patrones tácticos DDD aplicados por contexto | Entities (PaymentOrder, LedgerEntry, Customer), Value Objects (Money, IBAN, Address), Aggregate Roots (PaymentOrder, LedgerEntry), Domain Events (PaymentExecuted, FraudDetected), Repositories, Domain Services |

### Fase 3: Diseño Técnico 🔧
> **Objetivo**: Traducir diseño estratégico a arquitectura técnica ejecutable

| Documento | Descripción | Contenido Clave |
|-----------|-------------|-----------------|
| [3.1 C4 Model - Contexto](03-Diseño-Tecnico/3.1-C4-Model/C1-Contexto.md) | Vista de sistema con actores y sistemas externos | Actores (Clientes, Operadores), Sistema FinScale (Spring Boot WebFlux, 10 Bounded Contexts), Redes de Pago (SWIFT, SEPA, PIX), Proveedores (KYC, FX), Legacy (Monolito J2EE durante migración) |
| [3.1 C4 Model - Contenedores](03-Diseño-Tecnico/3.1-C4-Model/C2-Contenedores.md) | Descomposición en aplicaciones y servicios | Frontend (React Native, React Web), API Gateway (Kong), 10 Microservicios (Payment, Ledger, Fraud, Customer, FX, Clearing, Reconciliation), Integration Layer (Legacy Facade, HSM Proxy, CDC Adapter), Messaging (Kafka), Databases (PostgreSQL, TimescaleDB, Cassandra, Redis), Observability (Prometheus, Jaeger, ELK) |
| [3.1 C4 Model - Componentes](03-Diseño-Tecnico/3.1-C4-Model/C3-Componentes.md) | Arquitectura interna de servicios CORE | Payment Service (Hexagonal Architecture, Saga con Temporal.io), Ledger Service (Event Sourcing + TimescaleDB), Fraud Service (ML Scoring TensorFlow < 100ms), Integration Layer (Legacy Facade ACL, HSM Proxy, CDC Adapter) |
| [3.2 UML - Despliegue](03-Diseño-Tecnico/3.2-UML/Despliegue.md) | Distribución física en Kubernetes multi-AZ | 3 AZ (A, B, C) con EKS Node Groups, RDS Multi-AZ (PostgreSQL, TimescaleDB), Cassandra 9-node cluster, Redis Cluster, Kafka 3 brokers, Integration Layer dedicado, Direct Connect a Legacy datacenter |
| [3.2 UML - Infraestructura](03-Diseño-Tecnico/3.2-UML/Infraestructura.md) | VPC, Subnets, Security Groups, NAT Gateway | 3 AZs, subnets públicas/privadas, bastion hosts |
| [3.2 UML - Integración](03-Diseño-Tecnico/3.2-UML/Integracion.md) | Flujos de integración asíncrona vía Kafka | Event-Driven Communication, CQRS separando comandos/queries |
| [3.3 Patrones y Tácticas](03-Diseño-Tecnico/3.3-Patrones-Tacticas.md) | 7 patrones arquitectónicos con justificación WHY-WHAT-HOW | Strangler Fig, ACL, CDC, CQRS, Event Sourcing, Saga (Temporal.io), Circuit Breaker, Bulkhead |
| [3.4 Stack Tecnológico](03-Diseño-Tecnico/3.4-Stack-Tecnologico.md) | Justificación cuantitativa de decisiones tecnológicas | Spring Boot 3.2 WebFlux (850K req/s, 12ms p99), Java 21 (Virtual Threads), R2DBC (reactive DB drivers), EKS 1.28, PostgreSQL 15, TimescaleDB (event sourcing), Cassandra 4.1 (high-throughput writes), Redis 7.2 (cache + sessions), Kafka 3.5 MSK, Temporal.io (saga orchestration) |

### Fase 4: Infraestructura y Resiliencia ☁️
> **Objetivo**: Diseñar infraestructura cloud y estrategia de migración

| Documento | Descripción | Contenido Clave |
|-----------|-------------|-----------------|
| [4.1 Arquitectura Cloud](04-Infraestructura-Resiliencia/4.1-Arquitectura-Cloud.md) | Diseño multi-AZ con estrategia DR | AWS us-east-1 Primary (3 AZ: A/B/C con 33% capacity cada uno, 27 EKS nodes por AZ), us-west-2 DR Pasivo (RTO 1h, RPO 15min), CloudFront CDN global, Route53 Geo DNS, Direct Connect 1Gbps a Legacy datacenter |
| [4.2 Patrones de Resiliencia](04-Infraestructura-Resiliencia/4.2-Patrones-Resiliencia.md) | 5 patrones con Resilience4j para 99.999% uptime | Retry con Exponential Backoff (3 attempts, 100ms→200ms→400ms), Circuit Breaker (50% failure threshold, 60s timeout), Rate Limiting (5K req/s por servicio), Bulkhead (thread pools aislados), Chaos Engineering (validación continua) |
| [4.3 Estrategia Migración Strangler](04-Infraestructura-Resiliencia/4.3-Estrategia-Migracion-Strangler.md) | Roadmap de 15 meses en 3 fases con zero downtime | Fase 1: Payment+Fraud (5 meses), Fase 2: Ledger+FX (6 meses), Fase 3: Resto (4 meses). Debezium CDC para sincronización bidireccional |

### Fase 5: Gobierno y Liderazgo 👥
> **Objetivo**: Definir estructura de equipos, APIs y gobernanza de datos

| Documento | Descripción | Contenido Clave |
|-----------|-------------|-----------------|
| [5.1 Análisis ATAM](05-Gobierno-Liderazgo/5.1-Analisis-ATAM.md) | Architecture Tradeoff Analysis Method con 5 escenarios | Alta Disponibilidad (Peak Traffic), Recuperación ante Fallo AZ, GDPR Right to Erasure, Fraud Detection Fallback, Consistencia Transaccional (Saga) |
| [5.2 Gobierno de APIs](05-Gobierno-Liderazgo/5.2-Gobierno-APIs.md) | Políticas de diseño y lifecycle de APIs | REST Level 2 (HTTPS only, JSON, plural nouns), OpenAPI 3.0 contracts, Semantic Versioning (URL path /v1, /v2), Rate Limiting (1K req/min tier-based), OAuth2 + JWT (RS256), Idempotency keys, HATEOAS links |
| [5.3 Data Governance](05-Gobierno-Liderazgo/5.3-Data-Governance.md) | Clasificación, protección y lifecycle de datos | 4 niveles sensibilidad (PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED), Encryption at-rest AES-256 + in-transit TLS 1.3, PII tokenization, Retention (7 años transacciones, anonimización GDPR), AWS KMS key rotation 365 días |
| [5.4 Estrategia de Equipos](05-Gobierno-Liderazgo/5.4-Estrategia-Equipos.md) | Team Topologies con 6 Stream-Aligned + 3 Complicated-Subsystem + 3 Platform Teams | 79 personas: Payment Core (9p), Ledger & Compliance (9p), Customer & Compliance (6p), Fraud & Risk (9p), Treasury & Clearing (7p), Integration (7p), ML/AI (5p), Security (4p), Search (4p), Cloud Platform (8p), Data Platform (5p), Observability (6p) |

### Fase 6: Anexo - Motor de Dispersión 💰
> **Objetivo**: Aplicar 6 patrones GoF para resolver problema de pagos masivos (500K/día)

| Documento | Descripción | Problema que Resuelve |
|-----------|-------------|----------------------|
| [6.1 Builder + Prototype](06-Anexo-Motor-Dispersion/1-Builder-Prototype.md) | Construcción de órdenes de pago complejas | 40+ atributos, 90% campos recurrentes, validaciones condicionales |
| [6.2 Flyweight](06-Anexo-Motor-Dispersion/2-Flyweight.md) | Optimización de memoria para metadatos compartidos | 2.4 GB → 820 KB (reducción 99.97%) compartiendo 10 monedas + 50 países + 200 bancos |
| [6.3 Chain of Responsibility](06-Anexo-Motor-Dispersion/3-Chain-Responsibility.md) | Validación extensible de pagos | 4 validadores (Sintaxis, Balance, Sanciones, Velocity) con orden dinámico |
| [6.4 State](06-Anexo-Motor-Dispersion/4-State.md) | Gestión de ciclo de vida de pagos | 6 estados (Draft → Validated → FXLocked → Sent → Settled/Failed) con transiciones válidas |
| [6.5 Bridge](06-Anexo-Motor-Dispersion/5-Bridge.md) | Abstracción de canales de pago | 2 abstracciones × 3 implementaciones (Urgente/Normal × SWIFT/Ripple/Local) |
| [6.6 Observer](06-Anexo-Motor-Dispersion/6-Observer.md) | Notificaciones asíncronas multi-canal | 4+ observers (Accounting, Notifications, Analytics, Fraud) desacoplados del flujo principal |
| [6.7 Integración de Patrones](06-Anexo-Motor-Dispersion/7-Integracion.md) | Orquestación end-to-end de los 6 patrones GoF | MassPaymentProcessor integra Builder+Prototype (construcción/clonación), Flyweight (260 objetos compartidos), Chain (4 validators pipeline), State (6 estados lifecycle), Bridge (2×3 gateways), Observer (4+ listeners Kafka) procesando 500K instrucciones diarias |

