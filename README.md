# TicketMania

Proyecto de pruebas de rendimiento de una API de compra de boletas con Apache JMeter y GitHub Actions. Simula la consulta de tarifas, la creación de una reserva y su pago, con datos leídos desde un CSV y validación de la respuesta.

## Estructura

| Archivo | Función |
| --- | --- |
| `api/ticketmania_simple.py` | API HTTP implementada con la biblioteca estándar de Python. |
| `pruebas/jmeter/Ticketmania.jmx` | Plan de pruebas con peticiones HTTP, extractor JSON, temporizador y aserción de pago. |
| `pruebas/jmeter/datos.csv` | Tarifas y cantidades utilizadas en las compras. |
| `.github/workflows/rendimiento.yml` | Ejecución automática de las pruebas y publicación del reporte. |
| `README.md` | Documentación del proyecto. |

## Requisitos

- Python 3.8 o superior.
- Java 17 o superior.
- Apache JMeter 5.6.3, con su carpeta `bin` agregada al `PATH`.

La API no requiere dependencias adicionales de Python. El plan utiliza componentes incluidos en JMeter, sin plugins adicionales.

## API y flujo de compra

La API se ejecuta en `http://127.0.0.1:8080`.

| Método | Ruta | Función | Respuesta exitosa |
| --- | --- | --- | --- |
| GET | `/api/salud` | Comprobar disponibilidad. | HTTP 200, `{"estado":"OK"}`. |
| GET | `/api/tarifas` | Consultar las tarifas y sus precios. | HTTP 200, listado de tarifas. |
| POST | `/api/seleccion` | Crear una reserva con tarifa y cantidad. | HTTP 201, identificador de reserva y total. |
| POST | `/api/pago` | Pagar una reserva existente. | HTTP 200, estado `APROBADO` y datos de compra. |

Los POST envían JSON con las cabeceras `Content-Type: application/json` y `Accept: application/json`.

El plan envía la selección con `{"tarifa":"${tarifa}","cantidad":${cantidad}}`. Un extractor JSON lee `$.reserva` y guarda el identificador en `num_reserva`. El pago utiliza `{"reserva":"${num_reserva}"}`, y una aserción JSON comprueba que `$.estado` sea `APROBADO`.

Las reservas se almacenan en memoria y se consumen al pagarlas. Reiniciar la API elimina las reservas existentes.

### Tarifas y datos de prueba

| Tarifa | Precio por boleta (COP) |
| --- | ---: |
| `general` | 80.000 |
| `occidental` | 120.000 |
| `preferencial` | 180.000 |
| `vip` | 350.000 |

Contenido de `datos.csv`:

```csv
tarifa,cantidad
general,1
occidental,3
preferencial,2
vip,1
```

Cada fila representa una compra. El CSV omite el encabezado al leer los datos, se comparte entre todos los hilos y vuelve al principio al llegar al final. La ruta configurada es `datos.csv`, relativa al plan.

## Escenario de prueba

| Parámetro | Configuración |
| --- | --- |
| Grupo | Grupo de Hilos estándar |
| Usuarios | 200 |
| Periodo de subida | 300 segundos |
| Iteraciones por usuario | 1 |
| Temporizador | Aleatorio uniforme: 500 ms constantes más hasta 1.000 ms aleatorios |
| Peticiones por usuario | Consulta de tarifas, selección y pago |
| Total esperado | 600 peticiones: 200 de cada operación |

El temporizador aplica una pausa de 500 a 1.500 ms antes de cada petición. Los usuarios ingresan gradualmente y cada uno realiza una compra completa. Si una petición o su aserción falla, se detiene ese hilo.

## Ejecución local

### Windows: PowerShell

Desde la raíz del repositorio, iniciar la API y mantener la terminal abierta:

```powershell
python -u ./api/ticketmania_simple.py
```

En otra terminal, comprobar su disponibilidad:

```powershell
Invoke-RestMethod http://127.0.0.1:8080/api/salud
```

Desde la raíz del repositorio, ejecutar JMeter sin interfaz y abrir el reporte:

```powershell
Set-Location ./pruebas/jmeter
$salida = 'ejecucion-' + (Get-Date -Format 'yyyyMMdd-HHmmss')
jmeter.bat -n -t Ticketmania.jmx -l "$salida.jtl" -j "$salida.log" -e -o "$salida-reporte"
Invoke-Item "$salida-reporte/index.html"
```

### Linux o macOS

Iniciar la API desde la raíz del repositorio:

```bash
python3 -u api/ticketmania_simple.py
```

En otra terminal, desde la raíz del repositorio:

```bash
curl --fail http://127.0.0.1:8080/api/salud
cd pruebas/jmeter
salida="ejecucion-$(date +%Y%m%d-%H%M%S)"
jmeter -n -t Ticketmania.jmx -l "$salida.jtl" -j "$salida.log" -e -o "$salida-reporte"
```

Abrir `index.html` dentro de la carpeta de reporte generada. Los nombres con fecha permiten repetir las pruebas usando una carpeta nueva. El archivo `.jtl` contiene las muestras y el `.log` los registros de JMeter.

## GitHub Actions

El workflow **Rendimiento TicketMania** se ejecuta al hacer push a `main` o manualmente desde **Actions > Rendimiento TicketMania > Run workflow**.

El job utiliza `ubuntu-latest` y realiza las siguientes operaciones:

1. Descarga el repositorio e instala Java 17 y JMeter 5.6.3.
2. Inicia la API en segundo plano y espera una respuesta correcta de `/api/salud`.
3. Ejecuta JMeter sin interfaz desde `pruebas/jmeter/` y genera el reporte HTML.
4. Lee `resultados.jtl` y comprueba que existan 600 muestras exitosas: 200 consultas, 200 reservas y 200 pagos.
5. Publica el artefacto **reporte-rendimiento** y detiene la API.

El job falla si un paso termina con error, si hay muestras fallidas o si el conteo de peticiones no coincide. La publicación del artefacto utiliza `if: always()` para conservar las evidencias disponibles incluso cuando falle la prueba.

El artefacto incluye el reporte HTML, el JTL y los logs de JMeter y de la API, con una retención de 14 días. Después de descargarlo desde **Artifacts**, extraer el ZIP completo y abrir:

```text
ticketmania-jmeter/ticketmania-jmeter/pruebas/jmeter/reporte/index.html
```

## Resultados verificados

La [ejecución #1](https://github.com/Isaac-Chaves-G/ticketmania-jmeter/actions/runs/43549429572) finalizó correctamente el 13 de noviembre de 2026 con el commit `bfda9ed35500e1f42bffaf0788680e5965ed0ae7`.

| Petición | Muestras | Fallidas |
| --- | ---: | ---: |
| Tarifas | 200 | 0 |
| SeleccionTarifaYCantidad | 200 | 0 |
| Pago | 200 | 0 |
| **Total** | **600** | **0** |

El resumen de consola registró aproximadamente 5 minutos y 1 segundo de medición, 2,0 peticiones por segundo, 1 ms de respuesta media y 20 ms de respuesta máxima. Estos valores globales están redondeados; el dashboard incluye el detalle por petición y los percentiles.

Los 200 usuarios completaron el flujo de compra y los 200 pagos superaron la aserción `APROBADO`. Las métricas corresponden a una ejecución con llegada gradual y con la API y JMeter en la misma máquina.

[Descargar reporte y evidencias](https://github.com/Isaac-Chaves-G/ticketmania-jmeter/actions/runs/43549429572/artifacts/14095648674).
