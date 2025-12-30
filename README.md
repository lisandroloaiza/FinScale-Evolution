# FinScale Evolution - Arquitectura de Modernización

> **Kata Arquitectónico**: Modernización de plataforma fintech de Java 8 monolítico a arquitectura cloud-native de microservicios  
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

---

## 🎯 Decisiones Clave

### Arquitectura
- **✅ Event-Driven Microservices**: Escalabilidad independiente, eventual consistency
- **✅ CQRS + Event Sourcing**: Segregación lectura/escritura, auditoría completa
- **✅ Saga Pattern**: Transacciones distribuidas con compensación
- **✅ API Gateway**: Single entry point, rate limiting, autenticación centralizada

### Datos
- **✅ Database per Service**: Cada bounded context con su BD independiente
- **✅ Poliglota Persistence**: PostgreSQL (transaccional), Cassandra (series temporales), Redis (caché)
- **✅ Event Store**: TimescaleDB para event sourcing (optimizado para time-series)
- **✅ CDC con Debezium**: Sincronización monolito ↔ microservicios durante migración

### Infraestructura
- **✅ Kubernetes (EKS)**: Orquestación de contenedores, auto-scaling, self-healing
- **✅ Multi-Región**: us-east-1 (primary), eu-west-1, sa-east-1 (DR + latencia)
- **✅ Service Mesh (Istio)**: mTLS, observabilidad, traffic management
- **✅ GitOps (ArgoCD)**: Infraestructura como código, deployments declarativos

### Resiliencia
- **✅ Circuit Breaker**: Resilience4j con 50% threshold, 60s timeout
- **✅ Rate Limiting**: 5,000 req/s por microservicio
- **✅ Bulkhead**: Thread pools aislados (Payment: 200, Fraud: 100, Ledger: 150)
- **✅ Chaos Engineering**: Experimentos automáticos semanales (Chaos Monkey)

---

## 🛠️ Stack Tecnológico

### Backend
- **Framework**: Spring Boot 3.2.x + Spring WebFlux (reactive)
- **Lenguaje**: Java 21 LTS (Virtual Threads)
- **Build**: Gradle 8.5 con multi-module projects

### Datos
- **Transaccional**: PostgreSQL 15.x (RDS Multi-AZ)
- **Caché**: Redis 7.2 (ElastiCache con cluster mode)
- **Time-Series**: TimescaleDB 2.13 (event store)
- **NoSQL**: Apache Cassandra 4.1 (reporting)

### Mensajería
- **Event Streaming**: Apache Kafka 3.5 (MSK)
- **Schema Registry**: Confluent Schema Registry con Avro
- **Kafka Streams**: Procesamiento de eventos en tiempo real

### Cloud (AWS)
- **Compute**: EKS 1.28 (Kubernetes managed)
- **Networking**: VPC, ALB, NLB, Route53, CloudFront
- **Storage**: S3, EBS, EFS
- **Security**: KMS, Secrets Manager, IAM, Security Groups

### Observabilidad
- **Tracing**: OpenTelemetry + Jaeger
- **Metrics**: Prometheus + Grafana
- **Logs**: Fluent Bit → CloudWatch Logs
- **APM**: AWS X-Ray

### CI/CD
- **Source Control**: GitHub Enterprise
- **CI**: GitHub Actions
- **CD**: ArgoCD (GitOps)
- **Artifacts**: AWS ECR (Docker images)
- **IaC**: Terraform 1.6 + Terragrunt

---

## 👥 Equipos y Organización

### Topología: Team Topologies
- **6 Stream-Aligned Teams**: Ownership de bounded contexts end-to-end
- **1 Platform Team**: Infraestructura compartida (EKS, Kafka, Observabilidad)
- **1 Enabling Team**: Arquitectura, seguridad, mejores prácticas

