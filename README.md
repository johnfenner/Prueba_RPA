# Prueba Técnica PIX Robotics

---

## 📌 Descripción del proyecto

Este robot en **PIX Studio v2.27.4** realiza un flujo que:

1) **Descarga** (o reutiliza si ya existe) la respuesta del endpoint **Fake Store API** vía **HTTP GET** y la **guarda como respaldo** `.json`.  
2) **Extrae** por transacción los campos requeridos (`id`, `title`, `price`, `category`, `description`).  
3) **Inserta** en una **base de datos SQLite** (evitando duplicados por `id` y agregando `fecha_insercion`).  
4) **Genera** un **reporte Excel** basado en una **Plantilla.xlsx** con:  
   - **Hoja Productos**: todos los registros.  
   - **Hoja Resumen**: **total de productos**, **precio promedio general**, **promedio por categoría** y **cantidad por categoría**.  


## 🗂 Estructura de carpetas 
```
Prueba_RPA/
├─ Data/
│  ├─ Config.xlsx               ← Parámetros de configuración para el robot
│  ├─ FakeStore.db              ← SQLite (se crea en la 1ª ejecución)
│  └─ Productos_YYYY-MM-DD.json ← Respaldo JSON del día (se crea si no existe)
├─ Framework/
│  ├─ ReadConfig.pix
│  ├─ InitApplications.pix
│  ├─ GetTransactionItem.pix
│  ├─ ProcessTransactionItem.pix
├─ Logs/                         ← (opcional) 
├─ Evidencias/
│  └─ No se alcanza a terminar
├─ Reportes/
│  └─ Reporte_YYYY-MM-DD.xlsx    (se genera por día)
└─ Main.pix
```

---

## 🔧 Requisitos y dependencias

### Software
- **PIX Studio v2.27.4** (obligatorio).
- **Microsoft Excel** (para abrir/validar el reporte).

### Base de datos (SQLite vía ODBC)
- **Driver ODBC “SQLite3 ODBC Driver” (x64)** instalado.  
  - **Proveedor en PIX:** `System.Data.Odbc`  
  - **Cadena (fx):**  
    `"Driver=SQLite3 ODBC Driver;Database=" + Convert.ToString(Config["dbPath"]) + ";"`


---

## 🧩 Configuración — `Data\Config.xlsx` 

Agrega o verifica estas filas (ajusta rutas a tu equipo):

| Key               | Value                                                                |
|-------------------|----------------------------------------------------------------------|
| apiUrl            | https://fakestoreapi.com/products                                    |
| jsonFilePath      | C:\Prueba_RPA\Data\Productos_{yyyy-MM-dd}.json |
| dbPath            | C:\Prueba_RPA\Data\FakeStore.db |
| excelReport       | C:\Prueba_RPA\Reportes\Reporte_{yyyy-MM-dd}.xlsx |
| formUrl           |  |
| logsPath          | C:\Prueba_RPA\Prueba_RPA\Logs |
| evidencePath      | C:\Prueba_RPA\Prueba_RPA\Evidencias |
| onedrivePathJson  |                                                           |
| onedrivePathExcel |                                                    |


> Las claves **`jsonFilePath`** y **`excelReport`** **DEBEN** incluir `{yyyy-MM-dd}` (el robot lo reemplaza por la fecha actual).

---

## ▶️ Pasos para ejecución (clic-a-clic)

### 1) Abrir el proyecto
1. Inicia **PIX Studio v2.27.4**.  
2. **Archivo → Abrir proyecto** → selecciona el `.pixproj` dentro de `Prueba_RPA/`.

### 2) Verificar Config 
1. Abre `Data\\Config.xlsx` y confirma las **rutas** anteriores.  

### 3) Proveedor y conexión a BD (en `InitApplications.pix`)
- **Actividad**: *Conectar a base de datos*  
  - **Proveedor (fx):**  `"System.Data.Odbc"`
  - **Cadena (fx):**     `"Driver=SQLite3 ODBC Driver;Database=" + Convert.ToString(Config["dbPath"]) + ";"`
  - **Out:** `dbConn`

- **Crear tabla si no existe** (NonQuery):  
  `"CREATE TABLE IF NOT EXISTS Productos (
     id INTEGER PRIMARY KEY,
     title TEXT,
     price REAL,
     category TEXT,
     description TEXT,
     fecha_insercion TEXT
   );"`

