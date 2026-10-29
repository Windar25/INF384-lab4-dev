# Respuestas · Laboratorio 4
Alumno: [20206795]
Presupuesto: [8] min  Métrica DORA: [change failure rate]

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
- Link al run en verde en main: https://github.com/Windar25/INF384-lab4-dev/actions/runs/40956186279
- Link al pull request: https://github.com/Windar25/INF384-lab4-dev/pull/1
- Defecto de la aplicación introducido (archivo y descripción): `src/wallet.js`. Se introdujo una excepción explícita (`throw new Error`) en la función `transferir` para simular un fallo en la lógica de negocio de las transferencias.
- Job que debería detenerlo: `unit`
- Link al run fallido: https://github.com/Windar25/INF384-lab4-dev/actions/runs/40963429359/job/123761756367?pr=1


## Parte 3 · Decisión bajo restricción
Link al run medido:

| Job | Duración (min:seg) |
|---|---|
| unit | |
| integration | |
| sast | |
| sca | |
| build-push | |
| deploy | |
| smoke | |
| e2e | |
| load | |
| **Total** | |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | |
| p95 | |
| Máximo | |
| Tasa de error | |
| Requests por segundo | |

Conclusión sobre el comportamiento bajo carga:

SLO propuesto (percentil, valor, ventana) y justificación:

SLA propuesto y diferencia con el SLO:

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|

Total resultante: [ ] min
Link al run en main con los cambios:

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón:

## Declaración de uso de IA
Herramientas usadas y para qué:
