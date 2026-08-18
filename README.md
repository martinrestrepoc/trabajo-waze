# Trabajo Waze

Este proyecto analiza la red vial de La Estrella, Antioquia, con datos de OpenStreetMap y calcula rutas mediante una implementación propia del algoritmo A*.

- `WAZE_IA_CORRECCION.ipynb` es la versión principal. Implementa la búsqueda con la estructura de `Node` y `Tree`: `OSMRouteNode` adapta cada estado a la red vial, calcula costos y heurísticas y conserva su nodo padre, mientras `OSMRouteTree` utiliza el algoritmo A* heredado de `Tree` para encontrar la mejor ruta.
- `Waze_IA.ipynb` realiza la misma búsqueda y utiliza el mismo modelo de costos, pero implementa A* directamente en `OSMRouteMap` y `CompositeOSMRouteMap`. En lugar de objetos enlazados mediante padres, administra los mejores costos y la reconstrucción de la ruta con los diccionarios `g_score` y `came_from`.

El entorno se administra con [uv](https://docs.astral.sh/uv/). El proyecto está fijado a Python 3.11 y las versiones reproducibles de las dependencias se encuentran en `uv.lock`.

## Requisitos

- Git.
- Visual Studio Code.
- Las extensiones [Python](https://marketplace.visualstudio.com/items?itemName=ms-python.python) y [Jupyter](https://marketplace.visualstudio.com/items?itemName=ms-toolsai.jupyter) de Microsoft para VS Code.
- Conexión a internet para instalar las dependencias y consultar OpenStreetMap y Nominatim.

No es necesario instalar Python manualmente: `uv` puede descargar la versión 3.11 indicada en `.python-version`.

## 1. Clonar el repositorio

```bash
git clone URL_DEL_REPOSITORIO
cd trabajo-waze
```

Reemplaza `URL_DEL_REPOSITORIO` por la dirección HTTPS o SSH del repositorio en GitHub.

## 2. Instalar uv

Si ya tienes `uv`, comprueba la instalación:

```bash
uv --version
```

Si el comando no existe, instálalo siguiendo una de estas opciones.

### macOS o Linux

Instalador oficial:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

En macOS también puedes utilizar Homebrew:

```bash
brew install uv
```

### Windows

Ejecuta en PowerShell:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Después de la instalación, cierra y abre nuevamente la terminal y verifica:

```bash
uv --version
```

Consulta la [guía oficial de instalación de uv](https://docs.astral.sh/uv/getting-started/installation/) si necesitas otro método.

## 3. Crear el entorno e instalar las dependencias

Desde la raíz del repositorio ejecuta:

```bash
uv sync
```

Este comando:

- Lee Python 3.11 desde `.python-version`.
- Descarga Python 3.11 si no está disponible.
- Crea el entorno virtual `.venv`.
- Instala las dependencias registradas en `pyproject.toml` y `uv.lock`.

Puedes verificar el entorno con:

```bash
uv run python --version
```

El resultado debe comenzar con `Python 3.11`.

## 4. Abrir el proyecto en VS Code

1. Abre Visual Studio Code.
2. Selecciona **File → Open Folder** o **Archivo → Abrir carpeta**.
3. Abre la carpeta clonada `trabajo-waze`.
4. Abre el notebook que quieras ejecutar, por ejemplo `Waze_IA.ipynb`.

## 5. Seleccionar el kernel correcto

Con el notebook abierto:

1. Haz clic en **Select Kernel** o **Seleccionar kernel**, en la esquina superior derecha.
2. Selecciona **Python Environments**.
3. Elige el intérprete ubicado dentro de `.venv`:
   - macOS y Linux: `.venv/bin/python`
   - Windows: `.venv\Scripts\python.exe`
4. Confirma que VS Code muestre el entorno del proyecto con Python 3.11.

No selecciones un Python global ni un entorno de otro proyecto.

## 6. Ejecutar el notebook

Después de seleccionar el kernel de `.venv`, utiliza el botón **Run All** o **Ejecutar todo** de VS Code. También puedes ejecutar las celdas individualmente mediante el botón de ejecución que aparece junto a cada una, siempre respetando su orden.

La primera ejecución puede tardar mientras OSMnx descarga la red vial de OpenStreetMap. Las consultas de geocodificación realizadas con Nominatim también necesitan conexión a internet y ocasionalmente pueden requerir un nuevo intento si el servicio tarda en responder.

## Solución de problemas

### VS Code no muestra `.venv` como kernel

1. Comprueba que ejecutaste `uv sync` desde la raíz del repositorio.
2. Verifica que las extensiones Python y Jupyter estén instaladas y habilitadas.
3. En VS Code, abre la paleta de comandos y selecciona **Python: Select Interpreter**.
4. Selecciona manualmente `.venv/bin/python` en macOS o Linux, o `.venv\Scripts\python.exe` en Windows.
5. Si todavía no aparece, ejecuta **Developer: Reload Window** desde la paleta de comandos.

### Falta una dependencia

No instales paquetes con `pip` dentro del notebook. Agrégalos al proyecto desde la terminal:

```bash
uv add nombre-del-paquete
```

Después reinicia el kernel del notebook en VS Code y vuelve a ejecutar las celdas necesarias.

### Reconstruir el entorno

Si el entorno quedó desactualizado o fue eliminado, vuelve a ejecutar:

```bash
uv sync
```

La carpeta `.venv` es local y no debe subirse a GitHub. El archivo `uv.lock` sí debe mantenerse en el repositorio para que otros usuarios obtengan las mismas versiones de las dependencias.

---
## Referencias

### Fundamento de los ajustes de tiempo

Las penalizaciones por intersecciones, movimientos de giro y complejidad vial fueron extraídos de estudios de la [Federal Highway Administration (FHWA)](https://ops.fhwa.dot.gov/publications/fhwahop08024/chapter3.htm), [Transportation Research Part B](https://doi.org/10.1016/0191-2615(96)00001-X) y la [American Society of Civil Engineers (ASCE)](https://doi.org/10.1061/9780784483053.224).
