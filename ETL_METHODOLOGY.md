# Metodología de Desarrollo: Pipeline ETL (API a SQL)

Esta guía documenta la arquitectura global y el flujo de trabajo estándar para construir pipelines de ingeniería de datos utilizando Python, Pandas y bases de datos relacionales.

---

## Estructura del Proyecto Global

### Fase 1: Exploración y Prototipado (Ambiente: `notebooks/01_eda_exploratory.ipynb`)
**Objetivo:** Comprender el funcionamiento de la fuente de datos (API) y validar el acceso.
* Configuración de variables de entorno con `python-dotenv` para proteger credenciales.
* Petición HTTP (`requests.get`) validando código de estado (`200 OK`) y encabezado (`Content-Type: application/json`).
* Inspección de la estructura JSON mediante navegación por niveles (diccionarios y listas).
* Carga inicial de datos crudos hacia un DataFrame de Pandas para visualización tabular.

---

### Fase 2: Validación y Reglas de Limpieza (Ambiente: `notebooks/01_eda_exploratory.ipynb`)
**Objetivo:** Evaluar la calidad de los datos y definir el contrato de transformación.
* **Casteo de tipos:** Conversión de fechas (`pd.to_datetime`) y valores numéricos (`pd.to_numeric`).
* **Estandarización:** Renombrado de columnas a formato snake_case (ej. `date`, `exchange_rate`).
* **Diagnóstico de calidad:**
  * Identificación de valores nulos (`df.isnull().sum()`).
  * Detección de duplicados en la clave primaria (`df.duplicated()`).
  * Revisión de estadísticas descriptivas (`df.describe()`) para detectar outliers o valores incoherentes.
* **Aplicación de reglas:** Filtrado con `dropna()` y `drop_duplicates()`.

---

### Fase 3: Refactorización a Módulos (Ambiente: Carpeta `src/`)
**Objetivo:** Convertir el código de exploración en funciones modulares, puras y desacopladas.

* **`src/extract.py`:**
  * Función parametrizada (ID de serie, fecha de inicio, fecha de fin).
  * Manejo de excepciones y errores HTTP.
  * Retorna exclusivamente la estructura JSON cruda.

* **`src/transform.py`:**
  * Recibe los datos crudos en formato JSON.
  * Aplica las reglas de limpieza probadas en la Fase 2 con Pandas.
  * Retorna exclusivamente un DataFrame limpio.

* **`src/load.py`:**
  * Recibe el DataFrame transformado.
  * Gestiona la conexión a la base de datos objetivo con SQLAlchemy.
  * Ejecuta la inserción de datos (`df.to_sql`).

---

### Fase 4: Orquestación (Ambiente: Raíz `main.py`)
**Objetivo:** Integrar el flujo completo para permitir su ejecución punta a punta.
* Importación de los módulos de la carpeta `src/`.
* Definición de la función de control (`run_pipeline()`).
* Ejecución secuencial: **Extract** $\rightarrow$ **Transform** $\rightarrow$ **Load**.
* Adición del punto de entrada estándar `if __name__ == "__main__":`.

---

## Guía de Celdas para Jupyter Notebook (`notebooks/01_eda_exploratory.ipynb`)

Dentro del archivo `.ipynb`, el trabajo se organiza alternando celdas de Markdown y Código:

### [Markdown]
# 1. Exploración y Conexión a la API (Fase 1)
Extraemos los datos crudos del Tipo de Cambio FIX desde la API de Banxico y los cargamos en un DataFrame inicial.

### [Código]
import os
import requests
import pandas as pd
from dotenv import load_dotenv
from datetime import datetime

# Cargar variables de entorno desde la raíz
load_dotenv(dotenv_path="../.env")
BANXICO_TOKEN = os.getenv("BANXICO_TOKEN")

# Parámetros de consulta
series_id = "SF63528"
fecha_inicio = "2020-01-01"
fecha_fin = datetime.now().strftime("%Y-%m-%d")

url = f"https://www.banxico.org.mx/SieAPIRest/service/v1/series/{series_id}/datos/{fecha_inicio}/{fecha_fin}"
headers = {"Bmx-Token": BANXICO_TOKEN}

response = requests.get(url, headers=headers)

if response.status_code == 200 and "application/json" in response.headers.get("Content-Type", ""):
    raw_data = response.json()['bmx']['series'][0]['datos']
    df = pd.DataFrame(raw_data)
    print("Carga exitosa. Registros obtenidos:", len(df))
    print(df.head())

---

### [Markdown]
## 2. Transformación y Validación en Pandas (Fase 2)
Aplicamos casteo de tipos, limpieza de nulos y estandarización de columnas.

### [Código]
# 1. Renombrar columnas
df = df.rename(columns={'fecha': 'date', 'dato': 'exchange_rate'})

# 2. Conversión de tipos de datos
df['date'] = pd.to_datetime(df['date'], format='%d/%m/%Y')
df['exchange_rate'] = pd.to_numeric(df['exchange_rate'], errors='coerce')

# 3. Diagnóstico de calidad
print("Tipos de datos:\n", df.dtypes)
print("\nValores nulos:\n", df.isnull().sum())
print("\nDuplicados por fecha:", df['date'].duplicated().sum())

# 4. Limpieza final
df_clean = df.dropna().drop_duplicates(subset=['date'])
print("\nDataFrame resultante:", df_clean.shape)

---

### [Markdown]
## 3. Conclusiones para Producción
* La respuesta JSON requiere acceder al índice `[0]` de la lista `series`.
* Las fechas requieren el formato `%d/%m/%Y` para convertirse correctamente a datetime.
* Toda la lógica probada queda lista para migrarse a `src/extract.py` y `src/transform.py`.