### 4) Descarga JSON / respaldo local (en `InitApplications.pix`)
- **IF** `!File.Exists(jsonPathToday)`  
  - **Enviar solicitud HTTP (GET)**  
    - **URL (fx):** `Convert.ToString(Config["apiUrl"])`  
    - **Out:** `jsonContent`  
  - **Ejecutar código C#** (guardar UTF-8):
    ```csharp
    var dir = System.IO.Path.GetDirectoryName(jsonPathToday);
    if (!string.IsNullOrEmpty(dir)) System.IO.Directory.CreateDirectory(dir);
    System.IO.File.WriteAllText(jsonPathToday, jsonContent ?? "", System.Text.Encoding.UTF8);
    ```
  - **(Opcional) Subida JSON a OneDrive**  
    - **Método**: PUT  
    - **URL (fx):**
      `"https://graph.microsoft.com/v1.0/me/drive/root:" 
        + Convert.ToString(Config["onedrivePathJson"]) 
        + "Productos_" + DateTime.Now.ToString("yyyy-MM-dd") + ".json:/content"`
    - **Headers**:
      - `Authorization = "Bearer " + accessToken`  
      - `Content-Type = "application/json"`
    - **Datos (texto)**: `jsonContent`

> Si el JSON **ya existe**, el robot **NO** vuelve a descargar (idempotencia).

### 5) Conteo de items (para el WHILE de Main)
- **Ejecutar C#** (en `InitApplications.pix`) para calcular `totalCount`:
```csharp
var txt = System.IO.File.ReadAllText(jsonPathToday, System.Text.Encoding.UTF8);
var arr = Newtonsoft.Json.Linq.JArray.Parse(txt ?? "[]");
totalCount = arr.Count;
```

### 6) Bucle principal (en `Main.pix`)
- **Variables** en Main (tipadas):  
  `Config: Dictionary<string, object>` | `jsonPathToday: String` | `dbConn: Object` | `totalCount: Int32` | `Index: Int32` | `TransactionItem: Object` | `excelPathToday: String`
- **Orden**:
  1. `ReadConfig` → `Config`  
  2. `InitApplications` → `jsonPathToday`, `dbConn`, `totalCount`  
  3. `Index = 0`  
  4. **Mientras (`Index < totalCount`)**  
     - `GetTransactionItem (In: jsonPathToday, Index → Out: TransactionItem)`  
     - **IF (`TransactionItem != null`)**  
       - `ProcessTransactionItem (In: TransactionItem, Config, dbConn → Out: excelPathToday)`  
     - `Index = Index + 1`  
  5. `CloseApplications (In: dbConn)`

### 7) GetTransactionItem (leer item por índice)
- **Leer archivo** `jsonPathToday` → `jsonText`.  
- **IF** `(jsonText ?? "").Trim().Length == 0` → `TransactionItem = null`.  
- **Else → Ejecutar C#**:
```csharp
var arr = Newtonsoft.Json.Linq.JArray.Parse(jsonText ?? "[]");
var i = System.Convert.ToInt32(Index);
if (i < 0 || i >= arr.Count) { TransactionItem = null; }
else { TransactionItem = arr[i]; }
```

### 8) ProcessTransactionItem (persistencia + KPIs + Excel)
> **IMPORTANTE (ODBC):** en SQL usa **`?`** y respeta **orden** de parámetros.

1. **COUNT** (Escalar):  
   - **SQL (fx):** `"SELECT COUNT(*) FROM Productos WHERE id = ?;"`  
   - **Parámetro 1 (fx):**  
     `Convert.ToInt32(((Newtonsoft.Json.Linq.JToken)TransactionItem)["id"])`  
   - **Out:** `count`

2. **IF** `count == 0` → **INSERT** (NonQuery):  
   - **SQL (fx):**  
     `"INSERT INTO Productos (id, title, price, category, description, fecha_insercion) VALUES (?,?,?,?,?, datetime('now'));"
`
   - **Parámetros (en este orden):**
     1) `Convert.ToInt32(((Newtonsoft.Json.Linq.JToken)TransactionItem)["id"])`  
     2) `Convert.ToString(((Newtonsoft.Json.Linq.JToken)TransactionItem)["title"])`  
     3) `Convert.ToDouble(((Newtonsoft.Json.Linq.JToken)TransactionItem)["price"])`  
     4) `Convert.ToString(((Newtonsoft.Json.Linq.JToken)TransactionItem)["category"])`  
     5) `Convert.ToString(((Newtonsoft.Json.Linq.JToken)TransactionItem)["description"])`

