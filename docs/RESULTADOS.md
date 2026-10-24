# Resultados — Azure SRE Agent vs Azure Arc-Connected SQL VM

> Lab ejecutado el 24 de octubre de 2026. Ambiente real desplegado y probado (no solo teoría). Ver `docs/PLAN.md` para el plan original y la arquitectura.

## 🎯 Resumen ejecutivo

**Sí — Azure SRE Agent puede analizar, diagnosticar y ejecutar acciones reales sobre una máquina conectada vía Azure Arc, incluyendo comandos directos sobre el sistema operativo remoto (no solo sobre el plano de control de Azure).** Esto se confirmó experimentalmente, no solo en teoría.

| # | Prueba | Resultado |
|---|---|---|
| 1 | Lectura de heartbeat/salud vía Log Analytics | ✅ **PASS** |
| 2 | Métricas de performance (CPU/memoria) con gráfico | ✅ **PASS** |
| 3 | Recomendaciones de Microsoft Defender for Cloud | 🟡 **PARCIAL** (ver hallazgo abajo) |
| 4 | Investigación de alerta de Azure Monitor | ⬜ No ejecutado (por tiempo — ver recomendación) |
| 5 | **Ejecución de comando remoto sobre el SO (Run Command)** | ✅ **PASS CONFIRMADO — la pregunta clave** |

---

## Ambiente construido

- **VM:** `arc-sql-lab-linux` — Ubuntu 22.04 + SQL Server 2022 (Developer edition), simulando un servidor on-premises.
- **Arc:** conectada como `Microsoft.HybridCompute/machines/arc-sql-lab-linux`, status `Connected`.
- **Monitoreo:** Azure Monitor Agent (AMA) vía extensión de Arc, Log Analytics Workspace dedicado, Data Collection Rule (Perf + Syslog).
- **Seguridad:** Microsoft Defender for Servers Plan 2 habilitado a nivel de suscripción.
- **Agente SRE:** `sre-arc-sql-lab`, con managed identity con los roles RBAC: `Reader`, `Monitoring Reader`, `Log Analytics Reader` (sobre el RG/workspace) y **`Azure Connected Machine Resource Administrator`** (sobre la máquina Arc específica — el rol que habilita Run Command).

### Nota técnica importante: cómo se simuló "on-premises"

