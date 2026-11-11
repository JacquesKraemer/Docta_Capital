# Docta Capital: ejemplos de uso de la API con Python

El notebook `docta_api.ipynb` muestra cómo autenticarse en el servicio de datos de Docta, consultar instrumentos de renta fija y visualizar sus calendarios de pagos con Plotly.

Documentación oficial: https://docs.docta.com.ar/onboarding-ia

Las consultas del notebook utilizan `https://api.doctacapital.com.ar/api/v1`. El funcionamiento de las llamadas originales fue confirmado localmente por el usuario con sus credenciales. La documentación enlazada no pudo consultarse desde el entorno de esta actualización; las visualizaciones se basan en el formato de respuesta proporcionado y no incorporan endpoints adicionales.

## Requisitos e instalación

Necesitas Python, Jupyter y credenciales de acceso al servicio de Docta. El notebook original fue creado con Python 3.11.9; la ampliación se validó localmente con Python 3.12.

Desde la carpeta del repositorio:

```bash
python -m venv .venv
source .venv/bin/activate
# En Windows: .venv\Scripts\activate
python -m pip install jupyterlab ipykernel requests pandas python-dotenv plotly openpyxl
python -m ipykernel install --user --name docta --display-name "Python (Docta)"
python -m jupyterlab
```

Abre `docta_api.ipynb`, selecciona el kernel **Python (Docta)** y ejecuta las celdas en orden. Los comandos instalan las versiones disponibles; el repositorio no fija versiones de dependencias.

### Credenciales

Crea un archivo `.env` en la misma carpeta que el notebook:

```dotenv
CLIENT_ID=tu_client_id
CLIENT_SECRET=tu_client_secret
```

`load_dotenv()` carga estas variables al iniciar. Ambas deben estar definidas antes de ejecutar la autenticación. No publiques el `.env`, tus credenciales ni el token. Antes de realizar commits, agrega `.env` y `.venv/` a tu exclusión local de Git o a un `.gitignore`.

## Ejemplos incluidos

| Ejemplo | Endpoint relativo a `/api/v1` | Resultado |
| --- | --- | --- |
| Autenticación | `POST /auth/token` | Obtiene `access_token` con `grant_type=client_credentials`. |
| Listado de bonos | `GET /bonds/instruments?limit=50&sub_asset_class=DOLLAR_LINKED` | Muestra una primera página de instrumentos de la categoría elegida. |
| Ficha de un bono | `GET /bonds/instruments/{bono}` | Consulta los datos generales de `AN29`. |
| Rendimientos intradiarios | `GET /bonds/yields/{bono}/intraday` | Consulta los indicadores devueltos por el servicio, como TIR, TNA, TEM y duration. |
| Calendario de pagos | `GET /bonds/analytics/{ticker}/cashflow` | Guarda en `cash_flow` las respuestas para `AN29`, `AL41` y `AE38`. |
| Visualización | Sin llamadas adicionales | Genera gráficos interactivos a partir de `cash_flow`. |

Puedes cambiar `instrument` para elegir otra categoría admitida por el servicio, `bono` para la ficha y los rendimientos, y `bonos` para los calendarios. Cada ticker de `bonos` genera una consulta independiente. La celda de calendarios comprueba el estado HTTP y tiene un tiempo de espera de 30 segundos por solicitud; ante un error se interrumpe la ejecución. Vuelve a ejecutarla para reconstruir la lista completa.

## Gráficos de flujos de caja

La última celda acepta una lista de diccionarios obtenidos con `response.json()` o una lista de cadenas JSON obtenidas con `response.text`. Por ejemplo, esta estructura permite probar la visualización sin credenciales ni conexión:

```python
cash_flow = [{
    "ticker": "EJEMPLO",
    "data": [{
        "payment_date": "2029-11-30",
        "adj_capital": 100.0,
        "adj_interest_amount": 3.25,
        "cash_flow": 103.25,
    }],
    "metadata": {"nominal_units": 100},
}]
```

Asigna la lista antes de ejecutar la última celda; no ejecutes la celda de consulta de calendarios si quieres conservar tus datos de ejemplo.

Por cada bono se muestra:

- Una barra por fecha, con amortización e intereses ajustados apilados.
- Puntos con el flujo total `cash_flow` informado por la API.
- Detalles al pasar el cursor, leyenda interactiva, zoom y un selector de rango temporal.

La visualización utiliza `adj_capital` y `adj_interest_amount`. En bonos amortizables, el campo `interest_amount` puede diferir del interés efectivo incluido en `cash_flow`; por eso no se utiliza para construir la barra. El total se muestra directamente desde la respuesta, sin recalcularlo ni ocultar diferencias.

Los pagos se ordenan cronológicamente y se incluyen tanto los históricos como los futuros. Los importes corresponden a la base `metadata.nominal_units` de cada respuesta (100 en los ejemplos proporcionados), no a una tenencia particular. Si esa metadata falta, el gráfico lo indica. No se convierten monedas, no se agregan bonos en una cartera ni se calcula rentabilidad a partir de los pagos.

La celda deja disponibles `df_flujos`, una tabla con los pagos y su ticker, y `figuras_flujos`, una lista de figuras Plotly. No modifica la lista original `cash_flow`. Una respuesta con `data=[]` se omite con un aviso; los campos faltantes, importes nulos o fechas inválidas generan un error descriptivo.

Para exportar los resultados, después de ejecutar la visualización:

```python
df_flujos.to_excel("flujos_de_caja.xlsx", index=False)
for indice, figura in enumerate(figuras_flujos, start=1):
    figura.write_html(f"flujo_{indice}.html", include_plotlyjs=True)
```

Los HTML incluyen Plotly y pueden abrirse sin conexión. La exportación es opcional; el notebook no escribe estos archivos automáticamente.

## Dependencias

| Dependencia | Uso |
| --- | --- |
| `requests` | Autenticación y consultas HTTP. |
| `python-dotenv` | Lectura de credenciales desde `.env`. |
| `pandas` | Conversión de pagos en tablas, fechas, ordenamiento y validación de importes. |
| `plotly` | Gráficos interactivos mediante `graph_objects`; también se conservan los imports originales de Express y subplots. |
| `openpyxl` | Import original y motor para la exportación opcional a Excel. |
| `jupyterlab`, `ipykernel` | Ejecución interactiva y kernel de Python. |

`json`, `os`, `urllib.parse` y `datetime` pertenecen a la biblioteca estándar de Python; no se instalan por separado. Algunos imports originales están reservados para ampliar los ejemplos y no se usan en la visualización.

## Alcance y resolución de problemas

El notebook es un ejemplo educativo de consulta de datos y no envía órdenes de compra o venta. El listado solicita una sola página de hasta 50 instrumentos y no recorre cursores. La autenticación original no renueva el token automáticamente ni valida previamente la presencia de credenciales: si faltan, puede fallar en `urllib.parse.quote`; si el token vence, vuelve a ejecutar la autenticación y las consultas dependientes.

Ante un fallo de acceso, verifica credenciales, conectividad, permisos y disponibilidad del ticker según la documentación del servicio. Si aparecen gráficos vacíos, comprueba que `data` contenga pagos. Las salidas guardadas de las consultas originales son ejemplos históricos y no garantizan valores actuales.
