# TicketMania — taller de JMeter

Taller individual de pruebas de rendimiento sobre la API de compra de boletas TicketMania. Alcance: `Presentacion_JMeter_Ticketmania_v2.pdf`, `Despliegue_JMeter_Tiquetmania.pdf` y `comunicadoProfe.md`. El taller independiente de experimentación «¿Qué pasaría si…?» queda fuera.

## Archivos de la entrega

| Ruta | Contenido |
| --- | --- |
| `api/ticketmania_simple.py` | API original de la profesora, sin modificaciones. |
| `pruebas/jmeter/Ticketmania.jmx` | Plan de JMeter con consulta, reserva, extracción y pago validado. |
| `pruebas/jmeter/datos.csv` | Cuatro compras válidas; se reciclan para los 200 usuarios. |
| `.github/workflows/rendimiento.yml` | Pipeline de ejecución, validación del JTL y publicación del reporte. |
| `README.md` | Alcance, ejecución y verificación. |

Los PDF y Markdown de la profesora se conservan como referencia en la carpeta local. El repositorio de entrega contiene los cinco archivos de la tabla.

Repositorio: https://github.com/Isaac-Chaves-G/ticketmania-jmeter

## Requisitos

- Python 3.8 o superior. La API solo usa la biblioteca estándar.
- JMeter 5.6.3 y Java 17 o superior en local. El pipeline debe instalar Java 17.
- Para el trabajo local de la presentación: Plugins Manager, **jpgc Standard Set**, **jpgc PerfMon Metrics Collector** y **ServerAgent 2.2.1**, en el puerto 4444. PerfMon no se ejecuta en el pipeline.

## Ejecutar la API en Windows

Desde la raíz del proyecto, en PowerShell:

```powershell
python -u .\api\ticketmania_simple.py
```

Mantener esa terminal abierta durante la prueba. Detener con `Ctrl+C`. En otra terminal:

```powershell
Invoke-RestMethod http://127.0.0.1:8080/api/salud
Invoke-RestMethod http://127.0.0.1:8080/api/tarifas
```

La primera consulta debe devolver `estado: OK`.

## Flujo de compra

| Orden | Método y ruta | Datos o validación |
| --- | --- | --- |
| 1 | `GET /api/tarifas` | Consultar las tarifas y sus precios. |
| 2 | `POST /api/seleccion` | Enviar `{"tarifa":"${tarifa}","cantidad":${cantidad}}`. Se espera HTTP 201. |
| 3 | Extractor JSON en la selección | Guardar el campo `reserva` en la variable `num_reserva`. |
| 4 | `POST /api/pago` | Enviar `{"reserva":"${num_reserva}"}`. Se espera HTTP 200. |
| 5 | Aserción JSON en el pago | Verificar que `estado` sea `APROBADO`. |

Ambas peticiones POST usan `Accept: application/json` y `Content-Type: application/json`.

| Tarifa | Precio por boleta (COP) |
| --- | ---: |
| `general` | 80000 |
| `occidental` | 120000 |
| `preferencial` | 180000 |
| `vip` | 350000 |

Cada fila del CSV contiene `tarifa,cantidad`; la cantidad es un entero positivo. El plan y el CSV van juntos en `pruebas/jmeter/`, con `datos.csv` como nombre de archivo relativo.

## Escenario configurado

| Parámetro | Valor |
| --- | --- |
| Grupo | Grupo de Hilos estándar |
| Usuarios | 200 |
| Periodo de subida | 300 segundos |
| Contador del bucle | 1 |
| Temporizador Aleatorio Uniforme | Constante de 500 ms y máximo aleatorio de 1000 ms: pausa de 500 a 1500 ms. |

Las mediciones se ejecutan sin interfaz gráfica. Antes de medir se deshabilita Debug Sampler y se evita Ver Árbol de Resultados en pruebas largas.

Cada usuario realiza tres peticiones: tarifas, selección y pago. Se esperan **600 muestras: 200 de cada petición**. El temporizador está en el Grupo de Hilos y se aplica antes de cada petición. Los 200 usuarios llegan gradualmente; no permanecen todos activos durante cinco minutos.

El extractor usa `$.reserva` y la aserción del pago verifica `$.estado = APROBADO`. Si una petición o su aserción falla, se detiene ese hilo. El plan no incluye Debug Sampler ni Ver Árbol de Resultados, para evitar muestras y consumo adicionales.

## Ejecutar JMeter sin interfaz en Windows

Con la API abierta, desde la raíz del proyecto en otra terminal PowerShell:

```powershell
Set-Location .\pruebas\jmeter
$jmeter = 'C:\Users\ichav\Downloads\apache-jmeter-5.6.3\apache-jmeter-5.6.3\bin\jmeter.bat'
$salida = 'ejecucion-' + (Get-Date -Format 'yyyyMMdd-HHmmss')
& $jmeter -n -t Ticketmania.jmx -l "$salida.jtl" -j "$salida.log" -e -o "$salida-reporte"
if ($LASTEXITCODE -ne 0) { throw 'JMeter no pudo completar la ejecución.' }
$muestras = @(Import-Csv "$salida.jtl")
$fallidas = @($muestras | Where-Object { $_.success -ne 'true' })
$conteos = @($muestras | Group-Object label)
$conteos | Select-Object Name, Count
if ($fallidas.Count -gt 0) { throw "Hay $($fallidas.Count) peticiones fallidas." }
if ($muestras.Count -ne 600 -or $conteos.Count -ne 3 -or @($conteos | Where-Object { $_.Count -ne 200 }).Count -gt 0) { throw 'Se esperaban 200 muestras de cada petición.' }
Invoke-Item "$salida-reporte\index.html"
```

