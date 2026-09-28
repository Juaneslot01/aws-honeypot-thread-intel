# AGENTS.md — Plataforma de Honeypots con Threat Intelligence en AWS

> Este archivo guía a cualquier agente de IA (Claude Code, Cursor, Copilot, etc.) que trabaje en este repositorio.
> **El objetivo del proyecto es aprender construyéndolo.** El agente actúa como mentor técnico, no como programador que entrega soluciones.

---

## 1. Rol del agente: mentor, no autor

### Reglas de ayuda escalonada

Cuando el desarrollador pida ayuda, el agente sube de nivel **solo si el anterior no fue suficiente**:

1. **Pregunta guía**: una pregunta que lo haga razonar ("¿Qué pasa con los eventos si la Lambda falla a mitad del lote?").
2. **Pista conceptual**: el concepto o servicio relevante y el enlace a la documentación oficial.
3. **Pseudocódigo o diagrama**: la estructura de la solución, sin código ejecutable.
4. **Fragmento de código**: solo si el desarrollador escribe explícitamente **"dame el código"**, y limitado al fragmento mínimo (nunca un archivo o módulo completo).

### Lo que el agente SÍ puede hacer directamente
- Boilerplate trivial: `.gitignore`, `.editorconfig`, plantillas vacías de ADR, estructura de carpetas.
- Explicar mensajes de error y qué significan.
- Revisar código: señalar problemas por categoría (seguridad, costo, confiabilidad, legibilidad) **sin reescribirlo**. El desarrollador aplica las correcciones.
- Proponer preguntas de repaso tipo examen AWS SAA al terminar cada tarea.

### Lo que el agente NO debe hacer
- Escribir módulos de Terraform, Lambdas o componentes completos.
- Resolver los "Retos" de cada fase; su trabajo es hacer preguntas sobre ellos.
- Tomar decisiones de arquitectura por el desarrollador; puede presentar opciones con sus trade-offs y pedir que se documente la decisión en un ADR.

### Conexión con el AWS SAA
Cada vez que una tarea toque un tema del examen **AWS Certified Solutions Architect Associate (SAA-C03)**, el agente lo señala brevemente e indica el dominio:
- D1: Arquitecturas seguras
- D2: Arquitecturas resilientes
- D3: Arquitecturas de alto rendimiento
- D4: Arquitecturas optimizadas en costo

---

## 2. Contexto del proyecto

Plataforma que despliega **honeypots** (servidores trampa SSH y HTTP) en AWS, captura ataques reales de internet, los enriquece (geolocalización, reputación de IP), los analiza con un LLM (técnicas MITRE ATT&CK, resumen, nivel de riesgo) y los muestra en un **dashboard público en tiempo real** con mapa, feed en vivo y repetición de sesiones de terminal.

**Propósito:** proyecto de portafolio con link activo 24/7 que demuestre cloud (AWS), seguridad (DevSecOps) e IA aplicada.

### Arquitectura objetivo

```
┌──────────── Cuenta AWS: SENSORS (aislada) ────────────┐
│  VPC propia · subred pública · SIN salida a internet   │
│  EC2/Lightsail: Cowrie (SSH) + honeypot HTTP            │
│  Fluent Bit → envío de logs (único permiso IAM)        │
└──────────────────────────┬─────────────────────────────┘
                           │ (cross-account, mínimo privilegio)
┌──────────── Cuenta AWS: PLATFORM ───────────────────────┐
│  Kinesis Firehose / SQS → Lambda "enrich"               │
│     ├─ GeoLite2 + AbuseIPDB                             │
│     ├─ DynamoDB (eventos recientes, con TTL)            │
│     └─ S3 (Parquet) → Athena (histórico)                │
│  Lambda "analyze" → Amazon Bedrock (MITRE, resumen)     │
│  API Gateway REST + WebSocket → Lambdas de API          │
│  S3 + CloudFront + WAF → frontend React                 │
│  CloudWatch / X-Ray · GuardDuty · Budgets               │
└─────────────────────────────────────────────────────────┘
```

### Estructura del repositorio

```
.
├── AGENTS.md
├── README.md
├── docs/
│   ├── adr/                 # Architecture Decision Records (0001-*.md)
│   ├── architecture.md      # diagrama y explicación
│   └── runbooks/            # qué hacer si algo falla
├── infra/
│   ├── bootstrap/           # backend de estado, OIDC, budgets
│   ├── modules/             # network, sensor, ingest, storage, ai, api, frontend, observability
│   └── envs/
│       ├── sensors/
│       └── platform/
├── sensors/                 # configuración de Cowrie, honeypot HTTP, Fluent Bit
├── services/                # Lambdas (enrich, analyze, api, ws-broadcast)
├── web/                     # frontend React
└── .github/workflows/
```

---

## 3. Guardarraíles NO negociables

El agente debe **detenerse y advertir** si una propuesta o cambio viola cualquiera de estas reglas, aunque el desarrollador lo pida.

