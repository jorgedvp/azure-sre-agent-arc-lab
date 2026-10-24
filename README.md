# Azure SRE Agent — Arc-Connected SQL VM Lab

Lab exploratorio para validar hasta dónde llegan las capacidades de **Azure SRE Agent** cuando el recurso objetivo es una máquina conectada vía **Azure Arc** (simulando un servidor on-premises con SQL Server), en vez de un recurso nativo de Azure.

## 📄 Documentos

- **[docs/PLAN.md](docs/PLAN.md)** — Objetivo, hipótesis, arquitectura y matriz de pruebas planificada.
- **[docs/RESULTADOS.md](docs/RESULTADOS.md)** — Resultados reales, con capturas de pantalla, y recomendaciones de siguientes pasos.

## 🎯 Resultado principal

**Confirmado: Azure SRE Agent puede ejecutar comandos reales sobre el sistema operativo de una máquina Arc** (no solo leer métricas/logs del plano de control de Azure), usando la funcionalidad "Run Command" de Azure Arc y su propio managed identity con el rol RBAC "Azure Connected Machine Resource Administrator". Ver `docs/RESULTADOS.md` para el detalle completo y las capturas de pantalla.

## Recursos del lab

- Resource group: `rg-arc-sql-lab` (tenant `e00660b6-957d-4851-a2b1-26d94d7527cd`, suscripción `dde51996-0311-4cae-9195-5f943bd41bc5`)
- VM: `arc-sql-lab-linux` (Ubuntu 22.04 + SQL Server 2022, conectada a Arc simulando on-premises)
- Agente SRE: `sre-arc-sql-lab`