### Stream-Aligned Teams (55 personas)
| Team | Bounded Contexts | Personas | Stack Principal |
|------|------------------|----------|----------------|
| **Payment Core Team** | Payment Execution | 9 | Spring WebFlux, PostgreSQL, Kafka |
| **Ledger & Compliance** | General Ledger | 9 | Spring Boot, PostgreSQL, Event Sourcing |
| **Fraud & Risk** | Fraud Detection | 8 | Spring WebFlux, Cassandra, Redis, ML (Python) |
| **Treasury & Clearing** | Treasury & Settlement, Reconciliation | 7 | Spring Boot, PostgreSQL, Kafka Streams |
| **Customer & Compliance** | Customer & Account, Identity & Access | 6 | Spring WebFlux, PostgreSQL, Keycloak |
| **Integration & Notifications** | FX Rate, Notifications, Reporting | 16 | Spring WebFlux, Redis, Cassandra |

### Platform Team (15 personas)
- **Infrastructure**: EKS, Networking, Security
- **Data Platform**: Kafka, Databases, Backups
- **Observability**: Monitoring, Logging, Tracing
- **CI/CD**: Pipelines, GitOps, Environments

### Enabling Team (19 personas)
- **Architecture Office**: Decisiones arquitectónicas, ADRs, Tech Radar
- **Security Office**: PCI-DSS, Penetration Testing, Threat Modeling
- **QA Office**: Testing Strategy, Performance Testing, Chaos Engineering

---

## 📖 Cómo Leer esta Documentación

### Para Stakeholders de Negocio
1. Comienza con [1.1 Domain Storytelling](01-Entendimiento-Negocio/1.1-Domain-Storytelling.md) para entender los procesos actuales
2. Revisa [1.2 Drivers de Arquitectura](01-Entendimiento-Negocio/1.2-Drivers-Arquitectura.md) para ver los objetivos de negocio
3. Consulta [4.3 Estrategia de Migración](04-Infraestructura-Resiliencia/4.3-Estrategia-Migracion-Strangler.md) para el roadmap de 15 meses

### Para Arquitectos
1. Estudia Fase 2 completa (Diseño Estratégico DDD) para entender los bounded contexts
2. Revisa [3.3 Patrones y Tácticas](03-Diseño-Tecnico/3.3-Patrones-Tacticas.md) para las decisiones arquitectónicas
3. Analiza [5.1 Análisis ATAM](05-Gobierno-Liderazgo/5.1-Analisis-ATAM.md) para los trade-offs

### Para Tech Leads
1. Comienza con [2.2 Bounded Contexts](02-Diseño-Estrategico/2.2-Bounded-Contexts.md) para entender tu dominio
2. Revisa [3.4 Stack Tecnológico](03-Diseño-Tecnico/3.4-Stack-Tecnologico.md) para decisiones técnicas
3. Consulta [5.2 Gobierno de APIs](05-Gobierno-Liderazgo/5.2-Gobierno-APIs.md) para estándares de implementación

### Para Desarrolladores
1. Lee [2.4 Modelo de Dominio](02-Diseño-Estrategico/2.4-Modelo-Dominio.md) para los agregados y entidades
2. Revisa [3.1 C4 Model - Componentes](03-Diseño-Tecnico/3.1-C4-Model/C3-Componentes.md) para la estructura interna
3. Estudia Fase 6 (Anexo Motor Dispersión) para patrones GoF aplicados

### Para DevOps/SRE
1. Comienza con [4.1 Arquitectura Cloud](04-Infraestructura-Resiliencia/4.1-Arquitectura-Cloud.md)
2. Revisa [4.2 Patrones de Resiliencia](04-Infraestructura-Resiliencia/4.2-Patrones-Resiliencia.md)
3. Consulta [3.2 UML - Infraestructura](03-Diseño-Tecnico/3.2-UML/Infraestructura.md) para topología de red

---

## 📊 Métricas del Proyecto