3. **SELECT** (DataTable) → `dtProductos`:  
   `"SELECT id, title, price, category, description, fecha_insercion FROM Productos ORDER BY id;"`

4. **KPIs**:  
   - `total = dtProductos.Rows.Count`  
   - `promGeneral = (dtProductos == null || dtProductos.Rows.Count == 0 ? 0.0 : Convert.ToDouble(dtProductos.Compute("AVG(price)", "")))`

5. **Escritura en Plantilla** (`.\Data\Plantilla.xlsx`, hoja `"Resumen"`):  
   - **A1** = `"Total de productos"`  
   - **B1** = `total`  
   - **A2** = `"Precio promedio general"`  
   - **B2** = `promGeneral`

6. **Resumen por categoría** → `dtResumenCat` (DataTable):  
   `"SELECT category, AVG(price) AS avg_price, COUNT(*) AS total FROM Productos GROUP BY category ORDER BY category;"`  
   - **IF** `dtResumenCat != null && dtResumenCat.Rows.Count > 0` → **WriteRange** desde **A4** en `.\Data\Plantilla.xlsx`.

7. **Construir ruta del reporte del día**:  
```csharp
excelPathToday = Convert.ToString(Config["excelReport"])
                      .Replace("{yyyy-MM-dd}", DateTime.Now.ToString("yyyy-MM-dd"));
```

8. **Crear carpeta** (dir de `excelPathToday`) y **Copiar archivo**:  
   - Origen: `.\Data\Plantilla.xlsx`  
   - Destino: `excelPathToday` (sobrescribir = true)


---

## ✅ Cómo validar que todo funcionó

1) **JSON respaldo del día**  
`PROJECT_ROOT\Data\Productos_YYYY-MM-DD.json` existe y contiene el arreglo de productos.

2) **BD SQLite**  
`PROJECT_ROOT\Data\FakeStore.db` contiene la tabla **Productos**:  
- Ejecuta en DB Browser:
  - `SELECT COUNT(*) FROM Productos;`  
  - `SELECT category, ROUND(AVG(price),2), COUNT(*) FROM Productos GROUP BY category;`  
  Debe coincidir con el Excel.

3) **Reporte del día**  
`PROJECT_ROOT\Reportes\Reporte_YYYY-MM-DD.xlsx` con:  
- **Resumen**: A1/B1/A2/B2 completos + tabla desde A4.  
- **(Si implementaste Hoja Productos)**: lista de registros.


---

## 🧭 Trazabilidad:

- **HTTP GET + respaldo JSON + OneDrive(JSON)**  
  → `InitApplications.pix`: *Enviar solicitud HTTP (GET)*, *Ejecutar C#* (WriteAllText), *(opcional) PUT Graph*.

- **Extraer campos** (`id`, `title`, `price`, `category`, `description`)  
  → `GetTransactionItem.pix`: *Ejecutar C#* (JArray[Index]) y `ProcessTransactionItem.pix` (usa esos campos para INSERT).

- **BD (SQLite) + evitar duplicados + timestamp**  
  → `InitApplications.pix`: *CREATE TABLE IF NOT EXISTS Productos*.  
  → `ProcessTransactionItem.pix`: *COUNT* + *INSERT (?,?,?,?,?, datetime('now'))*.

- **Reporte Excel (local + OneDrive)**  
  → `ProcessTransactionItem.pix`: **KPIs** + **Resumen por categoría** en `.\Data\Plantilla.xlsx`; **Copiar** a `excelPathToday`; *(opcional) PUT Graph (Excel)*.

- **Formulario + evidencia**  
  → `ProcessTransactionItem.pix` (o subflujo web): **Chrome**, **llenar campos**, **adjuntar `excelPathToday`**, **enviar**, **captura** a `Evidencias\formulario_confirmacion.png`.

---

## 📝 Nota final de buena práctica

- Mantén **todas las rutas** y **URLs** en `Data\Config.xlsx` (para que no se pierdan en las actividades).   

