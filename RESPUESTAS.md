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
