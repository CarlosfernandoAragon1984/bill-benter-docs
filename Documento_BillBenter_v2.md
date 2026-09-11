# 📘 Documento técnico completo — Sistema `bill_benter_v2`

**Versión**: 1.0
**Fecha**: 2026-09-11
**Motor**: SQL Server 2019+ (compatibility level 150)
**Objetivo**: Base de datos para predecir resultados de carreras de caballos y detectar *value bets*.

---

## Índice

1. [Visión general del sistema](#1-visión-general)
2. [Diccionario de datos](#2-diccionario-de-datos)
3. [Vistas](#3-vistas)
4. [Funciones](#4-funciones)
5. [Stored Procedures](#5-stored-procedures)
6. [Flujo del scraper (Selenium .NET 8)](#6-flujo-del-scraper)
7. [Consultas SQL para el scraper](#7-consultas-sql-para-el-scraper)
8. [Ejemplo C# Selenium .NET 8](#8-ejemplo-c-selenium)
9. [Checklist de operación](#9-checklist-de-operación)

---

# 1. VISIÓN GENERAL

## 1.1 ¿Qué hace el sistema?

`bill_benter_v2` es una base de datos para **predecir resultados de carreras de caballos** y detectar **value bets** (apuestas con valor esperado positivo).

## 1.2 Flujo general

```
┌──────────────────┐
│  Web de turf     │
└────────┬─────────┘
         │ Selenium .NET 8 (scraping)
         ▼
┌──────────────────────────────────────┐
│  bill_benter_v2                      │
│  ┌────────────────────────────────┐  │
│  │ 1. Catálogos (caballos, etc.)  │  │
│  │ 2. Carreras + Participaciones  │  │
│  │ 3. Resultados                  │  │
│  │ 4. Variables_Calculadas  ◄─────┼──┼── (SP recalcula)
│  │ 5. Modelo_Coeficientes         │  │
│  └────────────────────────────────┘  │
└──────────────────┬───────────────────┘
                   │
                   ▼
         ┌────────────────────┐
         │ DetectarValueBets  │
         └────────────────────┘
```

## 1.3 Orden de carga de datos

**CRÍTICO**: respetar este orden por las Foreign Keys.

| # | Tabla | Depende de |
|---|---|---|
| 1 | `Hipodromos` | — |
| 2 | `Caballos` | — |
| 3 | `Jinetes` | — |
| 4 | `Entrenadores` | — |
| 5 | `Carreras` | `Hipodromos` |
| 6 | `Participaciones` | `Caballos`, `Carreras`, `Jinetes`, `Entrenadores` |
| 7 | `Resultados_Carreras` | `Participaciones` |
| 8 | `Variables_Calculadas` | `Participaciones` (vía SP) |
| 9 | `Modelo_Coeficientes` | — (entrenados en Python/R) |

---

# 2. DICCIONARIO DE DATOS

## 2.1 `dbo.Hipodromos`

Representa cada **hipódromo** (lugar físico donde se corren las carreras).

| Columna | Tipo | Null | Descripción | Ejemplo |
|---|---|---|---|---|
| `HipodromoID` | INT IDENTITY | NO | PK. Se autogenera. | 1 |
| `Nombre` | NVARCHAR(150) | NO | Nombre oficial. **Único.** | Hipódromo Argentino de Palermo |
| `Ciudad` | NVARCHAR(100) | SÍ | Ciudad | Buenos Aires |
| `Pais` | NVARCHAR(50) | SÍ | País | Argentina |
| `LongitudRecta` | INT | SÍ | Metros de la recta principal | 600 |
| `TipoSuperficie` | NVARCHAR(20) | SÍ | Superficie principal | Dirt / Césped / Arena |
| `Curvatura` | NVARCHAR(20) | SÍ | Sentido de las curvas | Izquierda / Derecha |
| `Altitud` | INT | SÍ | Metros sobre nivel del mar | 10 |
| `FactorTiempo` | DECIMAL(5,4) | SÍ | Multiplicador para tiempos | 1.0000 |
| `EfectoLocal` | BIT | SÍ | Si favorece caballos locales | 0 |
| `FechaInauguracion` | DATE | SÍ | Fecha de apertura | 1876-05-07 |
| `Activo` | BIT | NO | Si sigue operando. Default 1. | 1 |

**Regla**: se carga **una sola vez** al inicio. Si aparece un hipódromo nuevo, se hace INSERT.

---

## 2.2 `dbo.Caballos`

Cada **caballo** que corre.

| Columna | Tipo | Null | Descripción |
|---|---|---|---|
| `CaballoID` | INT IDENTITY | NO | PK. Autogenerado. |
| `Nombre` | NVARCHAR(100) | NO | Nombre del caballo |
| `FechaNacimiento` | DATE | SÍ | Cuándo nació |
| `Sexo` | CHAR(1) | SÍ | `M`=Macho, `H`=Hembra, `C`=Castrado |
| `Color` | NVARCHAR(50) | SÍ | Zaino, Alazán, Tordillo, etc. |
| `Criador` | NVARCHAR(100) | SÍ | Quién lo crió |
| `Propietario` | NVARCHAR(100) | SÍ | Dueño actual |

**Regla**: antes de insertar un caballo, **buscar por nombre**. Si existe, usar el ID. Si no, insertar.

---

## 2.3 `dbo.Jinetes`

Cada **jinete** (quien monta el caballo).

| Columna | Tipo | Null | Descripción |
|---|---|---|---|
| `JineteID` | INT IDENTITY | NO | PK |
| `Nombre` | NVARCHAR(100) | NO | Nombre completo |
| `PesoBase` | DECIMAL(5,2) | SÍ | Peso mínimo (kg). CHECK: 40-90 |
| `FechaInicio` | DATE | SÍ | Cuándo empezó a correr |
| `Activo` | BIT | NO | Default 1 |

**Regla**: buscar por nombre antes de insertar.

---

## 2.4 `dbo.Entrenadores`

Cada **entrenador** (prepara el caballo).

| Columna | Tipo | Null | Descripción |
|---|---|---|---|
| `EntrenadorID` | INT IDENTITY | NO | PK |
| `Nombre` | NVARCHAR(100) | NO | Nombre |
| `Establo` | NVARCHAR(100) | SÍ | Nombre del establo |
| `FechaInicio` | DATE | SÍ | Cuándo empezó |
| `Activo` | BIT | NO | Default 1 |

⚠️ **IMPORTANTE**: nunca insertar textos de condiciones de carrera acá. Solo **personas**.

---

## 2.5 `dbo.Carreras`

Cada **carrera** individual.

| Columna | Tipo | Null | Descripción | Ejemplo |
|---|---|---|---|---|
| `CarreraID` | INT IDENTITY | NO | PK | 1 |
| `Fecha` | DATE | NO | Fecha de la carrera | 2026-08-31 |
| `HipodromoID` | INT | NO | FK a Hipodromos | 1 |
| `Distancia` | INT | SÍ | Metros | 1800 |
| `Superficie` | NVARCHAR(20) | SÍ | Dirt / Césped / Arena | Arena |
| `CondicionPista` | NVARCHAR(20) | SÍ | NORMAL / HUMEDA / PESADA | NORMAL |
| `ClaseCarrera` | NVARCHAR(255) | SÍ | Condiciones (texto largo) | Yeguas de 5 años... |
| `PremioTotal` | DECIMAL(12,2) | SÍ | Premio total en $ | 9350000.00 |
| `NombreCarrera` | NVARCHAR(150) | SÍ | Nombre del premio | GIRL'S DAY |

**Regla**: antes de insertar, verificar que no exista (misma fecha + hipódromo + distancia + nombre).

---

## 2.6 `dbo.Participaciones`

**Tabla central**. Cada **inscripción** de un caballo en una carrera. Incluye `Odds`.

| Columna | Tipo | Null | Descripción |
|---|---|---|---|
| `ParticipacionID` | INT IDENTITY | NO | PK |
| `CaballoID` | INT | NO | FK Caballos |
| `CarreraID` | INT | NO | FK Carreras |
| `JineteID` | INT | SÍ | FK Jinetes |
| `EntrenadorID` | INT | SÍ | FK Entrenadores |
| `NumeroCaja` | INT | SÍ | Caja de partida (1-14) |
| `PesoAsignado` | DECIMAL(5,2) | SÍ | Peso que lleva (kg) |
| `Odds` | DECIMAL(10,2) | SÍ | Cuota final. CHECK: > 1 |

**Regla**: por cada carrera, insertar una fila por caballo. UNIQUE `(CaballoID, CarreraID)` evita duplicados.

---

## 2.7 `dbo.Resultados_Carreras`

Solo el **resultado deportivo**. NO duplica fecha, caballo, ni odds.

| Columna | Tipo | Null | Descripción |
|---|---|---|---|
| `ResultadoID` | INT IDENTITY | NO | PK |
| `ParticipacionID` | INT | NO | FK Participaciones. UNIQUE |
| `PosicionFinal` | INT | SÍ | 1-30 (NULL si retirado). CHECK |
| `TiempoFinal` | TIME(7) | SÍ | Tiempo del caballo |
| `DistanciaGanador` | DECIMAL(5,2) | SÍ | Cuerpos de distancia al ganador |
| `PesoActual` | DECIMAL(5,2) | SÍ | Peso real al momento de correr |
| `PesoDeclarado` | DECIMAL(5,2) | SÍ | Peso declarado |

**Regla**: insertar **después** de la carrera. Si un caballo se retiró, `PosicionFinal = NULL`.

---

## 2.8 `dbo.Variables_Calculadas`

Variables **predictoras** calculadas point-in-time (solo con datos previos a cada carrera). **NO se cargan con el scraper**: se calculan con el SP `sp_RecalcularVariablesPointInTime`.

| Columna | Descripción |
|---|---|
| `VariableID` | PK |
| `ParticipacionID` | FK, UNIQUE |
| `FechaCalculo` | Cuándo se calculó |
| `PorcentajeVictorias_180dias` | % de victorias en últimos 180 días |
| `PorcentajeVictorias_5ultimas` | % en últimas 5 carreras |
| `PorcentajeVictorias_2anos` | % en últimos 2 años |
| `PromedioPosicionesPonderado` | Promedio ponderado por recencia (decae exp.) |
| `NumeroCarreras_Carrera` | Cuántas carreras previas tiene |
| `Dias_UltimaCarrera` | Días desde la última. 9999 si es primera |
| `EsPrimeraCarrera` | 1 si no tiene historial |
| `PorcentajeVictorias_Jinete_365dias` | Forma del jinete |
| `NumeroVictorias_Jinete_365dias` | Victorias del jinete |
| `NumeroCarreras_Jinete_Carrera` | Carreras del jinete |
| `NumeroCarreras_Entrenador_Carrera` | Carreras del entrenador |
| `PorcentajeVictorias_Entrenador_365dias` | Forma del entrenador |
| `FactorTiempoHipodromo` | Factor del hipódromo |
| `FactorRecencia` | Decaimiento exponencial |
| `EfectoCaja` | Efecto de la caja (si hay datos) |
| `AdaptacionHipodromo` | Adaptación del caballo al hipódromo |
| `PesoActual` | Peso del caballo (de Participaciones) |

---

## 2.9 `dbo.Modelo_Coeficientes`

Los **β** del modelo. Se cargan una vez, después de entrenar en Python/R.

| Columna | Descripción |
|---|---|
| `CoeficienteID` | PK |
| `VariableNombre` | Nombre exacto (ej: `INTERCEPTO`, `PORCENTAJE_VICTORIAS_180DIAS`) |
| `Beta` | Valor del coeficiente |
| `DesviacionEstandar` | Error estándar |
| `PValor` | Significancia |
| `FechaEntrenamiento` | Cuándo se entrenó |
| `VersionModelo` | 1, 2, 3... |
| `Activo` | 1 si está en uso |

**Regla**: UNIQUE `(VariableNombre, VersionModelo)`. Para actualizar el modelo, insertás una nueva versión y ponés `Activo = 0` en la anterior.

---

## 2.10 `dbo.Estadisticas_Hipodromo` + `dbo.EfectosCaja`

Estadísticas agregadas por hipódromo y fecha. `EfectosCaja` guarda una fila por (estadística, caja).

**`Estadisticas_Hipodromo`**

| Columna | Descripción |
|---|---|
| `EstadisticaID` | PK |
| `HipodromoID` | FK |
| `Fecha` | Fecha |
| `VelocidadBase` | Velocidad base |
| `VariacionVelocidad` | Variación |
| `DiasLluvia_Ultimos7` | Días con lluvia |
| `TemperaturaPromedio` | Temperatura |

**`EfectosCaja`**

| Columna | Descripción |
|---|---|
| `EfectoCajaID` | PK |
| `EstadisticaID` | FK |
| `NumeroCaja` | 1-14 |
| `Valor` | Efecto de esa caja |

---

# 3. VISTAS

## 3.1 `dbo.Vista_CarrerasCompleta`

**Qué devuelve**: cada carrera con todos los datos del hipódromo y condición de pista.

**Cuándo usarla**: para dashboards, reportes, o cuando el scraper necesita saber qué datos ya tiene del hipódromo.

```sql
SELECT * FROM dbo.Vista_CarrerasCompleta 
WHERE Fecha = '2026-08-31';
```

## 3.2 `dbo.Vista_DatosCarrera`

**Qué devuelve**: cada participación con datos del caballo, jinete, entrenador, y **forma reciente point-in-time** (último año y 180 días).

**Cuándo usarla**: para análisis exploratorio y para verificar que las variables se calcularon bien.

```sql
SELECT * FROM dbo.Vista_DatosCarrera 
WHERE CarreraID = 1;
```

⚠️ **Performance**: esta vista tiene subconsultas correlacionadas. **Evitá usarla con `SELECT *` sobre toda la tabla**. Siempre filtrá por `CarreraID` o `ParticipacionID`.

---

# 4. FUNCIONES

## 4.1 `dbo.CalcularPonderacionRecencia(@DiasDiferencia)`

Devuelve un factor 0-1 que **decae exponencialmente**: 1.0 hoy, 0.5 a los 120 días.

**Cuándo usarla**: dentro de otros cálculos para ponderar datos por antigüedad.

```sql
SELECT dbo.CalcularPonderacionRecencia(30);  -- ~0.84
```

## 4.2 `dbo.CalcularAdaptacionHipodromo(@CaballoID, @HipodromoID, @Fecha)`

Devuelve **cuánto le gusta a un caballo un hipódromo** (0-1). 0.5 = neutro.

```sql
SELECT dbo.CalcularAdaptacionHipodromo(545, 1, '2026-09-11');
```

## 4.3 `dbo.ObtenerEfectoCaja(@HipodromoID, @NumeroCaja, @Fecha)`

Devuelve el **efecto de largar desde cierta caja** en un hipódromo.

⚠️ **Por ahora devuelve 0** porque `Estadisticas_Hipodromo` está vacía.

## 4.4 `dbo.CalcularScoreCaballo(@ParticipacionID, @VersionModelo)`

Devuelve `EXP(score)` del caballo en una participación. Es la **base del modelo**.

**Cuándo usarla**: internamente, dentro de `DetectarValueBets`. Vos no la llamás directamente.

## 4.5 `dbo.DetectarValueBets(@CarreraID, @VersionModelo)`

**LA función clave**. Devuelve una tabla con:

| Columna | Qué es |
|---|---|
| `Caballo` | Nombre |
| `ProbModeloPct` | Probabilidad según el modelo |
| `ProbMercadoPct` | Probabilidad implícita del mercado (normalizada) |
| `Odds` | Cuota |
| `EdgePct` | Diferencia modelo - mercado |
| `Decision` | APOSTAR / APOSTAR BAJO / NO APOSTAR |
| `KellyPct` | % de banca a apostar (Kelly fraccional 1/4) |

**Cuándo usarla**: cada vez que quieras evaluar una carrera para apostar.

```sql
SELECT * FROM dbo.DetectarValueBets(1, 1)
ORDER BY EdgePct DESC;
```

---

# 5. STORED PROCEDURES

## 5.1 `dbo.sp_RecalcularVariablesPointInTime(@LimpiarPrimero)`

**Qué hace**: recorre todas las participaciones y calcula `Variables_Calculadas` usando **solo datos previos** a cada carrera.

**Cuándo ejecutarlo**:
- Después de cargar resultados nuevos.
- Después de cargar muchas carreras nuevas.
- Una vez al día (por ejemplo, después del scraper).

**Parámetros**:
- `@LimpiarPrimero = 1`: borra y recalcula todo (recomendado).
- `@LimpiarPrimero = 0`: agrega sin borrar (cuidado con UNIQUE).

```sql
EXEC dbo.sp_RecalcularVariablesPointInTime @LimpiarPrimero = 1;
```

---

# 6. FLUJO DEL SCRAPER

## 6.1 ¿Qué tenés que scrapear?

| Dato | Cuándo | Frecuencia |
|---|---|---|
| **Programa de carreras** (futuras) | Día previo | Diario |
| **Resultados** (carreras ya corridas) | Después de cada reunión | Diario |
| **Odds finales** | Justo antes de la carrera | Por carrera |
| **Estadísticas de hipódromo** | Ocasional | Semanal/mensual |

## 6.2 Flujo paso a paso

```
┌─────────────────────────────────────────────────┐
│ 1. Scrapear programa del día siguiente          │
│    - Hipódromo, carreras, caballos, jinetes,    │
│      entrenadores, cajas, pesos                 │
└────────────────────┬────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────┐
│ 2. Insertar/actualizar en BD                    │
│    - Hipodromos (si es nuevo)                   │
│    - Caballos / Jinetes / Entrenadores          │
│    - Carreras                                   │
│    - Participaciones                            │
└────────────────────┬────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────┐
│ 3. Scrapear odds (justo antes de la carrera)    │
│    - Actualizar Participaciones.Odds            │
└────────────────────┬────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────┐
│ 4. Scrapear resultados (después de la reunión)  │
│    - Insertar Resultados_Carreras               │
└────────────────────┬────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────┐
│ 5. Ejecutar SP de recálculo de variables        │
│    EXEC sp_RecalcularVariablesPointInTime 1     │
└────────────────────┬────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────┐
│ 6. (Opcional) Ejecutar DetectarValueBets        │
│    para las carreras de mañana                  │
└─────────────────────────────────────────────────┘
```

---

# 7. CONSULTAS SQL PARA EL SCRAPER

## 7.1 Insertar hipódromo (si no existe)

```sql
IF NOT EXISTS (SELECT 1 FROM dbo.Hipodromos WHERE Nombre = @Nombre)
BEGIN
    INSERT INTO dbo.Hipodromos (Nombre, Ciudad, Pais, LongitudRecta, TipoSuperficie, Curvatura, Altitud, FactorTiempo, EfectoLocal, FechaInauguracion, Activo)
    VALUES (@Nombre, @Ciudad, @Pais, @LongitudRecta, @TipoSuperficie, @Curvatura, @Altitud, @FactorTiempo, @EfectoLocal, @FechaInauguracion, 1);
END
SELECT HipodromoID FROM dbo.Hipodromos WHERE Nombre = @Nombre;
```

## 7.2 Insertar o recuperar caballo

```sql
DECLARE @CaballoID INT;

SELECT @CaballoID = CaballoID FROM dbo.Caballos WHERE Nombre = @Nombre;

IF @CaballoID IS NULL
BEGIN
    INSERT INTO dbo.Caballos (Nombre, FechaNacimiento, Sexo, Color, Criador, Propietario)
    VALUES (@Nombre, @FechaNac, @Sexo, @Color, @Criador, @Propietario);
    SET @CaballoID = SCOPE_IDENTITY();
END
ELSE
BEGIN
    UPDATE dbo.Caballos
    SET FechaNacimiento = ISNULL(@FechaNac, FechaNacimiento),
        Sexo = ISNULL(@Sexo, Sexo),
        Color = ISNULL(@Color, Color),
        Criador = ISNULL(@Criador, Criador),
        Propietario = ISNULL(@Propietario, Propietario)
    WHERE CaballoID = @CaballoID;
END

SELECT @CaballoID AS CaballoID;
```

## 7.3 Insertar o recuperar jinete

```sql
DECLARE @JineteID INT;
SELECT @JineteID = JineteID FROM dbo.Jinetes WHERE Nombre = @Nombre;

IF @JineteID IS NULL
BEGIN
    INSERT INTO dbo.Jinetes (Nombre, PesoBase, FechaInicio, Activo)
    VALUES (@Nombre, @PesoBase, @FechaInicio, 1);
    SET @JineteID = SCOPE_IDENTITY();
END
SELECT @JineteID AS JineteID;
```

## 7.4 Insertar o recuperar entrenador

Idem jinete, con `Entrenadores`.

## 7.5 Insertar carrera

```sql
DECLARE @CarreraID INT;

SELECT @CarreraID = CarreraID FROM dbo.Carreras
WHERE Fecha = @Fecha AND HipodromoID = @HipodromoID 
  AND Distancia = @Distancia AND ISNULL(NombreCarrera,'') = ISNULL(@NombreCarrera,'');

IF @CarreraID IS NULL
BEGIN
    INSERT INTO dbo.Carreras (Fecha, HipodromoID, Distancia, Superficie, CondicionPista, ClaseCarrera, PremioTotal, NombreCarrera)
    VALUES (@Fecha, @HipodromoID, @Distancia, @Superficie, @CondicionPista, @ClaseCarrera, @PremioTotal, @NombreCarrera);
    SET @CarreraID = SCOPE_IDENTITY();
END

SELECT @CarreraID AS CarreraID;
```

## 7.6 Insertar participación

```sql
DECLARE @ParticipacionID INT;

SELECT @ParticipacionID = ParticipacionID FROM dbo.Participaciones
WHERE CaballoID = @CaballoID AND CarreraID = @CarreraID;

IF @ParticipacionID IS NULL
BEGIN
    INSERT INTO dbo.Participaciones (CaballoID, CarreraID, JineteID, EntrenadorID, NumeroCaja, PesoAsignado, Odds)
    VALUES (@CaballoID, @CarreraID, @JineteID, @EntrenadorID, @NumeroCaja, @PesoAsignado, @Odds);
    SET @ParticipacionID = SCOPE_IDENTITY();
END
ELSE
BEGIN
    UPDATE dbo.Participaciones
    SET JineteID = ISNULL(@JineteID, JineteID),
        EntrenadorID = ISNULL(@EntrenadorID, EntrenadorID),
        NumeroCaja = ISNULL(@NumeroCaja, NumeroCaja),
        PesoAsignado = ISNULL(@PesoAsignado, PesoAsignado),
        Odds = ISNULL(@Odds, Odds)
    WHERE ParticipacionID = @ParticipacionID;
END

SELECT @ParticipacionID AS ParticipacionID;
```

## 7.7 Actualizar odds (antes de la carrera)

```sql
UPDATE dbo.Participaciones
SET Odds = @Odds
WHERE ParticipacionID = @ParticipacionID;
```

## 7.8 Insertar resultado

```sql
MERGE dbo.Resultados_Carreras AS target
USING (SELECT @ParticipacionID AS ParticipacionID) AS source
ON target.ParticipacionID = source.ParticipacionID
WHEN MATCHED THEN
    UPDATE SET 
        PosicionFinal = @PosicionFinal,
        TiempoFinal = @TiempoFinal,
        DistanciaGanador = @DistanciaGanador,
        PesoActual = @PesoActual,
        PesoDeclarado = @PesoDeclarado
WHEN NOT MATCHED THEN
    INSERT (ParticipacionID, PosicionFinal, TiempoFinal, DistanciaGanador, PesoActual, PesoDeclarado)
    VALUES (@ParticipacionID, @PosicionFinal, @TiempoFinal, @DistanciaGanador, @PesoActual, @PesoDeclarado);
```

## 7.9 Ejecutar recálculo de variables (al final del día)

```sql
EXEC dbo.sp_RecalcularVariablesPointInTime @LimpiarPrimero = 1;
```

## 7.10 Consultar value bets para las carreras de mañana

```sql
DECLARE @Manana DATE = DATEADD(day, 1, CAST(GETDATE() AS DATE));

SELECT 
    c.CarreraID,
    c.NombreCarrera,
    c.Fecha,
    h.Nombre AS Hipodromo,
    vb.Caballo,
    vb.Odds,
    vb.ProbModeloPct,
    vb.ProbMercadoPct,
    vb.EdgePct,
    vb.Decision,
    vb.KellyPct
FROM dbo.Carreras c
JOIN dbo.Hipodromos h ON c.HipodromoID = h.HipodromoID
CROSS APPLY dbo.DetectarValueBets(c.CarreraID, 1) vb
WHERE c.Fecha = @Manana
  AND vb.Decision = 'APOSTAR'
ORDER BY c.CarreraID, vb.EdgePct DESC;
```

---

# 8. EJEMPLO C# SELENIUM .NET 8

## 8.1 Estructura del proyecto

```
BillBenter.Scraper/
├── Program.cs
├── appsettings.json
├── Scrapers/
│   ├── ProgramaScraper.cs
│   ├── OddsScraper.cs
│   └── ResultadosScraper.cs
├── Data/
│   ├── Database.cs
│   └── Repositories/
│       ├── HipodromoRepository.cs
│       ├── CaballoRepository.cs
│       ├── JineteRepository.cs
│       ├── EntrenadorRepository.cs
│       ├── CarreraRepository.cs
│       ├── ParticipacionRepository.cs
│       └── ResultadoRepository.cs
└── Models/
    ├── CarreraScraped.cs
    ├── ParticipacionScraped.cs
    └── ResultadoScraped.cs
```

## 8.2 `appsettings.json`

```json
{
  "ConnectionStrings": {
    "BillBenter": "Server=localhost;Database=bill_benter_v2;Trusted_Connection=True;TrustServerCertificate=True;"
  },
  "Scraper": {
    "BaseUrl": "https://www.turf.com.ar",
    "Headless": true,
    "TimeoutSeconds": 30
  }
}
```

## 8.3 `Program.cs`

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using BillBenter.Scraper.Scrapers;
using BillBenter.Scraper.Data;

var config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .Build();

var services = new ServiceCollection();
services.AddSingleton<IConfiguration>(config);
services.AddSingleton<Database>();
services.AddScoped<ProgramaScraper>();
services.AddScoped<OddsScraper>();
services.AddScoped<ResultadosScraper>();

var provider = services.BuildServiceProvider();

var modo = args.Length > 0 ? args[0] : "programa";

switch (modo)
{
    case "programa":
        await provider.GetRequiredService<ProgramaScraper>().EjecutarAsync();
        break;
    case "odds":
        await provider.GetRequiredService<OddsScraper>().EjecutarAsync();
        break;
    case "resultados":
        await provider.GetRequiredService<ResultadosScraper>().EjecutarAsync();
        break;
    default:
        Console.WriteLine("Modo no reconocido: programa | odds | resultados");
        break;
}
```

## 8.4 `Database.cs`

```csharp
using Microsoft.Data.SqlClient;
using Microsoft.Extensions.Configuration;

namespace BillBenter.Scraper.Data;

public class Database
{
    private readonly string _connectionString;

    public Database(IConfiguration config)
    {
        _connectionString = config.GetConnectionString("BillBenter")!;
    }

    public SqlConnection CrearConexion() => new SqlConnection(_connectionString);
}
```

## 8.5 `CaballoRepository.cs`

```csharp
using Microsoft.Data.SqlClient;
using BillBenter.Scraper.Data;

namespace BillBenter.Scraper.Data.Repositories;

public class CaballoRepository
{
    private readonly Database _db;
    public CaballoRepository(Database db) => _db = db;

    public int ObtenerOInsertar(string nombre, DateTime? fechaNac, string? sexo, 
                                string? color, string? criador, string? propietario)
    {
        using var conn = _db.CrearConexion();
        conn.Open();

        using var cmd = new SqlCommand(@"
            DECLARE @CaballoID INT;
            SELECT @CaballoID = CaballoID FROM dbo.Caballos WHERE Nombre = @Nombre;
            IF @CaballoID IS NULL
            BEGIN
                INSERT INTO dbo.Caballos (Nombre, FechaNacimiento, Sexo, Color, Criador, Propietario)
                VALUES (@Nombre, @FechaNac, @Sexo, @Color, @Criador, @Propietario);
                SET @CaballoID = SCOPE_IDENTITY();
            END
            ELSE
            BEGIN
                UPDATE dbo.Caballos
                SET FechaNacimiento = ISNULL(@FechaNac, FechaNacimiento),
                    Sexo = ISNULL(@Sexo, Sexo),
                    Color = ISNULL(@Color, Color),
                    Criador = ISNULL(@Criador, Criador),
                    Propietario = ISNULL(@Propietario, Propietario)
                WHERE CaballoID = @CaballoID;
            END
            SELECT @CaballoID;
        ", conn);

        cmd.Parameters.AddWithValue("@Nombre", nombre);
        cmd.Parameters.AddWithValue("@FechaNac", (object?)fechaNac ?? DBNull.Value);
        cmd.Parameters.AddWithValue("@Sexo", (object?)sexo ?? DBNull.Value);
        cmd.Parameters.AddWithValue("@Color", (object?)color ?? DBNull.Value);
        cmd.Parameters.AddWithValue("@Criador", (object?)criador ?? DBNull.Value);
        cmd.Parameters.AddWithValue("@Propietario", (object?)propietario ?? DBNull.Value);

        return (int)cmd.ExecuteScalar()!;
    }
}
```

## 8.6 `ProgramaScraper.cs` (esqueleto)

```csharp
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using OpenQA.Selenium.Support.UI;
using Microsoft.Extensions.Configuration;

namespace BillBenter.Scraper.Scrapers;

public class ProgramaScraper
{
    private readonly IConfiguration _config;

    public ProgramaScraper(IConfiguration config) => _config = config;

    public async Task EjecutarAsync()
    {
        var options = new ChromeOptions();
        if (_config.GetValue<bool>("Scraper:Headless"))
            options.AddArgument("--headless=new");
        options.AddArgument("--no-sandbox");
        options.AddArgument("--disable-dev-shm-usage");

        using var driver = new ChromeDriver(options);
        driver.Manage().Timeouts().ImplicitWait = TimeSpan.FromSeconds(10);

        try
        {
            var baseUrl = _config["Scraper:BaseUrl"];
            driver.Navigate().GoToUrl($"{baseUrl}/programa");

            var wait = new WebDriverWait(driver, TimeSpan.FromSeconds(20));
            wait.Until(d => d.FindElement(By.CssSelector(".carrera-row")));

            var carreras = driver.FindElements(By.CssSelector(".carrera-row"));
            foreach (var carrera in carreras)
            {
                // 1. Extraer datos de la carrera
                // 2. Extraer participaciones
                // 3. Insertar en BD con repositorios
            }
        }
        finally
        {
            driver.Quit();
        }
    }
}
```

## 8.7 Recomendaciones Selenium

1. **Usar `WebDriverWait`** en lugar de `Thread.Sleep`.

   ```csharp
   var wait = new WebDriverWait(driver, TimeSpan.FromSeconds(20));
   wait.Until(d => d.FindElement(By.CssSelector(".carrera")));
   ```

2. **Guardar HTML crudo** por si el scraping falla, para reprocesar después.

   ```csharp
   File.WriteAllText($"logs/{DateTime.Now:yyyyMMdd_HHmmss}.html", driver.PageSource);
   ```

3. **Usar `Headless = false`** mientras desarrollás, para ver qué pasa.

4. **Rate limiting**: `await Task.Delay(2000)` entre páginas para no tumbar el sitio.

5. **Retry pattern con Polly**:

   ```bash
   dotnet add package Polly
   ```

   ```csharp
   var retry = Policy
       .Handle<WebDriverException>()
       .WaitAndRetryAsync(3, i => TimeSpan.FromSeconds(Math.Pow(2, i)));
   await retry.ExecuteAsync(async () => { /* scraping */ });
   ```

6. **Transacciones**: agrupar los INSERT por carrera en una transacción.

   ```csharp
   using var tx = conn.BeginTransaction();
   // ... inserts ...
   tx.Commit();
   ```

7. **Selectores robustos**: preferir `By.CssSelector` antes que `By.XPath` (más rápido). Y si el HTML cambia seguido, usar atributos estables (`data-*`, `id`) en lugar de clases CSS.

8. **Manejo de errores por carrera**: si una carrera falla, no abortar todo el scraping. Loguear el error y seguir con la siguiente.

   ```csharp
   foreach (var carrera in carreras)
   {
       try
       {
           // procesar carrera
       }
       catch (Exception ex)
       {
           Console.WriteLine($"Error en carrera: {ex.Message}");
           File.AppendAllText("logs/errores.log",
               $"{DateTime.Now}: {ex}\n");
       }
   }
   ```

---

# 9. CHECKLIST DE OPERACIÓN

## 9.1 Setup inicial (una sola vez)

- [ ] Crear base `bill_benter_v2` con el DDL.
- [ ] Verificar que todas las tablas existan.
- [ ] Insertar `Modelo_Coeficientes` versión 1 (entrenados con Python).
- [ ] Configurar `appsettings.json` con la cadena de conexión.

## 9.2 Rutina diaria

- [ ] **Mañana**: ejecutar `ProgramaScraper` → carga programa del día.
- [ ] **Antes de cada carrera**: ejecutar `OddsScraper` → actualiza odds.
- [ ] **Después de la reunión**: ejecutar `ResultadosScraper` → carga resultados.
- [ ] **Al final del día**: ejecutar SP de variables:

  ```sql
  EXEC dbo.sp_RecalcularVariablesPointInTime @LimpiarPrimero = 1;
  ```

- [ ] **Verificar value bets** para mañana:

  ```sql
  SELECT * FROM dbo.DetectarValueBets(...) WHERE Decision = 'APOSTAR';
  ```

## 9.3 Rutina mensual

- [ ] Exportar datos a CSV.
- [ ] Reentrenar el modelo en Python.
- [ ] Insertar nueva `VersionModelo` en `Modelo_Coeficientes`.
- [ ] Marcar la anterior como `Activo = 0`.
- [ ] Validar con backtesting.

## 9.4 Troubleshooting

| Problema | Causa | Solución |
|---|---|---|
| `PK_Participaciones` violación | Duplicado | Usar `IF NOT EXISTS` antes del INSERT |
| `FK_Participaciones_Caballo` violación | CaballoID no existe | Insertar caballo primero |
| `CK_Resultados_Posicion` violación | PosicionFinal fuera de 1-30 | Usar `CASE WHEN BETWEEN 1 AND 30 THEN ... ELSE NULL` |
| Scraper lento | `Thread.Sleep` | Usar `WebDriverWait` |
| Scraper falla | Sitio cambió | Guardar HTML y revisar selectores |
| Value bets todos APOSTAR | Coeficientes malos | Reentrenar |

---

# 10. GLOSARIO

| Término | Significado |
|---|---|
| **Value bet** | Apuesta cuyo valor esperado es positivo (probabilidad real > probabilidad implícita del mercado) |
| **Odds** | Cuota que paga la casa de apuestas |
| **Point-in-time** | Cálculo de variables usando **solo** información disponible **antes** del evento |
| **Data leakage** | Contaminación de variables con información del futuro. Rompe el modelo. |
| **Kelly criterion** | Fórmula para calcular el % óptimo de banca a apostar según el edge |
| **Kelly fraccional** | Kelly multiplicado por un factor (< 1) para ser más conservador |
| **Overround** | Margen que se queda la casa de apuestas. Hace que Σ(1/Odds) > 1 |
| **Probabilidad implícita** | 1/Odds. Es lo que el mercado cree que vale el caballo |
| **Takeout** | Porcentaje que el hipódromo se queda de cada apuesta (~20-25%) |
| **Backtesting** | Simular el modelo en datos históricos para medir su rendimiento |
| **Walk-forward** | Backtesting que respeta el orden temporal (entrenar con pasado, evaluar con futuro) |
| **Speed rating** | Puntuación de velocidad de un caballo en una carrera |
| **Caja (post position)** | Número de partida del caballo. Algunas cajas favorecen en ciertos hipódromos |
| **Hándicap** | Carrera donde se asignan pesos para igualar chances |
| **Yunta** | Caballo castrado mayor de 4 años |

---

# 11. MÉTRICAS DEL MODELO

Cuando reentrenés el modelo, estas son las métricas que tenés que mirar:

| Métrica | Qué mide | Valor ideal |
|---|---|---|
| **Log Loss** | Qué tan calibradas están las probabilidades | Cuanto más bajo, mejor |
| **Brier Score** | Error cuadrático medio de las probabilidades | Cuanto más bajo, mejor |
| **AUC** | Capacidad de discriminar ganador vs. perdedor | > 0.65 es aceptable en turf |
| **Calibración** | Si dice 20%, ¿gana 20%? | Debería ser una línea recta |
| **ROI** | Retorno sobre inversión simulada | > 0% (idealmente > 20% para cubrir takeout) |
| **Yield** | Ganancia por apuesta | > 0 |
| **Strike Rate** | % de aciertos | 15-30% típico |
| **Max Drawdown** | Peor caída de la banca | Cuanto más chico, mejor |

## 11.1 Fórmulas

**Log Loss**:

```
LogLoss = -1/N * Σ [y_i * log(p_i) + (1 - y_i) * log(1 - p_i)]
```

**ROI**:

```
ROI = (GananciaTotal - InversiónTotal) / InversiónTotal * 100
```

**Yield**:

```
Yield = GananciaTotal / NúmeroApuestas
```

**Kelly fraccional (1/4)**:

```
f = ((p * odds) - 1) / (odds - 1) * 0.25
```

Donde:
- `p` = probabilidad real (según el modelo)
- `odds` = cuota decimal

---

# 12. ROADMAP

## Fase 1 (Completada)
- ✅ Esquema de base de datos
- ✅ Migración de datos base
- ✅ Cálculo de variables point-in-time
- ✅ Funciones de detección de value bets

## Fase 2 (En curso)
- 🔄 Conseguir datos históricos (1-2 años)
- 🔄 Reentrenar modelo con Python/R
- 🔄 Construir scraper Selenium .NET 8
- 🔄 Backtesting formal

## Fase 3 (Futuro)
- ⏳ Dashboard de monitoreo
- ⏳ Sistema de apuestas automatizado
- ⏳ Integración con APIs de casas de apuestas
- ⏳ Modelo de tamaño variable de apuesta (Kelly dinámico)

## Fase 4 (Optimización)
- ⏳ Feature engineering avanzado
- ⏳ Modelos de ensamble (XGBoost, LightGBM)
- ⏳ Redes neuronales para patrones ocultos
- ⏳ Procesamiento en tiempo real

---

# 13. REFERENCIAS

## 13.1 Bibliografía sobre betting

- **Benter, W.** (1994). *Computer Based Horse Race Handicapping and Wagering Systems: A Report*. En "Efficiency of Racetrack Betting Markets".
- **Ziemba, W. T.** (2008). *Sports and Stochastics: Risk and Uncertainty in Sports*.
- **Bolton, R. J., & Chapman, R. G.** (1986). *Searching for Positive Returns at the Track: A Multinomial Logit Model for Handicapping Horse Races*. Management Science.

## 13.2 Librerías recomendadas

**Python**:
- `scikit-learn`: modelos de ML clásicos
- `statsmodels`: regresión con p-valores e intervalos de confianza
- `pandas`: manipulación de datos
- `xgboost` / `lightgbm`: gradient boosting

**R**:
- `mlogit`: modelos logit multinomiales condicionales
- `survival`: modelos de Cox para time-to-event
- `tidyverse`: manipulación de datos

**.NET**:
- `Selenium.WebDriver`: automatización del navegador
- `Microsoft.Data.SqlClient`: conexión a SQL Server
- `Polly`: retry patterns
- `Serilog`: logging estructurado

## 13.3 Fuentes de datos de turf (Argentina)

- Hipódromo Argentino de Palermo: https://www.palermo.com.ar
- Hipódromo de San Isidro: https://www.hipodromosanisidro.com
- Hipódromo de La Plata: https://www.hipodromolaplata.gba.gob.ar
- Portales: Turf Argentina, De Turf, Hipódromos y Galopes

---

# 14. ANEXOS

## 14.1 Estructura de tablas (resumen visual)

```
Hipodromos
├── HipodromoID (PK)
├── Nombre
└── ...

Caballos                Jinetes                 Entrenadores
├── CaballoID (PK)      ├── JineteID (PK)       ├── EntrenadorID (PK)
├── Nombre              ├── Nombre              ├── Nombre
└── ...                 └── ...                 └── ...

                          Carreras
                          ├── CarreraID (PK)
                          ├── Fecha
                          ├── HipodromoID (FK)
                          └── ...

                          Participaciones (tabla central)
                          ├── ParticipacionID (PK)
                          ├── CaballoID (FK)
                          ├── CarreraID (FK)
                          ├── JineteID (FK)
                          ├── EntrenadorID (FK)
                          ├── NumeroCaja
                          ├── PesoAsignado
                          └── Odds

                          Resultados_Carreras
                          ├── ResultadoID (PK)
                          ├── ParticipacionID (FK, UNIQUE)
                          ├── PosicionFinal
                          ├── TiempoFinal
                          └── ...

                          Variables_Calculadas
                          ├── VariableID (PK)
                          ├── ParticipacionID (FK, UNIQUE)
                          ├── PorcentajeVictorias_180dias
                          └── ...

                          Modelo_Coeficientes
                          ├── CoeficienteID (PK)
                          ├── VariableNombre
                          ├── Beta
                          └── ...
```

## 14.2 Consultas útiles de monitoreo

### ¿Cuántas carreras hay por mes?

```sql
SELECT 
    YEAR(Fecha) AS Anio,
    MONTH(Fecha) AS Mes,
    COUNT(*) AS Carreras,
    COUNT(DISTINCT HipodromoID) AS Hipodromos
FROM dbo.Carreras
GROUP BY YEAR(Fecha), MONTH(Fecha)
ORDER BY Anio DESC, Mes DESC;
```

### ¿Cuántas participaciones tienen odds?

```sql
SELECT 
    COUNT(*) AS Total,
    SUM(CASE WHEN Odds IS NOT NULL THEN 1 ELSE 0 END) AS ConOdds,
    CAST(SUM(CASE WHEN Odds IS NOT NULL THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS DECIMAL(5,2)) AS PctConOdds
FROM dbo.Participaciones;
```

### Top 10 jinetes por victorias en el último año

```sql
SELECT TOP 10
    j.Nombre,
    COUNT(*) AS Carreras,
    SUM(CASE WHEN r.PosicionFinal = 1 THEN 1 ELSE 0 END) AS Victorias,
    CAST(SUM(CASE WHEN r.PosicionFinal = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS DECIMAL(5,2)) AS WinPct
FROM dbo.Participaciones p
JOIN dbo.Jinetes j ON p.JineteID = j.JineteID
JOIN dbo.Resultados_Carreras r ON r.ParticipacionID = p.ParticipacionID
JOIN dbo.Carreras c ON p.CarreraID = c.CarreraID
WHERE c.Fecha >= DATEADD(year, -1, GETDATE())
GROUP BY j.Nombre
ORDER BY Victorias DESC;
```

### Distribución de caballos por edad

```sql
SELECT 
    DATEDIFF(year, FechaNacimiento, GETDATE()) AS Edad,
    COUNT(*) AS Cantidad
FROM dbo.Caballos
WHERE FechaNacimiento IS NOT NULL
GROUP BY DATEDIFF(year, FechaNacimiento, GETDATE())
ORDER BY Edad;
```

## 14.3 Consultas de validación del sistema

### ¿El SP de variables está actualizado?

```sql
SELECT 
    MAX(FechaCalculo) AS UltimoCalculo,
    COUNT(*) AS TotalVariables,
    COUNT(DISTINCT ParticipacionID) AS ParticipacionesCubiertas
FROM dbo.Variables_Calculadas;
```

### ¿Cuántas variables tienen data?

```sql
SELECT 
    COUNT(*) AS Total,
    SUM(CASE WHEN PorcentajeVictorias_180dias IS NOT NULL THEN 1 ELSE 0 END) AS ConVict180,
    SUM(CASE WHEN Dias_UltimaCarrera < 9999 THEN 1 ELSE 0 END) AS ConHistorial,
    SUM(CASE WHEN EsPrimeraCarrera = 1 THEN 1 ELSE 0 END) AS PrimerasCarreras
FROM dbo.Variables_Calculadas;
```

### ¿La función DetectarValueBets funciona?

```sql
-- Buscar una carrera con odds cargadas
DECLARE @CarreraID INT = (
    SELECT TOP 1 CarreraID 
    FROM dbo.Participaciones 
    WHERE Odds IS NOT NULL 
    GROUP BY CarreraID 
    HAVING COUNT(*) >= 5
);

SELECT * FROM dbo.DetectarValueBets(@CarreraID, 1);
```

---

# FIN DEL DOCUMENTO

**Versión**: 1.0
**Última actualización**: 2026-09-11
**Mantenido por**: [Tu nombre]
**Contacto**: [Tu email]
