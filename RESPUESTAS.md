# Respuestas · Laboratorio 4
Alumno: [20206795]
Presupuesto: [13] min  Métrica DORA: [change failure rate]

## Parte 1 · Diagnóstico
### Defecto 1
- **Job y etapa:** `build-push`, etapa **build**.
- **Problema que causa en el pipeline:** El escaneo de la imagen con Trivy no incluye la bandera `--exit-code 1` para fallar ante vulnerabilidades críticas o altas. Esto permite que una imagen con vulnerabilidades conocidas en el sistema operativo base o runtime se publique en ECR y se despliegue en AWS Lambda sin detener el pipeline.
- **Link al commit:** https://github.com/Windar25/INF384-lab4-dev/commit/5eab3f9acdc5e086f83aa3b2d331037362075636
- **Corrección:** Ajustar configuración.
- **Evidencia:** En el resumen del run #1 de 14m 6s, Trivy detectó 211 hallazgos de severidad HIGH y CRITICAL pero finalizó como informativo sin fallar el job:
  > `Escaneo de la imagen: 211 hallazgos HIGH o CRITICAL. Informativo, no detiene.`
  > Run de referencia: https://github.com/Windar25/INF384-lab4-dev/actions/runs/40956186279

### Defecto 2
- **Job y etapa:** `load`, etapa **post-deploy**.
- **Problema que causa en el pipeline:** El job `load` está configurado para depender únicamente de `smoke` (`needs: smoke`), ejecutándose en paralelo con el job `e2e` (Playwright). Los 30 usuarios virtuales de k6 sobrecargan la función Lambda en ejecución, introduciendo latencia que provoca fallos espurios/falsos positivos en las pruebas E2E y desperdiciando minutos de ejecución.
- **Link al commit:** https://github.com/Windar25/INF384-lab4-dev/commit/003a849e70fb13f0ed1bf0b0b31c00ec20fa0dbc
- **Corrección:** Mover de etapa / Ajustar configuración (`needs: [smoke, e2e]`).
- **Evidencia:** En el gráfico de ejecución DAG del run #1, `e2e` (32s) y `load` (12m 10s) se inician simultáneamente tras la finalización de `smoke`:
  > `needs: [smoke]` en el job `load` de `.github/workflows/ci.yml`.

### Defecto 3
- **Job y etapa:** Nivel global del workflow (`ci.yml`) / `deploy`, etapa **deploy**.
- **Problema que causa en el pipeline:** Ausencia del bloque `concurrency` en la definición del workflow. Dado que los Pull Requests y los pushes a `main` se despliegan sobre la misma función de AWS Lambda, dos ejecuciones simultáneas se sobreescriben entre sí a mitad del proceso, generando condiciones de carrera e inconsistencias durante las pruebas de integración o E2E.
- **Link al commit:** https://github.com/Windar25/INF384-lab4-dev/commit/1637bdbc663603e1e78a597c125052c30b8a64c2
- **Corrección:** Ajustar configuración.
- **Evidencia:** Inspección directa del encabezado del archivo `.github/workflows/ci.yml`, el cual carece de la clave `concurrency:` para controlar ejecuciones concurrentes.

### Defecto 4
- **Job y etapa:** Nivel global del workflow (`ci.yml`), etapa **pre-build** / Activación.
- **Problema que causa en el pipeline:** El disparador del evento no utiliza `paths-ignore`. Modificaciones en archivos de documentación o Markdown (`.md`) desencadenan la ejecución completa del pipeline (construcción de imágenes, despliegues a AWS y pruebas de carga), consumiendo de forma innecesaria el presupuesto de minutos del runner.
- **Link al commit:** https://github.com/Windar25/INF384-lab4-dev/commit/40b8550e0311ef52036d338f36417fd5fd2738a7
- **Corrección:** Ajustar configuración.
- **Evidencia:** La directiva `on:` dentro de `.github/workflows/ci.yml` escucha `push` y `pull_request` sin el filtro `paths-ignore`:
  > `on: push: branches: [main] pull_request:` (sin omitir archivos `.md`).

