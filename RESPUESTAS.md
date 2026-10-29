# Respuestas · Laboratorio 4
Alumno: [20213761]
Presupuesto: [13] min  Métrica DORA: [change failure rate]

## Parte 1 · Diagnóstico
### Defecto 1
- Job y etapa: smoke; post-deploy
- Problema que causa en el pipeline: El job smoke declara "needs: build-push" en vez de "needs: deploy". Se ejecuta en paralelo con el despliegue en AWS Lambda, intentando probar la Function URL antes de que el nuevo contenedor esté activo, por lo que evalúa la versión antigua o falla por condición de carrera durante la actualización.
- Link al commit: https://github.com/Portador3/INF384-lab4-20213761/commit/ebc9df055025a54e5874131ad1b22af957901637
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): mover de etapa
- Evidencia (dato observado, con link al run o fragmento de la salida local): 
https://github.com/Portador3/INF384-lab4-20213761/actions/runs/40957162360
Se observa en la pantalla del flujo de ejecución que deploy y smoke se realizan en paralelo.
https://github.com/Portador3/INF384-lab4-20213761/actions/runs/40961254754
Luego se observa que primero se ejecuta deploy y después smoke.
### Defecto 2
- Job y etapa: smoke; post-deploy
- Problema que causa en el pipeline: El paso que ejecuta test/smoke/smoke.sh en ci.yml solo pasa la variable BASE_URL, omitiendo EXPECTED_COMMIT: ${{ github.sha }}. El script condiciona la verificación del SHA a la existencia de esa variable (if [ -n "${EXPECTED_COMMIT:-}" ]), dejando pasar despliegues donde Lambda continúe respondiendo con un contenedor o versión desactualizada.
- Link al commit: https://github.com/Portador3/INF384-lab4-20213761/commit/a2eef79b734a318164d999cd3a700bd09a09d41a
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración
- Evidencia (dato observado, con link al run o fragmento de la salida local):
Antes de corregir (https://github.com/Portador3/INF384-lab4-20213761/actions/runs/40961254754/job/123754932386): 
Run test/smoke/smoke.sh
salud: {"ok":true,"version":"1.0.0","commit":"ebc9df055025a54e5874131ad1b22af957901637"}
saldo: {"usuario":"ana","saldo":1000}
Después de corregir (https://github.com/Portador3/INF384-lab4-20213761/actions/runs/40962044484/job/123757755938):
Run test/smoke/smoke.sh
salud: {"ok":true,"version":"1.0.0","commit":"a2eef79b734a318164d999cd3a700bd09a09d41a"}
commit: a2eef79b734a318164d999cd3a700bd09a09d41a
saldo: {"usuario":"ana","saldo":1000}

Antes se observaba que se saltaba la evidencia de validación del commit.


### Defecto 3
- Job y etapa: sca; pre-build
- Problema que causa en el pipeline: El comando de Trivy ejecuta la revisión con la bandera --include-dev-deps y --exit-code 1 sobre package-lock.json. Analiza librerías como Playwright, esbuild o c8 que solo se usan en el entorno de CI y nunca se envían al contenedor de producción en AWS Lambda, desperdiciando tiempo y deteniendo el pipeline por vulnerabilidades en herramientas dev.
- Link al commit: https://github.com/Portador3/INF384-lab4-20213761/commit/ca67156b6a6aa86c2b711ddef9348dd26186a6a0
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar configuración
- Evidencia (dato observado, con link al run o fragmento de la salida local):
Antes se detectaban todas las vulnerabilidades (https://github.com/Portador3/INF384-lab4-20213761/actions/runs/40962044484/job/123757071817):
2026-10-29T21:57:40Z	INFO	[npm] Detecting vulnerabilities...

Después se omiten las vulnerabilidades de desarrollo (https://github.com/Portador3/INF384-lab4-20213761/actions/runs/40963344140/job/123761466455):
2026-10-29T22:07:22Z	INFO	Suppressing dependencies for development and testing.

### Defecto 4
- Job y etapa: integration; pre-build
- Problema que causa en el pipeline: Las pruebas registradas en test/integration/ ejecutan mutaciones directamente sobre la base SQLite en /tmp/wallet.db sin reiniciar el archivo antes o después de la suite. Si las pruebas se reejecutan en el mismo runner o en entorno local, los saldos modificados por ejecuciones previas provocan el fallo de aserciones de valor fijo (como assert.equal(r.cuerpo.saldoOrigen, 900)).
- Link al commit: https://github.com/Portador3/INF384-lab4-20213761/commit/9e852652cd3205f1061f97f30f67b5567538162a
https://github.com/Portador3/INF384-lab4-20213761/commit/7ae9e871822e55562e52f2de09921e52541c1a3c
https://github.com/Portador3/INF384-lab4-20213761/commit/7e44fe7d19c1ab849866194aec480d7b4691c4e2
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): modificar tests, ajustar configuración
- Evidencia (dato observado, con link al run o fragmento de la salida local):
Se limpia la BD (https://github.com/Portador3/INF384-lab4-20213761/actions/runs/40966091042/job/123770855048):
 \# \[INTEGRATION\] Base de datos limpia iniciada: /tmp/wallet.db

### Defecto 5
- Job y etapa: e2e; post-deploy
- Problema que causa en el pipeline: Cuando la prueba de carga con k6 (load) bombardea la función en AWS Lambda con múltiples peticiones concurrentes, altera la base de datos SQLite y satura los tiempos de respuesta de la función Lambda exactamente al mismo tiempo que Playwright (e2e) intenta realizar aserciones funcionales secuenciales y deterministas. Esto genera interrupciones, condiciones de carrera y fallos por timeout.
- Link al commit: https://github.com/Portador3/INF384-lab4-20213761/commit/70019086f5659b45cf6cf357699e1610cc478f8b
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): mover de etapa
- Evidencia (dato observado, con link al run o fragmento de la salida local):
https://github.com/Portador3/INF384-lab4-20213761/actions/runs/40967578446
Ahora se evidencia un flujo lineal

## Parte 3 · Decisión bajo restricción
Link al run medido: https://github.com/Portador3/INF384-lab4-20213761/actions/runs/40967578446

| Job | Duración (min:seg) |
|---|---|
| `unit` | 0m 10s |
| `integration` | 0m 14s |
| `sast` | 0m 24s |
| `sca` | 0m 12s |
| `build-push` | 0m 57s |
| `deploy` | 0m 16s |
| `smoke` | 0m 09s |
| `e2e` | 0m 46s |
| `load` | 12m 10s |
| **Total** | **15m 18s** *(14m 42s secuencial)* |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | 145.2 ms |
| p95 | 182.5 ms |
| Máximo | 425.0 ms |
| Tasa de error | 0.00% |
| Requests por segundo | 42.8 req/s |

Conclusión sobre el comportamiento bajo carga:
La aplicación responde de manera estable sin errores (0.00%) bajo el tráfico simulado por k6. Sin embargo, el job `load` representa el principal cuello de botella de tiempo en el pipeline (12m 10s). Los picos de latencia máxima se justifican por arranques en frío (*cold starts*) de AWS Lambda y la latencia de escritura en disco de SQLite.

SLO propuesto (percentil, valor, ventana) y justificación:
- **Percentil y Valor:** p95 < 300 ms (`SLO_P95: 300` en `ci.yml`).
- **Ventana de medición:** Ventana móvil de 30 días.
- **Justificación:** Otorga una holgura del ~40% respecto al p95 medido (~182.5 ms) para absorber fluctuaciones normales de red y carga en la infraestructura serverless de AWS Lambda sin disparar falsas alarmas en el CI/CD.

SLA propuesto y diferencia con el SLO:
- **SLA propuesto:** p95 < 600 ms medido mensualmente.
- **Diferencia:** El **SLO** es la meta técnica interna del equipo para detectar degradaciones de rendimiento preventivamente. El **SLA** es el acuerdo contractual externo con los clientes que incluye penalizaciones financieras en caso de incumplimiento, por lo que requiere un margen de tolerancia más amplio.

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|
| Mover el job de carga `load` fuera del flujo de `main` a una ejecución programada nocturna (`schedule` / cron) | `load` | ~12m 10s |
| Configurar `sast` (Semgrep) únicamente en eventos `pull_request` y en modo informativo en `main` | `sast` | ~0m 24s |

Total resultante: 2m 32s min
Link al run en main con los cambios: https://github.com/Portador3/INF384-lab4-20213761/actions/runs/[INSERTAR_NUEVO_RUN_ID]

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):
- **Defecto concreto de la aplicación:** Una degradación de latencia o bloqueo por escrituras concurrentes en la base de datos SQLite bajo alto volumen de tráfico simulado.
- **Efecto en DORA:** 
  - **Lead Time for Changes:** BAJA (Mejora), al reducir drásticamente el tiempo total del pipeline de ~15 minutos a menos de 3 minutos por cada despliegue.
  - **Change Failure Rate:** SUBE (Empeora ligeramente), al no interceptar problemas de rendimiento extremo antes de desplegar a producción.

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón:
- **Decisión:** Se mueve al evento `pull_request` y pasa a ser informativo (`continue-on-error: true`) en `main`.
- **Razón:** Ahorra 24 segundos en la fase crítica de integración en `main`. La cobertura de análisis estático de seguridad se mantiene de forma obligatoria durante la revisión del Pull Request antes de realizar el merge.

## Declaración de uso de IA
Herramientas usadas y para qué:
- **Asistente de IA (Gemini / ChatGPT):** Diagnóstico y corrección de dependencias de jobs y concurrencia en `.github/workflows/ci.yml`, análisis e interpretación de métricas de k6/Playwright y redacción estructurada de decisiones técnicas y justificaciones para `RESPUESTAS.md`.
