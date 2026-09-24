# 📘 Guía de uso — Bill Benter v2

Comandos y flujos para operar el sistema completo.

## Índice

1. [Setup inicial](#1-setup-inicial)
2. [Uso del scraper (C#)](#2-uso-del-scraper-c)
3. [Uso del ML pipeline (Python)](#3-uso-del-ml-pipeline-python)
4. [Predicción de una carrera](#4-predicción-de-una-carrera)
5. [Git / GitHub](#5-git--github)
6. [Troubleshooting](#6-troubleshooting)

---

## 1. Setup inicial

### Requisitos

| Herramienta | Versión mínima | Verificar |
|---|---|---|
| SQL Server | 2019+ | `sqlcmd -?` |
| .NET SDK | 8.0 | `dotnet --version` |
| Python | 3.12+ | `python --version` |
| ODBC Driver | 17 o 18 | `odbcad32.exe` |
| Git | 2.x | `git --version` |

### Instalar dependencias Python

```powershell
cd D:\webScraping\PalermoScraper\ml
pip install -r requirements.txt
```

### Verificar conexión

```powershell
python src\db.py
```

**Esperado:** `✅ Conectado. Caballos en BD: 19138`

---

## 2. Uso del scraper (C#)

### Estructura

```powershell
cd D:\webScraping\PalermoScraper
dotnet run --project Console -- <modo> [args]
```

### Modos

| Modo | Descripción | Duración |
|---|---|---|
| `calendario` | Lista reuniones del año | 5 seg |
| `reunion <id>` | Scrapea y muestra (no persiste) | 5 seg |
| `guardar-reunion <id>` | Scrapea y persiste | 10 seg |
| `completar-caballos [n]` | Enriquece caballos | variable |
| `historico <desde> <hasta>` | Batch de años | horas |
| `actualizar <anio>` | Solo reuniones nuevas | minutos |
| `recalcular [no]` | SP de variables | 13 seg |

### Comandos frecuentes

**Listar reuniones del año:**

```powershell
dotnet run --project Console -- calendario
```

**Cargar histórico completo (una vez):**

```powershell
dotnet run --project Console -- historico 2021 2026
```

**Actualizar (rutina diaria):**

```powershell
dotnet run --project Console -- actualizar 2026
dotnet run --project Console -- completar-caballos 100
dotnet run --project Console -- recalcular
```

---

## 3. Uso del ML pipeline (Python)

### Scripts disponibles

| Script | Qué hace | Duración |
|---|---|---|
| `db.py` | Test de conexión | 2 seg |
| `features.py` | Export del dataset | 10 seg |
| `modelo.py` | Entrena XGBoost | 2-3 min |
| `modelo_sin_odds.py` | Entrena sin Odds | 2-3 min |
| `calibrar.py` | Calibración isotonic | 30 seg |
| `benter.py` | Combinación Benter | 1 min |
| `backtest.py` | Backtest base | 30 seg |
| `backtest_calibrado.py` | Backtest calibrado | 30 seg |
| `backtest_benter.py` | Backtest Benter | 30 seg |
| `ver_metricas.py` | Ver métricas | 30 seg |

### Flujo completo de re-entrenamiento

```powershell
cd D:\webScraping\PalermoScraper\ml

python src\db.py
python src\features.py
python src\modelo.py
python src\modelo_sin_odds.py
python src\calibrar.py
python src\benter.py
python src\backtest.py
python src\backtest_benter.py
python src\ver_metricas.py
```

### Notebook de análisis

```powershell
cd D:\webScraping\PalermoScraper\ml
python -m jupyterlab
```

Abrir: http://localhost:8888/lab → `notebooks/analisis_bill_benter.ipynb`

---

## 4. Predicción de una carrera

### Opción A — Python (recomendado)

Crear `ml\src\predecir_carrera.py` (ver código abajo) y ejecutar:

```powershell
python src\predecir_carrera.py
```

**Código completo de `predecir_carrera.py`:** ver [Anexo A](#anexo-a--predecir_carrerapy).

### Opción B — Cargar carrera a mano en SQL

Si la carrera no está en la BD (carreras futuras):

**1. Insertar carrera:**

```sql
USE bill_benter_v2;

DECLARE @HipodromoID INT = (SELECT HipodromoID FROM dbo.Hipodromos WHERE Nombre LIKE '%Palermo%');

INSERT INTO dbo.Carreras (Fecha, HipodromoID, Distancia, Superficie, CondicionPista, ClaseCarrera, PremioTotal, NombreCarrera)
VALUES ('2026-09-24', @HipodromoID, 1400, 'Arena', 'NORMAL', 'Todo caballo 5 años y más', 5250000, 'SUGARRETA');

SELECT SCOPE_IDENTITY() AS CarreraID;  -- Anotar este ID
```

**2. Insertar cada caballo (participación):**

```sql
DECLARE @CarreraID INT = <ID_ANTERIOR>;
DECLARE @CaballoID INT;
DECLARE @JineteID INT;
DECLARE @EntrenadorID INT;

-- Caballo
SELECT @CaballoID = CaballoID FROM dbo.Caballos WHERE Nombre = 'PURA PANZA';
IF @CaballoID IS NULL
BEGIN
    INSERT INTO dbo.Caballos (Nombre) VALUES ('PURA PANZA');
    SET @CaballoID = SCOPE_IDENTITY();
END

-- Jinete
SELECT @JineteID = JineteID FROM dbo.Jinetes WHERE Nombre = 'Aguirre Walter A';
IF @JineteID IS NULL
BEGIN
    INSERT INTO dbo.Jinetes (Nombre) VALUES ('Aguirre Walter A');
    SET @JineteID = SCOPE_IDENTITY();
END

-- Entrenador
SELECT @EntrenadorID = EntrenadorID FROM dbo.Entrenadores WHERE Nombre = 'Anglat Roberto O';
IF @EntrenadorID IS NULL
BEGIN
    INSERT INTO dbo.Entrenadores (Nombre) VALUES ('Anglat Roberto O');
    SET @EntrenadorID = SCOPE_IDENTITY();
END

-- Participación
INSERT INTO dbo.Participaciones (CaballoID, CarreraID, JineteID, EntrenadorID, NumeroCaja, PesoAsignado, Odds)
VALUES (@CaballoID, @CarreraID, @JineteID, @EntrenadorID, 1, 57, 5.0);
```

**Repetir para cada caballo de la carrera.**

**3. Recalcular variables:**

```sql
EXEC dbo.sp_RecalcularVariablesPointInTime @LimpiarPrimero = 1;
```

**4. Predecir con Python:**

Cambiar `CARRERA_ID` en `predecir_carrera.py` y ejecutar.

---

## 5. Git / GitHub

### Subir cambios

```powershell
cd D:\webScraping\PalermoScraper   # o ml\
git add .
git commit -m "Descripción del cambio"
git push
```

### Repos

- **ML**: https://github.com/CarlosFernandoAragon1984/bill-benter-ml
- **Scraper**: https://github.com/CarlosFernandoAragon1984/bill-benter-scraper
- **Docs**: https://github.com/CarlosFernandoAragon1984/bill-benter-docs

---

## 6. Troubleshooting

| Problema | Causa | Solución |
|---|---|---|
| `ModuleNotFoundError: features` | CWD incorrecto | Reiniciar kernel o usar ruta absoluta |
| `pyodbc.Error: Driver not found` | Driver ODBC | Instalar ODBC Driver 17 o 18 |
| `remote: Repository not found` | Repo no existe en GitHub | Crearlo primero |
| `could not read Username` | Sin credenciales | Configurar PAT |
| SP tarda más de 2h | Subqueries correlacionadas | Usar versión optimizada |
| `Paste Unavailable` en Jupyter | Restricción del navegador | Usar `Ctrl+V` |

---

## Anexo A — `predecir_carrera.py`

```python
"""Predice una carrera específica usando el modelo Benter."""
import pandas as pd
import numpy as np
import xgboost as xgb
from sklearn.calibration import CalibratedClassifierCV
from sklearn.frozen import FrozenEstimator
from features import cargar_features

CARRERA_ID = 9288   # ← CAMBIAR

FEATURES = [
    'PorcentajeVictorias_5ultimas', 'PorcentajeVictorias_180dias',
    'PorcentajeVictorias_2anos', 'PromedioPosicionesPonderado',
    'NumeroCarreras_Carrera', 'Dias_UltimaCarrera', 'EsPrimeraCarrera',
    'PorcentajeVictorias_Jinete_365dias', 'NumeroVictorias_Jinete_365dias',
    'NumeroCarreras_Jinete_Carrera', 'NumeroCarreras_Entrenador_Carrera',
    'PorcentajeVictorias_Entrenador_365dias', 'FactorTiempoHipodromo',
    'FactorRecencia', 'EfectoCaja', 'AdaptacionHipodromo', 'PesoActual',
    'NumeroCaja', 'Distancia', 'HipodromoID', 'Odds',
]
TARGET = 'Gano'
W1 = 0.2225  # peso modelo
W2 = 0.7912  # peso mercado
EPS = 1e-6

def logit(p):
    p = np.clip(p, EPS, 1 - EPS)
    return np.log(p / (1 - p))

def sigmoid(x):
    return 1 / (1 + np.exp(-x))

print(f"📥 Cargando features para CarreraID={CARRERA_ID}...")
df = cargar_features()
df['PesoActual'] = df['PesoActual'].fillna(df['PesoActual'].median())

carrera = df[df['CarreraID'] == CARRERA_ID].copy()

if len(carrera) == 0:
    print(f"❌ No se encontró la carrera {CARRERA_ID}")
    exit(1)

print(f"   {len(carrera)} caballos")

val = df[(df['FechaCarrera'] > '2024-12-31') & (df['FechaCarrera'] <= '2025-12-31')]
model = xgb.XGBClassifier()
model.load_model('../models/xgb_v2.json')
cal = CalibratedClassifierCV(FrozenEstimator(model), method='isotonic')
cal.fit(val[FEATURES], val[TARGET])

carrera['ProbModelo'] = cal.predict_proba(carrera[FEATURES])[:, 1]
carrera['InvOdds'] = 1.0 / carrera['Odds']
suma = carrera['InvOdds'].sum()
carrera['ProbMercado'] = carrera['InvOdds'] / suma

logit_mod = logit(carrera['ProbModelo'].values)
logit_mkt = logit(carrera['ProbMercado'].values)
carrera['ProbBenter'] = sigmoid(W1 * logit_mod + W2 * logit_mkt)
carrera['ProbBenter'] = carrera['ProbBenter'] / carrera['ProbBenter'].sum()
carrera['Edge'] = carrera['ProbBenter'] - carrera['ProbMercado']

carrera = carrera.sort_values('Edge', ascending=False)

print(f"\n{'='*80}\n🏇 Carrera {CARRERA_ID}\n{'='*80}")
for _, row in carrera.iterrows():
    print(f"\nParticipacionID {int(row['ParticipacionID'])}")
    print(f"  Odds:        {row['Odds']:.2f}")
    print(f"  ProbModelo:  {row['ProbModelo']*100:.2f}%")
    print(f"  ProbMercado: {row['ProbMercado']*100:.2f}%")
    print(f"  ProbBenter:  {row['ProbBenter']*100:.2f}%")
    print(f"  Edge:        {row['Edge']*100:+.2f}%")
```

---

**Última actualización:** 2026-09-24