### Capacidad
- **Throughput**: 2,000 TPS → **1,000,000 TPS** (500x)
- **Latencia P99**: 3,000 ms → **100 ms** (30x mejora)
- **Carga Diaria**: 500,000 transacciones procesadas

### Disponibilidad
- **Uptime**: 99.5% → **99.999%** (43.8h → 5.26 min downtime/año)
- **RTO**: 6 horas → **15 minutos**
- **RPO**: 24 horas → **1 minuto**

### Agilidad
- **Time-to-Market**: 4 meses → **2 semanas** (8x más rápido)
- **Deployment Frequency**: Trimestral → **Diario** (múltiples deploys/día)
- **Lead Time**: 16 semanas → **3 días**
- **MTTR**: 8 horas → **30 minutos**

### Eficiencia
- **Reducción Costos Operacionales**: 35% (gracias a auto-scaling y rightsizing)
- **Optimización Memoria**: 5.2 GB → **1.85 GB** en Motor de Dispersión (64% reducción)
- **Utilización CPU**: 70% promedio → **45%** (mayor headroom para picos)

---

## 🏆 Principios de Diseño

1. **Domain-First**: El dominio de negocio guía las decisiones técnicas
2. **Evolutionary Architecture**: Fitness functions y decisiones reversibles
3. **You Build It, You Run It**: Ownership completo por equipo
4. **API First**: Contratos antes de implementación
5. **Security by Design**: Shift-left de seguridad (threat modeling temprano)
6. **Observability First**: Tracing, metrics, logs desde día 1
7. **Chaos Engineering**: Fallos inyectados para validar resiliencia
8. **Data Governance**: Ownership claro, encryption at rest/transit, retention policies

---

## 📝 Convenciones de Documentación

- **Diagramas**: Mermaid integrado en Markdown (renderizable en GitHub/GitLab)
- **Decisiones**: Justificación explícita con pros/cons y alternativas consideradas
- **Métricas**: Calculables desde requirements del kata (no ficticias)
- **Terminología**: DDD Ubiquitous Language (Payment Execution Context, no Payment Service)
- **Coherencia**: Documentos autocontenidos sin dependencias circulares explícitas

---

## 🔗 Referencias Externas

### Metodología
- [Why Driven Design (WhyDD)](https://github.com/DomainDrivenDesign/WhyDD)
- [C4 Model](https://c4model.com/)
- [Team Topologies](https://teamtopologies.com/)

### Patrones
- [Microservices Patterns - Chris Richardson](https://microservices.io/patterns/)
- [Cloud Design Patterns - Microsoft](https://docs.microsoft.com/en-us/azure/architecture/patterns/)
- [Gang of Four Design Patterns](https://refactoring.guru/design-patterns)

### AWS
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [MSK Best Practices](https://docs.aws.amazon.com/msk/latest/developerguide/bestpractices.html)

---

## ✅ Estado del Proyecto

| Fase | Estado | Completitud |
|------|--------|-------------|
| **Fase 1**: Entendimiento Negocio | ✅ Completado | 3/3 documentos |
| **Fase 2**: Diseño Estratégico (DDD) | ✅ Completado | 4/4 documentos |
| **Fase 3**: Diseño Técnico | ✅ Completado | 7/7 documentos |
| **Fase 4**: Infraestructura & Resiliencia | ✅ Completado | 3/3 documentos |
| **Fase 5**: Gobierno & Liderazgo | ✅ Completado | 4/4 documentos |
| **Fase 6**: Anexo Motor Dispersión | ✅ Completado | 7/7 documentos |

**Total**: 31 documentos | **Palabras**: ~85,000 | **Diagramas**: 45+ Mermaid diagrams

---

## 📄 Licencia

Este proyecto es una solución de kata arquitectónico con propósitos educativos y de demostración de capacidades de diseño de arquitectura de software empresarial.

---

**Última Actualización**: Diciembre 2025  
**Versión**: 1.0  
**Autor**: Diseño arquitectónico colaborativo con asistencia de IA
