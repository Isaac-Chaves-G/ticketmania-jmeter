# TicketMania

Pruebas de rendimiento de una API de compra de boletas con Apache JMeter. El flujo consulta tarifas, crea una reserva y realiza el pago.

## Entrega de experimentación local

- [Documento del taller con las diez respuestas y sus capturas](entrega/Taller_Que_pasaria_si_Entrega.docx).
- [Evidencias de las 26 mediciones completas](entrega/Evidencias_Experimentacion.zip).

Se ejecutaron 13 escenarios con dos repeticiones por escenario. El documento compara ambas repeticiones e incluye 27 capturas de configuración y resultados, sin duplicarlas cuando sirven para varias preguntas. El ZIP conserva los JTL originales, las mediciones de PerfMon, los logs, las configuraciones, los planes de cada ejecución y los 26 reportes HTML. `comparacion.json` contiene las métricas y los hashes SHA 256 de los archivos originales.

Para consultar las evidencias, extraer todo el ZIP conservando sus carpetas. Los reportes se encuentran en `pruebas/experimentacion/resultados/20261113_224419/<escenario>_r<repeticion>/reporte_verificado/index.html`. Sus recursos estáticos se comparten entre reportes; no mover un HTML por separado. Las capturas seleccionadas están en la carpeta `capturas` del mismo lote.

Las mediciones son locales: la API y JMeter comparten computador. Los errores observados bajo carga se conservan en los resultados y se analizan en el documento. La prueba sin pausas que quedó incompleta se repitió y el intento parcial quedó fuera de la entrega.

## Archivos

| Ruta | Contenido |
| --- | --- |
| `api/ticketmania_simple.py` | API HTTP de tarifas, reservas y pagos. |
| `pruebas/jmeter/Ticketmania.jmx` | Plan ejecutado por GitHub Actions. |
| `pruebas/jmeter/datos.csv` | Datos del flujo de compra. |
| `pruebas/experimentacion/Experimentacion.jmx` | Plan parametrizado de carga, estrés y concurrencia. |
| `pruebas/experimentacion/datos.csv` | Tarifas, cantidades y precios esperados para validar el total. |
| `pruebas/experimentacion/escenarios.json` | Parámetros de los 13 escenarios. |
| `.github/workflows/rendimiento.yml` | Ejecución del plan base y publicación del reporte. |

## Requisitos

- Python 3.8 o superior.
- Java 17 o superior.
- Apache JMeter 5.6.3, con `bin` en el `PATH`.
- Para CPU y memoria: PerfMon Metrics Collector y ServerAgent en el equipo medido.

## Iniciar la API

Desde la raíz del repositorio, en PowerShell:

```powershell
python -u ./api/ticketmania_simple.py
```

Mantener esa terminal abierta. La API escucha en `http://127.0.0.1:8080`.

| Método | Ruta | Operación |
| --- | --- | --- |
| GET | `/api/salud` | Comprobar disponibilidad. |
| GET | `/api/tarifas` | Consultar tarifas. |
| POST | `/api/seleccion` | Crear una reserva con `tarifa` y `cantidad`. |
| POST | `/api/pago` | Pagar usando el identificador `reserva`. |

Las reservas se guardan en memoria y se consumen al pagarlas. Reiniciar la API las elimina.

## Ejecutar el plan base

En otra terminal, desde la raíz:

```powershell
Set-Location ./pruebas/jmeter
$salida = 'ejecucion-' + (Get-Date -Format 'yyyyMMdd-HHmmss')
jmeter.bat -n -t Ticketmania.jmx -l "$salida.jtl" -j "$salida.log" -e -o "$salida-reporte"
Invoke-Item "$salida-reporte/index.html"
```

Configuración: 200 usuarios, subida de 300 s, un ciclo y pausas de 500 a 1.500 ms. Cada usuario consulta tarifas, reserva y paga: 600 peticiones en total.

## Ejecutar la experimentación

En otra terminal, desde la raíz, ejemplo con 100 usuarios:

```powershell
Set-Location ./pruebas/experimentacion
$salida = 'experimento-' + (Get-Date -Format 'yyyyMMdd-HHmmss')
jmeter.bat -n -t Experimentacion.jmx -Jpuerto=8080 -Jusuarios=100 -Jsubida=10 -Jduracion=60 -Jciclos=-1 -Jpausa=500 -Jaleatorio=1000 -Jmodo=flujo -Jsincronizados=1 -l "$salida.jtl" -j "$salida.log" -e -o "$salida-reporte"
Invoke-Item "$salida-reporte/index.html"
```

`escenarios.json` documenta los valores de cada escenario; se aplican mediante propiedades `-J` en el comando. No se carga automáticamente.

| Propiedad | Función |
| --- | --- |
| `usuarios`, `subida` | Hilos y periodo de incorporación en segundos. |
| `duracion`, `ciclos` | Duración en segundos y ciclos por usuario; `-1` repite durante la duración. |
| `pausa`, `aleatorio` | Pausa constante y rango aleatorio en milisegundos; ambos en cero eliminan la pausa. |
| `modo` | `flujo`, `tarifas` o `seleccion`. |
| `sincronizados` | Tamaño de la barrera: `1` para llegadas sin agrupación o `500` para sincronizar. Espera máxima: 30 s. |
| `puerto` | Puerto de la API; usar `8080` con el comando de inicio anterior. |

El CSV experimental contiene `tarifa,cantidad,precio`. Las aserciones verifican `total = precio × cantidad` y la respuesta de pago `APROBADO`.

Para registrar CPU y memoria, iniciar ServerAgent y añadir al plan un PerfMon Metrics Collector con host `127.0.0.1`, puerto `4444`, métricas `CPU` y `Memory`, y una ruta de salida para sus muestras. API y JMeter comparten equipo en las pruebas locales.

Cada ejecución genera su JTL, log y reporte HTML en una salida nueva. Comparar muestras, media, máximo, porcentaje de errores y throughput.

## GitHub Actions

El workflow **Rendimiento TicketMania** ejecuta únicamente `pruebas/jmeter/Ticketmania.jmx`. Comprueba 600 muestras exitosas y publica el artefacto `reporte-rendimiento`, con HTML, JTL y logs durante 14 días. Puede iniciarse desde **Actions → Rendimiento TicketMania → Run workflow**. El plan de experimentación se ejecuta localmente.