Azure Arc no permite instalarse directamente sobre una VM nativa de Azure (lo bloquea explícitamente: *"Azure Connected Machine Agent is designed for use outside Azure"*). Para simular un servidor on-premises real dentro de una VM de Azure (único método soportado oficialmente por Microsoft para pruebas, documentado en ["How to evaluate Azure Arc-enabled servers with an Azure virtual machine"](https://learn.microsoft.com/en-us/azure/azure-arc/servers/plan-evaluate-on-azure-virtual-machine)), se aplicó:
1. Deshabilitar el Azure Linux Guest Agent (`walinuxagent`).
2. **Bloquear el endpoint de Azure IMDS** (`169.254.169.254`) vía firewall (`ufw`).
3. Setear la variable de entorno `MSFT_ARC_TEST=true`.

Con esto, la VM "se ve a sí misma" como si estuviera fuera de Azure, y Arc la trata exactamente igual que un servidor físico real — mismo agente, mismas capacidades, mismas limitaciones.

---

## Detalle de resultados

### Test 1 y 2 — Observabilidad vía Log Analytics ✅
El agente consultó directamente las tablas `Heartbeat` y `Perf` del workspace, reportó la máquina como "Healthy" con su último heartbeat, y **generó un gráfico real** (línea, CPU% y Memoria% de la última hora) usando su herramienta de ejecución de Python. Tras un 403 inicial del conector MCP dedicado de Monitor (recurrente — ver nota abajo), usó su herramienta genérica de Azure CLI con éxito en cuanto el RBAC propagó (~3-5 min).

### Test 3 — Microsoft Defender for Cloud 🟡
Las recomendaciones de Defender **aún no existían** en el momento de la prueba (es normal — el ciclo inicial de evaluación de seguridad toma horas tras habilitar el plan, no es instantáneo). Lo valioso: **el agente no se quedó bloqueado por la ausencia de datos** — en vez de eso:
- Detectó que faltaba la extensión `MDE.Linux` (Microsoft Defender for Endpoint), la causa raíz de por qué no había recomendaciones.
- Generó sus propios hallazgos de seguridad analizando directamente las reglas de NSG (SSH abierto a cualquier origen).
- Propuso y — con aprobación — **ejecutó el comando exacto para desplegar la extensión faltante**, usando su propio managed identity.

Esto demuestra una capacidad más interesante que "leer recomendaciones de Defender": el agente puede razonar sobre postura de seguridad de forma proactiva y **cerrar el gap de instrumentación él mismo**.

### Test 5 — Run Command remoto sobre el sistema operativo ✅ (el hallazgo central)

**Esta era la pregunta clave de Jorge: ¿puede el agente ejecutar acciones reales sobre el SO de una máquina Arc, no solo sobre el plano de control de Azure?**

**Respuesta: Sí, confirmado.** El agente:
1. Construyó el comando `az connectedmachine run-command create` (y, tras un problema de tooling interno, el equivalente `az rest` directo al API de ARM).
2. **Pidió aprobación humana explícita en el chat** antes de ejecutar (el guardrail de seguridad funciona igual que en los otros labs).
3. Ejecutó un script real en la VM: `df -h`, `free -h`, `systemctl status mssql-server`.
4. Recibió el resultado real del sistema operativo y lo formateó en un reporte de salud estructurado con veredictos:
   - **Disco:** 56 GB libres (11% usado) → Healthy
   - **Memoria:** 5.9 GiB disponible (~76%) → Healthy
   - **SQL Server:** `active (running)`, 42 min de uptime, PID, tareas, memoria, CPU → Healthy
   - **"All three checks pass — the machine looks healthy at the OS level."**

**Nota sobre los 2 fallos intermedios:** el agente tuvo 2 intentos fallidos antes del éxito, causados por un bug de persistencia de la extensión de Azure CLI (`connectedmachine`) dentro de su propio sandbox de ejecución — **no** relacionado con permisos ni con Arc en sí. El agente diagnosticó esto correctamente por su cuenta ("The CLI extension needs to be installed first... the write-command tool isn't picking up the extension") y resolvió el problema cambiando de estrategia (usar `az rest` en vez de la extensión CLI dedicada) sin intervención humana. Esto es, en sí mismo, una demostración relevante de la capacidad de auto-corrección del agente.

**Requisito de RBAC confirmado:** el rol que habilita esto es **"Azure Connected Machine Resource Administrator"**, que otorga el permiso `Microsoft.HybridCompute/machines/runCommands/write`. Sin este rol específico sobre la máquina Arc (no basta con Contributor genérico sobre el RG), el agente no podría ejecutar comandos remotos.

---

## 🔮 Recomendaciones de pruebas adicionales

Con la capacidad base ya confirmada, estas son las extensiones naturales que valdría la pena probar en una siguiente iteración:

1. **Alerta real de Azure Monitor → investigación automática.** Configurar una alerta basada en log (ej. CPU > 80% por 5 min, o espacio en disco < 10%) sobre la máquina Arc, y validar que el agente la detecte y arranque una investigación sin que se le pida explícitamente — tal como ya hace con recursos nativos de Azure en los otros labs.

2. **Remediación real end-to-end con aprobación.** Provocar un problema real en la VM (ej. llenar el disco a propósito, o detener el servicio de SQL Server) y pedirle al agente que lo diagnostique **y lo resuelva** vía Run Command (ej. `systemctl start mssql-server`) — cerrando el círculo completo que ya se demostró que es técnicamente posible.

3. **Azure Arc-enabled SQL Server** (extensión `Microsoft.AzureArcData/SqlServerInstances`, distinta del Run Command genérico): da telemetría específica de SQL — versión, parches pendientes, vulnerability assessment, integración con Microsoft Defender for SQL. El agente podría correlacionar hallazgos específicos de SQL (no solo "el servicio está corriendo", sino "la versión tiene una CVE conocida" o "faltan índices").

4. **Azure Automation Update Management vía Arc / Azure Update Manager.** Validar si el agente puede consultar o disparar el estado de compliance de parches del sistema operativo — relevante para la pregunta de Jorge sobre VMs on-premises que necesitan gestión de parches.

5. **Escenario de Azure Files con alertas de espacio** (la segunda idea mencionada por el cliente en la sesión anterior). Aunque no es parte de este lab de Arc, se puede armar como un lab separado más simple, ya que usa métricas nativas de Azure Monitor (`FileCapacity`, `FileShareSize`) sin necesitar Arc.

6. **Repetir el contraste "Policy vs SRE Agent" pero a nivel de sistema operativo**, usando Azure Policy Guest Configuration (audita configuración del SO, ej. "SSH no debe permitir root login") vs la capacidad del agente de detectarlo y corregirlo vía Run Command — una extensión directa del Compliance Lab que ya existe para recursos de Azure.

---

## Limitaciones encontradas (a tener en cuenta)

- **Arc no permite operaciones concurrentes** sobre la misma máquina (extensiones y run-commands comparten una cola — "Only one instance of an extension may be in progress at a time"). Si el agente está desplegando una extensión y se le pide simultáneamente un Run Command, uno de los dos fallará. No es un problema en el flujo normal (secuencial) de una demo, pero vale la pena saberlo.
- **El conector MCP dedicado de Azure Monitor dio 403 inicialmente** — el agente usa su fallback de Azure CLI genérico sin problemas, pero esto sugiere que el conector dedicado podría tener un bug de propagación de permisos distinto al de RBAC estándar (a investigar si se vuelve a presentar).
- **Las recomendaciones de Defender tardan horas en poblarse** tras habilitar el plan — no es instantáneo, hay que planificar el tiempo de antelación si se quiere mostrar esto en una demo con cliente.
- **RBAC puede tardar unos minutos en propagar** tras asignar un rol nuevo — se vio consistentemente en este lab y en labs anteriores (3-5 min).
