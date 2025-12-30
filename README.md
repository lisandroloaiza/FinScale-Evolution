# FinScale Evolution - Arquitectura de Modernización

> **Metodología**: Why Driven Design (WhyDD) - 6 Fases de Análisis Estratégico y Técnico  
> **Objetivo**: Escalar de 2K TPS → 1M TPS | 99.5% → 99.999% disponibilidad | Time-to-Market de 4 meses → 2 semanas
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
- [Cómo Leer esta Documentación](#cómo-leer-esta-documentación)

---

## 🎯 Contexto del Problema

**FinScale** es una plataforma fintech que procesa transferencias internacionales P2P. El sistema legacy presenta limitaciones críticas:

### Sistema Actual (AS-IS)
- **Monolito J2EE**: 1.2M líneas de código Java 8
- **Base de Datos**: Oracle 11g centralizada (shared database pattern)
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
| [1.2 Drivers de Arquitectura](01-Entendimiento-Negocio/1.2-Drivers-Arquitectura.md) | 22 Quality Attributes priorizados (Functionality, Usability, Reliability, Performance, Security) | Escalabilidad (1M TPS), Disponibilidad (99.999%), Deployability (CI/CD), Time-to-Market (2 semanas) |
| [1.3 Business Capabilities](01-Entendimiento-Negocio/1.3-Business-Capabilities.md) | Mapa jerárquico de capacidades de negocio | 5 áreas: Gestión Pagos, Procesamiento Transacciones, Gestión Clientes, Compliance, Operaciones |

### Fase 2: Diseño Estratégico (DDD) 🎨
> **Objetivo**: Aplicar Domain-Driven Design para identificar bounded contexts y subdominios

| Documento | Descripción | Contenido Clave |
|-----------|-------------|-----------------|
| [2.1 Core Domain Chart](02-Diseño-Estrategico/2.1-Core-Domain-Chart.md) | Clasificación de subdominios por complejidad y diferenciación | 3 CORE: General Ledger (0.92), Fraud Detection (0.82), Payment Execution (0.75) |
| [2.2 Bounded Contexts](02-Diseño-Estrategico/2.2-Bounded-Contexts.md) | Definición de 10 bounded contexts con lenguaje ubicuo | Payment Execution, General Ledger, Fraud Detection, FX Rate Management, Treasury & Settlement, Reconciliation Engine, Customer & Account, Notifications, Reporting & Analytics, Identity & Access |
| [2.3 Context Map](02-Diseño-Estrategico/2.3-Context-Map.md) | Relaciones entre contextos (Partnership, Customer-Supplier, Conformist, ACL) | 15 relaciones mapeadas con patrones DDD tácticos |
| [2.4 Modelo de Dominio](02-Diseño-Estrategico/2.4-Modelo-Dominio.md) | Agregados, entidades y value objects por contexto | Payment (Aggregate Root), FXQuote (Entity), AccountBalance (Value Object) |

### Fase 3: Diseño Técnico 🔧
> **Objetivo**: Traducir diseño estratégico a arquitectura técnica ejecutable

| Documento | Descripción | Contenido Clave |
|-----------|-------------|-----------------|
| [3.1 C4 Model - Contexto](03-Diseño-Tecnico/3.1-C4-Model/C1-Contexto.md) | Vista de sistema con actores externos | Cliente, App Móvil, FinScale Platform, Bancos, Reguladores |
| [3.1 C4 Model - Contenedores](03-Diseño-Tecnico/3.1-C4-Model/C2-Contenedores.md) | Arquitectura de microservicios con event backbone | 10 microservices + API Gateway + Kafka + Databases |
| [3.1 C4 Model - Componentes](03-Diseño-Tecnico/3.1-C4-Model/C3-Componentes.md) | Estructura interna de Payment Execution Context | REST Controllers, Command Handlers, Event Handlers, Domain Services |
| [3.2 UML - Despliegue](03-Diseño-Tecnico/3.2-UML/Despliegue.md) | Topología AWS multi-región | EKS Clusters, RDS Multi-AZ, MSK, Route53, CloudFront |
| [3.2 UML - Infraestructura](03-Diseño-Tecnico/3.2-UML/Infraestructura.md) | VPC, Subnets, Security Groups, NAT Gateway | 3 AZs, subnets públicas/privadas, bastion hosts |
| [3.2 UML - Integración](03-Diseño-Tecnico/3.2-UML/Integracion.md) | Flujos de integración asíncrona vía Kafka | Event-Driven Communication, CQRS separando comandos/queries |
| [3.3 Patrones y Tácticas](03-Diseño-Tecnico/3.3-Patrones-Tacticas.md) | 15 patrones arquitectónicos aplicados | CQRS, Event Sourcing, Saga, Outbox, API Gateway, Circuit Breaker |
| [3.4 Stack Tecnológico](03-Diseño-Tecnico/3.4-Stack-Tecnologico.md) | Decisiones técnicas detalladas con justificación | Spring Boot 3.2, EKS 1.28, PostgreSQL 15, Kafka 3.5, Redis 7.2 |

### Fase 4: Infraestructura y Resiliencia ☁️
> **Objetivo**: Diseñar infraestructura cloud y estrategia de migración

| Documento | Descripción | Contenido Clave |
|-----------|-------------|-----------------|
| [4.1 Arquitectura Cloud](04-Infraestructura-Resiliencia/4.1-Arquitectura-Cloud.md) | Diseño AWS multi-región con DR | 3 regiones (us-east-1, eu-west-1, sa-east-1), RTO 15min, RPO 1min |
| [4.2 Patrones de Resiliencia](04-Infraestructura-Resiliencia/4.2-Patrones-Resiliencia.md) | Implementación de tácticas de disponibilidad | Circuit Breaker (Resilience4j), Rate Limiting (5K req/s), Bulkhead, Retry con Exponential Backoff |
| [4.3 Estrategia Migración Strangler](04-Infraestructura-Resiliencia/4.3-Estrategia-Migracion-Strangler.md) | Roadmap de 15 meses en 3 fases con zero downtime | Fase 1: Payment+Fraud (5 meses), Fase 2: Ledger+FX (6 meses), Fase 3: Resto (4 meses). Debezium CDC para sincronización bidireccional |

### Fase 5: Gobierno y Liderazgo 👥
> **Objetivo**: Definir estructura de equipos, APIs y gobernanza de datos

| Documento | Descripción | Contenido Clave |
|-----------|-------------|-----------------|
| [5.1 Análisis ATAM](05-Gobierno-Liderazgo/5.1-Analisis-ATAM.md) | Architecture Tradeoff Analysis Method con 5 escenarios de calidad | Performance vs Consistency, Scalability vs Complexity, Security vs Usability, Availability vs Cost, Deployability vs Safety |
| [5.2 Gobierno de APIs](05-Gobierno-Liderazgo/5.2-Gobierno-APIs.md) | Estándares de diseño y versionamiento de APIs | REST Level 2, OpenAPI 3.1, Semantic Versioning, Rate Limiting, OAuth2 + JWT |
| [5.3 Data Governance](05-Gobierno-Liderazgo/5.3-Data-Governance.md) | Políticas de gobierno de datos y privacidad | Ownership (Payment Context → Payment DB), Encryption (AES-256), Retention (7 años), GDPR Right to Erasure |
| [5.4 Estrategia de Equipos](05-Gobierno-Liderazgo/5.4-Estrategia-Equipos.md) | Team Topologies con 6 Stream-Aligned Teams | 89 personas: Payment Core Team (9p), Ledger & Compliance (9p), Fraud & Risk (8p), Treasury & Clearing (7p), Customer & Compliance (6p), Platform Engineering (15p) |

### Fase 6: Anexo - Motor de Dispersión 💰
> **Objetivo**: Aplicar 6 patrones GoF para resolver problema de pagos masivos (500K/día)

| Documento | Descripción | Problema que Resuelve |
|-----------|-------------|----------------------|
| [6.1 Builder + Prototype](06-Anexo-Motor-Dispersion/1-Builder-Prototype.md) | Construcción de órdenes de pago complejas | 40+ atributos, 90% campos recurrentes, validaciones condicionales |
| [6.2 Flyweight](06-Anexo-Motor-Dispersion/2-Flyweight.md) | Optimización de memoria para metadatos | 5.2 GB → 1.85 GB (reducción 64%) compartiendo 10 monedas + 50 países + 200 bancos |
| [6.3 Chain of Responsibility](06-Anexo-Motor-Dispersion/3-Chain-Responsibility.md) | Validación extensible de pagos | 4 validadores (Sintaxis, Balance, Sanciones, Velocity) con orden dinámico |
| [6.4 State](06-Anexo-Motor-Dispersion/4-State.md) | Gestión de ciclo de vida de pagos | 6 estados (Draft → Validated → FXLocked → Sent → Settled/Failed) con transiciones válidas |
| [6.5 Bridge](06-Anexo-Motor-Dispersion/5-Bridge.md) | Abstracción de canales de pago | 2 abstracciones × 3 implementaciones (Urgente/Normal × SWIFT/Ripple/Local) |
| [6.6 Observer](06-Anexo-Motor-Dispersion/6-Observer.md) | Notificaciones asíncronas multi-canal | 4+ observers (Accounting, Notifications, Analytics, Fraud) desacoplados del flujo principal |
| [6.7 Integración de Patrones](06-Anexo-Motor-Dispersion/7-Integracion.md) | Orquestación completa del flujo de dispersión | Flujo end-to-end de 500K pagos diarios con todos los patrones integrados |