### Defecto 5
- **Job y etapa:** `smoke`, etapa **post-deploy**.
- **Problema que causa en el pipeline:** El script `test/smoke/smoke.sh` contiene una validación de versión condicionada a la variable `EXPECTED_COMMIT`. Al no ser inyectada desde el job `smoke` en `ci.yml`, el pipeline omitía la verificación que garantiza que la función AWS Lambda tenga desplegado exactamente el commit en ejecución, pudiendo aprobar ejecuciones con artefactos desactualizados.
- **Link al commit:** https://github.com/Windar25/INF384-lab4-dev/commit/09d378a26dd3c422ce44062cac2fd7f846d0d114
- **Corrección:** Ajustar configuración.
- **Evidencia:** Inspección de `test/smoke/smoke.sh` (bloque `if [ -n "${EXPECTED_COMMIT:-}" ]`) y la ausencia de la clave `EXPECTED_COMMIT:` dentro de `env:` en el job `smoke` de `.github/workflows/ci.yml`.


## Parte 2 · Implementación
- Link al run en verde en main: https://github.com/Windar25/INF384-lab4-dev/actions/runs/40965328531
- Link al pull request: https://github.com/Windar25/INF384-lab4-dev/pull/1
- Defecto de la aplicación introducido (archivo y descripción): `src/wallet.js`. Se introdujo una excepción explícita (`throw new Error`) en la función `transferir` para simular un fallo en la lógica de negocio de las transferencias.
- Job que debería detenerlo: `unit`
- Link al run fallido: https://github.com/Windar25/INF384-lab4-dev/actions/runs/40963429359/job/123761756367?pr=1

## Parte 3 · Decisión bajo restricción
Link al run medido: https://github.com/Windar25/INF384-lab4-dev/actions/runs/40965328531

| Job | Duración (min:seg) |
|---|---|
| unit | 00:12 |
| integration | 00:12 |
| sast | 00:17 |
| sca | 00:22 |
| build-push | 01:55 |
| deploy | 00:21 |
| smoke | 00:12 |
| e2e | 00:32 |
| load | 12:13 |
| **Total** | **16:16** |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | 67.66 ms |
| p95 | 69.54 ms |
| Máximo | 329.73 ms |
| Tasa de error | 0.00% |
| Requests por segundo | 48.80 req/s |

Conclusión sobre el comportamiento bajo carga:
La aplicación Monedero Andino responde con alta estabilidad bajo la carga evaluada por k6, manteniendo disponibilidad continua y una tasa de error de 0.00% sin caídas de peticiones HTTP, con una latencia p95 de 69.54 ms.

SLO propuesto (percentil, valor, ventana) y justificación:
Percentil 95 (p95) < 300 ms evaluado en una ventana de 30 días. Se configuró `SLO_P95: 300` en `ci.yml`. Se establece este valor holgado por encima de los 69.54 ms medidos para absorber variaciones normales de red y rendimiento en los runners de GitHub Actions sin hacer fallar el job `load` por falsos positivos.

SLA propuesto y diferencia con el SLO:
Percentil 95 (p95) < 500 ms con 99.5% de disponibilidad mensual. El SLO es un objetivo de ingeniería interno para detectar degradaciones técnicas de forma temprana; el SLA es el acuerdo contractual con clientes externos que incluye penalizaciones económicas, por lo que posee un margen de tolerancia superior para proteger a la organización.

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|
| Ajustar el stage de carga constante en test/load/transferir.js a 4 minutos | load | 8 min |
| Configurar escaneo de imágenes de Trivy como informativo (`continue-on-error: true`) | build-push | 0.5 min |

Duración total resultante: 9 min 01 s
Link al run en main con los cambios: https://github.com/Windar25/INF384-lab4-dev/actions/runs/40968175684

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):
Al acortar la duración de las pruebas de carga o paralelizar análisis de seguridad, degradaciones puntuales bajo tráfico sostenido o casos de borde podrían no ser detectadas antes del despliegue.
**Efecto en Change Failure Rate (CFR):** Sube. Si un defecto no detectado en CI llega a producción y falla en ejecución, incrementará la tasa de cambios que generan fallos.

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón:
Se mueve a informativo (`continue-on-error: true`) o a un evento asíncrono. Los análisis estáticos consumen tiempo de ejecución en el pipeline sin validar funcionalidad en tiempo de ejecución; no deben bloquear entregas críticas de valor.
**Efecto en Change Failure Rate (CFR):** Sube. Al no ser bloqueante ante hallazgos estáticos, existe el riesgo de desplegar vulnerabilidades no corregidas a producción.

## Declaración de uso de IA
Herramientas usadas y para qué:
- **Gemini:** Se utilizó como herramienta de apoyo y guía técnica para comprender los requerimientos del laboratorio. Ayudó a interpretar los fallos y la lógica de configuración en `ci.yml`, analizar los resultados de las pruebas de carga con k6 y orientar la estructuración de las respuestas teóricas y justificaciones en `RESPUESTAS.md`