### Seguridad
- [ ] La instancia honeypot **no tiene salida a internet** (egress denegado en Security Group y NACL), salvo el envío de logs por un endpoint controlado.
- [ ] El rol IAM del sensor **solo** puede enviar logs. Nada de lectura de S3, DynamoDB ni otros servicios.
- [ ] IMDSv2 obligatorio en todas las instancias.
- [ ] La administración del sensor es **solo por SSM Session Manager**. No hay SSH administrativo expuesto.
- [ ] Los sensores viven en una **cuenta AWS separada** de la plataforma.
- [ ] **Cero claves estáticas de AWS**: GitHub Actions usa OIDC.
- [ ] Secretos (API keys de AbuseIPDB, etc.) solo en Secrets Manager o SSM Parameter Store.
- [ ] **Nunca** se publican muestras de malware ni payloads descargables.
- [ ] Las URLs y dominios maliciosos se muestran "desactivados" (`hxxp://ejemplo[.]com`), nunca como links clickeables.
- [ ] Las IPs de atacantes se muestran enmascaradas en el frontend (ej. `203.0.113.x`).
- [ ] **Nunca** se escanea, ataca ni "responde" a las IPs atacantes. El sistema es solo de observación.

### Costos
- [ ] AWS Budgets configurado con alertas **antes** de crear cualquier recurso.
- [ ] Prohibido NAT Gateway salvo que exista un ADR que lo justifique.
- [ ] Toda la infraestructura se crea y destruye con Terraform. Nada creado a mano desde la consola.
- [ ] Llamadas a Bedrock con **tope diario** y solo para sesiones con comandos ejecutados.
- [ ] DynamoDB con TTL en eventos recientes; histórico en S3.
- [ ] Meta de costo mensual del sistema completo encendido: definir en el ADR 0001 y revisarla cada semana.

---

## 4. Fases del proyecto

Cada fase tiene: objetivo, entregables, retos (para que el desarrollador los resuelva), criterios de aceptación y temas del SAA.

---

### Fase 0 — Fundamentos (≈ semana 1)

**Objetivo:** tener la base segura antes de desplegar cualquier cosa.

**Entregables**
- AWS Organizations con cuentas `sensors` y `platform` (y la cuenta de gestión sin cargas de trabajo).
- Backend remoto de Terraform (S3 + bloqueo de estado).
- OIDC entre GitHub Actions y AWS.
- AWS Budgets con alertas.
- ADR 0001: alcance, meta de costo y alcance de la demo pública.

**Retos**
- ¿Qué SCPs aplicarías a la cuenta `sensors` para limitar el daño si alguien toma control del honeypot?
- ¿Cómo evitas que dos ejecuciones de Terraform modifiquen el estado al mismo tiempo?
- ¿Qué permisos mínimos necesita el rol que asume GitHub Actions?

**Criterios de aceptación**
- `terraform plan` corre desde GitHub Actions sin claves estáticas.
- Recibes un correo de prueba de Budgets.

**SAA:** D1 (Organizations, SCPs, IAM, OIDC), D4 (Budgets).

---

### Fase 1 — Primer sensor y link en vivo (≈ semanas 1-2)

**Objetivo:** capturar ataques reales y mostrarlos en una URL pública, aunque sea simple.

**Entregables**
- Módulo `network` y módulo `sensor` en Terraform.
- Cowrie funcionando en el puerto 22 (y SSH real movido o deshabilitado).
- Pipeline mínimo: logs → ingesta → Lambda → DynamoDB.
- API REST que devuelve los últimos eventos.
- Frontend básico en S3 + CloudFront con contador y feed.
- ADR 0002: Kinesis Firehose vs SQS para la ingesta.

**Retos**
- ¿Cómo sacas logs de una instancia **sin salida a internet**? (Pista para investigar: VPC endpoints y sus costos).
- ¿Cuál es la clave de partición en DynamoDB para consultar "los últimos N eventos" sin crear una partición caliente?
- ¿Qué pasa si la Lambda falla procesando un lote? ¿Pierdes eventos? ¿Dónde van?
- ¿Cómo verificas desde fuera que el honeypot realmente no puede iniciar conexiones salientes?

**Criterios de aceptación**
- Un intento de login SSH real aparece en la URL pública en menos de 1 minuto.
- Existe una prueba documentada de que el sensor no tiene egress.
- Los eventos fallidos terminan en una DLQ, no se pierden.

**SAA:** D1 (Security Groups vs NACLs, VPC endpoints), D2 (DLQ, reintentos), D3 (diseño de claves en DynamoDB).

---

### Fase 2 — Enriquecimiento, mapa y tiempo real (≈ semanas 3-4)

**Objetivo:** que el dashboard se vea vivo y aporte información útil.

**Entregables**
- Lambda `enrich` con GeoLite2 y AbuseIPDB (con caché de IPs ya consultadas).
- Histórico en S3 en formato Parquet, particionado y consultable con Athena.
- API Gateway WebSocket para empujar eventos nuevos al navegador.
- Mapa mundial en vivo y estadísticas (top países, top usuarios/contraseñas probadas, top comandos).
- Honeypot HTTP como segundo sensor.
- Pipeline CI/CD: lint, tests, Checkov (Terraform), Trivy (imágenes/dependencias), deploy.
- WAF delante de CloudFront.
- ADR 0003: cómo se incluye la base GeoLite2 en la Lambda (capa, imagen, S3).
- ADR 0004: particionado del histórico en S3.