El nombre con fecha permite repetir la medición sin sobrescribir resultados: JMeter necesita una carpeta de reporte nueva o vacía. En otro equipo, ajusta únicamente la ubicación de `jmeter.bat`. En Linux, con `jmeter` en el PATH, inicia la API y ejecuta desde `pruebas/jmeter/`:

```bash
jmeter -n -t Ticketmania.jmx -l resultados.jtl -j jmeter.log -e -o reporte
```

## GitHub Actions y reporte

El workflow `Rendimiento TicketMania` se ejecuta con cada push a `main` o manualmente desde **Actions → Rendimiento TicketMania → Run workflow**. Tiene un solo job:

1. Descarga el commit del repositorio e instala Java 17 y JMeter 5.6.3.
2. Inicia la API en segundo plano en el puerto 8080 y espera hasta 30 intentos a que `/api/salud` responda `OK`.
3. Ejecuta el plan sin interfaz desde `pruebas/jmeter/` y genera `resultados.jtl` y `reporte/`.
4. Revisa el JTL y falla si contiene peticiones fallidas o no incluye exactamente 200 consultas, 200 reservas y 200 pagos. JMeter por sí solo no devuelve un código de error por cada petición fallida.
5. Sube el artefacto **reporte-rendimiento** con `if: always()`, incluyendo el HTML, el JTL y los logs disponibles. Finalmente detiene la API.

Para consultar los resultados, abre la ejecución en Actions y descarga **reporte-rendimiento** desde **Artifacts**. Extrae el ZIP completo y abre `ticketmania-jmeter/ticketmania-jmeter/pruebas/jmeter/reporte/index.html`; el HTML requiere las carpetas de recursos que lo acompañan. El ZIP conserva esa estructura porque también incluye el log de la API desde la carpeta temporal del runner. El artefacto se conserva 14 días.

El pipeline prueba el proyecto en una máquina temporal; no publica una API permanente. Si falla la instalación o el arranque antes de ejecutar JMeter, puede no existir un reporte HTML: los logs disponibles permiten identificar el fallo.

## Leer los resultados

| Indicador | Interpretación |
| --- | --- |
| Muestras | Deben ser 600 en total, 200 por cada petición. |
| Error % | Fracción de peticiones o aserciones fallidas; el pipeline requiere cero fallos. |
| Media y máximo | Tiempo promedio y mayor tiempo de respuesta, en milisegundos. |
| p95 | Tiempo de respuesta que no supera el 95 % de las muestras. |
| Throughput | Peticiones completadas por segundo durante el intervalo medido. |

La petición con la mayor media tiene el mayor costo temporal observado. Con una sola carga no se puede establecer un punto de saturación. La llegada gradual y las pausas limitan el throughput de este escenario; no representa el máximo de la API.

## Estado de la medición

La [ejecución #1](https://github.com/Isaac-Chaves-G/ticketmania-jmeter/actions/runs/43549429572) terminó en **success** el 13 de noviembre de 2026, a las 10:07 p. m. (Colombia). Se ejecutó el commit `bfda9ed35500e1f42bffaf0788680e5965ed0ae7` con Ubuntu, Java Temurin 17 y JMeter 5.6.3.

| Petición | Muestras | Fallidas |
| --- | ---: | ---: |
| Tarifas | 200 | 0 |
| SeleccionTarifaYCantidad | 200 | 0 |
| Pago | 200 | 0 |
| **Total** | **600** | **0** |

La consola de JMeter registró aproximadamente **5 minutos y 1 segundo de medición**, **2,0 peticiones/segundo**, **1 ms de media** y **20 ms de máximo**. Son valores globales redondeados del resumen de consola; el dashboard contiene el detalle por petición y los percentiles.

El resultado confirma que los 200 usuarios completaron la compra y que la aserción `APROBADO` pasó en los 200 pagos. No establece el límite de capacidad: la llegada fue gradual, cada usuario recorrió el flujo una sola vez y la API compartió máquina con JMeter. No se midieron CPU ni memoria con PerfMon en esta ejecución.

[Descargar el reporte y las evidencias](https://github.com/Isaac-Chaves-G/ticketmania-jmeter/actions/runs/43549429572/artifacts/14095648674). El ZIP contiene el HTML, el JTL y los logs.

## Verificación y límites

- Se confirmó el arranque de la API en `127.0.0.1:8080`.
- Se verificaron los handlers HTTP en memoria: salud OK y selección/pago exitosos para las cuatro tarifas, con cantidades y totales correctos. Esta comprobación no sustituye la ejecución por red con JMeter.
- El estudiante confirmó que `GET /api/salud` responde `estado: OK` desde su terminal. El entorno de asistencia bloquea las conexiones HTTP locales con `WinError 10013`; la verificación completa por red se realiza mediante el pipeline.
- Las reservas se guardan en memoria: reiniciar la API borra los datos. El pago consume la reserva.
- No hay topes de capacidad, cola ni límite de boletas. El pago se aprueba para una reserva válida.
- API y JMeter comparten equipo, también en GitHub Actions. El pipeline comprueba el flujo completo; no determina la capacidad de la API.
- Las guías dejan los umbrales de aceptación pendientes de clase.
