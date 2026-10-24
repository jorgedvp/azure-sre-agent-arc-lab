# Plan — Azure SRE Agent vs Azure Arc-Connected SQL VM

## Objetivo

Validar experimentalmente (no solo en teoría) hasta dónde llegan las capacidades de **Azure SRE Agent** cuando el recurso objetivo es una VM **conectada vía Azure Arc** (simulando un servidor on-premises con SQL Server), en vez de un recurso nativo de Azure.

Preguntas que este lab busca responder:
1. ¿Puede el agente **ver y analizar** una máquina Arc a través de Azure Monitor / Log Analytics (heartbeat, performance counters, logs)?
2. ¿Puede el agente **leer recomendaciones de Microsoft Defender for Cloud** sobre esa máquina?
3. ¿Puede el agente **reaccionar a una alerta** de Azure Monitor configurada sobre métricas de la VM Arc?
4. ¿Puede el agente **ejecutar diagnósticos** (consultas, scripts de solo lectura) sobre la máquina Arc?
5. **La pregunta clave:** ¿puede el agente **ejecutar una acción remota real sobre el sistema operativo** de la VM Arc (no solo sobre el plano de control de Azure), usando la funcionalidad "Run Command" del Connected Machine Agent?

## Hipótesis (basada en investigación previa a la ejecución)

- Azure SRE Agent no tiene un conector "nativo" dedicado a Azure Arc — pero sus herramientas `RunAzCliReadCommands` / `RunAzCliWriteCommands` ejecutan **Azure CLI genérico**. Esto significa que, en teoría, el agente puede invocar cualquier comando `az` para el cual su identidad administrada tenga permisos RBAC — incluyendo la extensión `connectedmachine` de Azure CLI.
- **Run Command en Azure Arc-enabled servers** (preview) está incorporado en el Connected Machine Agent (v1.33+), permite ejecutar scripts remotos (PowerShell en Windows, bash en Linux) **sin necesitar RDP/SSH**, corriendo como **Local System / root**. No requiere instalar ninguna extensión adicional.
- El permiso necesario es el rol RBAC **"Azure Connected Machine Resource Administrator"** (otorga `Microsoft.HybridCompute/machines/runCommands/write`) sobre la máquina Arc o el resource group.
- **Conclusión de la hipótesis:** si le damos al managed identity del SRE Agent ese rol RBAC sobre la máquina Arc, el agente debería poder ejecutar `az connectedmachine run-command create` para correr un script real en el sistema operativo remoto — convirtiendo esto en una capacidad de **remediación real sobre infraestructura on-premises simulada**, no solo sobre recursos nativos de Azure.

## Arquitectura del lab

Como no hay un servidor físico on-premises disponible, se usa el patrón estándar (el mismo que usa el propio repo oficial `Azure/arc_jumpstart_docs` de Microsoft para demos): desplegar una VM nativa de Azure, y luego conectarla a Azure Arc usando el script `azcmagent connect`, lo cual la registra como un recurso `Microsoft.HybridCompute/machines` — indistinguible, desde el punto de vista de Arc/Azure Monitor/SRE Agent, de un servidor físico real on-premises.

```
┌─────────────────────────────────────────────────────────────┐
│  rg-arc-sql-lab (Azure, mismo tenant/sub que los labs previos)│
│                                                               │
│  ┌──────────────┐     azcmagent connect      ┌─────────────┐│
│  │ VM Azure      │ ─────────────────────────> │ Arc Machine ││
│  │ Windows+SQL   │   (se registra como si     │ (Hybrid     ││
│  │ (simula       │    fuera on-prem)          │  Compute)   ││
│  │  on-premises) │                            └──────┬──────┘│
│  └──────────────┘                                     │       │
│         │ AMA extension (via Arc)                     │       │
│         ▼                                             │       │
│  ┌──────────────────┐        ┌─────────────────┐      │       │
│  │ Log Analytics WS  │<───── │ Defender for      │<────┘       │
│  │ (Heartbeat, Perf,  │       │ Servers Plan 2    │             │
│  │  SecurityEvent)    │       │ (recomendaciones) │             │
│  └─────────┬──────────┘      └──────────────────┘             │
│            │                                                   │
│            ▼                                                   │
│  ┌──────────────────────┐                                     │
│  │ SRE Agent (nuevo)      │──RunAzCliReadCommands──> Log       │
│  │ sre-arc-sql-lab        │                           Analytics│
│  │ RBAC: Reader +         │──RunAzCliWriteCommands──> az       │
│  │ Azure Connected        │   (run-command create)   connected │
│  │ Machine Resource Admin │                           machine  │
│  └──────────────────────┘                                     │
└─────────────────────────────────────────────────────────────┘
```