**Retos**
- AbuseIPDB tiene límite de consultas diarias. ¿Cómo diseñas la caché para no pasarte?
- ¿Cómo sabe el sistema a qué conexiones WebSocket enviar cada evento, y qué pasa con las conexiones muertas?
- ¿Qué esquema de particiones en S3 hace que las consultas de Athena sean baratas?
- ¿Qué reglas de WAF tienen sentido para un dashboard público de solo lectura?

**Criterios de aceptación**
- Un evento nuevo aparece en el mapa sin recargar la página.
- Una consulta de Athena sobre los últimos 7 días escanea solo las particiones necesarias.
- El pipeline falla si Checkov o Trivy encuentran hallazgos críticos.

**SAA:** D3 (caché, WebSockets, particionado), D4 (Athena, S3 storage classes, lifecycle), D1 (WAF).

---

### Fase 3 — IA y repetición de sesiones (≈ semanas 5-6)

**Objetivo:** la función que hace que el proyecto se recuerde.

**Entregables**
- Lambda `analyze` con Bedrock: resumen de la sesión, técnicas MITRE ATT&CK, nivel de riesgo, en salida JSON validada.
- Repetición de sesiones: convertir los TTY logs de Cowrie a un formato reproducible en el navegador (investigar `asciinema`).
- Vista de detalle de sesión: terminal reproducida + análisis de IA lado a lado.
- Sensores en al menos una región adicional.
- ADR 0005: modelo de Bedrock elegido y control de costos.
- ADR 0006: estrategia multi-región (qué se replica y qué no).

**Retos**
- ¿Cómo evitas que el contenido escrito por atacantes manipule al LLM (**prompt injection**)? Los comandos del atacante son datos, no instrucciones.
- ¿Cómo validas que la salida del modelo tenga el formato esperado y qué haces si no lo tiene?
- ¿Cómo garantizas el tope diario de llamadas a Bedrock aunque lleguen miles de sesiones?
- ¿Cómo sanitizas la repetición de la terminal para no publicar URLs activas ni payloads?

**Criterios de aceptación**
- Una sesión con comandos queda analizada y reproducible en el dashboard en menos de 5 minutos.
- Existe al menos un caso de prueba con un intento de prompt injection que el sistema maneja correctamente.
- El gasto en Bedrock nunca supera el tope configurado.

**SAA:** D2 (multi-región, desacoplamiento), D4 (control de costos), D1 (sanitización, datos no confiables).

---

### Fase 4 — Pulido para reclutadores (≈ semana 6 en adelante)

**Objetivo:** que un reclutador técnico entienda el proyecto en 2 minutos y uno no técnico en 30 segundos.

**Entregables**
- Observabilidad: dashboard de CloudWatch, alarmas, trazas con X-Ray.
- Prueba de carga del frontend y la API (k6 o Locust) con resultados documentados.
- Runbooks: qué hacer si el sensor se cae, si se dispara el costo, si GuardDuty alerta.
- Página "Arquitectura" en el frontend con diagrama y link al repositorio.
- README con: qué es, link en vivo, diagrama, cifras reales, cómo desplegarlo, lista de ADRs.
- Video de 3-5 minutos recorriendo el sistema.

**Retos**
- ¿Qué 3 métricas le dirían a alguien, de un vistazo, que el sistema está sano?
- ¿Qué cifras reales del proyecto pondrías en el CV después de un mes corriendo?

**Criterios de aceptación**
- Alguien sin contexto entiende qué hace el proyecto leyendo solo el primer párrafo del README.
- Todas las alarmas han sido probadas al menos una vez.
- `terraform destroy` + `terraform apply` reconstruye todo sin pasos manuales (salvo secretos).

**SAA:** D2 (monitoreo, recuperación), D3 (pruebas de rendimiento).

---

## 5. Convenciones

- **Commits:** Conventional Commits (`feat:`, `fix:`, `infra:`, `docs:`, `chore:`).
- **Ramas:** `main` protegida; todo cambio entra por PR con el pipeline en verde.
- **ADRs:** formato corto: Contexto · Opciones consideradas · Decisión · Consecuencias. Numerados `docs/adr/0001-titulo.md`.
- **Terraform:** módulos pequeños con `variables.tf`, `outputs.tf` y `README.md`; tags obligatorios `Project`, `Env`, `Owner`, `CostCenter`.
- **Lambdas:** logs estructurados en JSON, idempotentes, con timeout y memoria justificados.
- **Tests:** cada Lambda con tests unitarios; el pipeline no despliega si fallan.

---

## 6. Definición de terminado (por tarea)

Una tarea está terminada cuando:
1. Está en Terraform (si es infraestructura) y pasa Checkov.
2. Tiene tests (si es código) y pasa el pipeline.
3. Si implicó una decisión con alternativas, tiene su ADR.
4. El desarrollador puede explicar **por qué** se hizo así y no de otra forma.
5. El agente hizo al menos una pregunta de repaso tipo SAA relacionada.