## Pasos de ejecución

1. **Provisionar la VM** (Windows Server 2022 + SQL Server, imagen de marketplace o instalación manual de SQL Express) en una nueva resource group `rg-arc-sql-lab`.
2. **Crear Service Principal de onboarding** con el rol "Azure Connected Machine Onboarding" para registrar la máquina en Arc sin usar credenciales interactivas.
3. **Conectar la VM a Arc** ejecutando `azcmagent connect` dentro de la VM (vía `az vm run-command invoke`, ya que en este punto todavía es una VM nativa de Azure con acceso de administrador).
4. **Instalar Azure Monitor Agent (AMA)** como extensión de Arc + crear una Data Collection Rule (DCR) apuntando a un Log Analytics Workspace nuevo, recolectando Heartbeat, Performance counters, y eventos de Windows relevantes para SQL Server.
5. **Habilitar Microsoft Defender for Servers Plan 2** a nivel de suscripción (o scoped al RG si es posible) para que genere recomendaciones de seguridad sobre la máquina Arc.
6. **Crear el SRE Agent nuevo** (`sre-arc-sql-lab`) en `rg-arc-sql-lab`, conectar el conector de Azure Monitor apuntando al Log Analytics Workspace, y asignar al managed identity del agente:
   - `Reader` sobre el resource group (para leer Defender assessments, Log Analytics, estado de la máquina Arc).
   - `Azure Connected Machine Resource Administrator` sobre la máquina Arc específicamente (para permitir Run Command).
7. **Ejecutar la matriz de pruebas** (ver abajo) vía chat con el agente, documentando cada resultado (éxito/falla/limitación) con capturas de pantalla.
8. **Documentar resultados y recomendaciones** en `docs/RESULTADOS.md`.

## Matriz de pruebas

| # | Prueba | Cómo se valida | Resultado esperado si funciona |
|---|---|---|---|
| 1 | Lectura de heartbeat/salud de la máquina Arc | Preguntarle al agente "¿está sana la VM `arc-sql-lab-vm`? Dame el último heartbeat" | Agente consulta tabla `Heartbeat` en Log Analytics y responde con el timestamp/estado |
| 2 | Lectura de métricas de performance (CPU/memoria/disco) | Preguntarle "dame el uso de CPU y memoria de las últimas 2 horas" | Agente consulta tabla `Perf` y resume tendencias |
| 3 | Lectura de recomendaciones de Defender for Cloud | Preguntarle "qué recomendaciones de seguridad tiene Defender sobre esta máquina" | Agente ejecuta `az security assessment list` (u otra vía) y resume hallazgos |
| 4 | Investigación de una alerta de Azure Monitor sobre la VM Arc | Configurar una alerta (ej. CPU > umbral) y simularla / o pedirle que la investigue directamente | Agente correlaciona la alerta con métricas y da un diagnóstico |
| 5 | **Ejecución remota de comando (Run Command) sobre el SO** | Pedirle "revisa el estado del servicio de SQL Server en la VM y si está detenido, reinícialo" / o una tarea más simple como "chequea el espacio en disco libre ejecutando un comando en la VM" | Agente arma y ejecuta `az connectedmachine run-command create` (pidiendo aprobación), y el resultado del script se refleja en la VM real |

## Recomendaciones de pruebas adicionales (para una siguiente iteración, no ejecutadas en este lab)

- **Arc-enabled SQL Server** (extensión `Microsoft.AzureArcData/SqlServerInstances`): da telemetría específica de SQL (versión, parches, vulnerability assessment, Defender for SQL) — el agente podría correlacionar hallazgos de SQL con el resto del diagnóstico.
- **Azure Automation Update Management vía Arc**: validar si el agente puede disparar o consultar compliance de parches del sistema operativo.
- **Change Tracking & Inventory vía Arc**: detectar drift de configuración (archivos, servicios, software instalado) y ver si el agente puede correlacionar un cambio reciente con un incidente.
- **Azure Policy Guest Configuration sobre Arc machines**: réplica del contraste "Policy vs SRE Agent" ya hecho en el Compliance Lab, pero a nivel de configuración de sistema operativo en vez de recursos de Azure.
- **Escenario multi-step de remediación con aprobación**: Defender encuentra una vulnerabilidad → agente la explica → agente propone un Run Command de remediación → usuario aprueba → agente ejecuta y verifica.
