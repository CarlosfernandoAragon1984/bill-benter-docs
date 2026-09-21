# Palermo Scraper

Scraper de datos del **Hipódromo Argentino de Palermo** para alimentar la base `bill_benter_v2`, usada para predecir resultados de carreras y detectar *value bets*.

---

## Tabla de contenidos

1. [Contexto](#1-contexto)
2. [Fuentes de datos](#2-fuentes-de-datos)
3. [Decisiones de diseño](#3-decisiones-de-diseño)
4. [Stack tecnológico](#4-stack-tecnológico)
5. [Estructura del repositorio](#5-estructura-del-repositorio)
6. [Entidades del dominio](#6-entidades-del-dominio)
7. [Value Objects](#7-value-objects)
8. [Interfaces](#8-interfaces)
9. [Mapeo HTML → Base de datos](#9-mapeo-html--base-de-datos)
10. [Estrategia de scraping](#10-estrategia-de-scraping)
11. [Configuración](#11-configuración)
12. [Cómo ejecutar](#12-cómo-ejecutar)
13. [Roadmap](#13-roadmap)
14. [Convenciones de código](#14-convenciones-de-código)

---

## 1. Contexto

El proyecto `bill_benter_v2` es una base de datos SQL Server para predecir resultados de carreras de caballos y detectar *value bets*. El modelo predictivo necesita datos históricos y actuales de:

- Carreras y sus condiciones.
- Caballos participantes con sus datos biográficos.
- Jinetes y entrenadores.
- Resultados con posiciones, tiempos y distancias.
- Odds (*pagarias*) finales.

Este repositorio implementa el **scraper** que extrae esos datos de la web pública del Hipódromo de Palermo y los persiste en `bill_benter_v2` de forma **idempotente** (se puede ejecutar múltiples veces sin duplicar datos).

**Alcance histórico**: 5 años hacia atrás por defecto (configurable). Se considera suficiente porque ningún caballo en actividad corre más de 5-6 años.

---

## 2. Fuentes de datos

El sitio `https://old.palermo.com.ar` es **100% server-side rendered**, por lo que no se requiere un navegador headless. Todo el scraping se hace con `HttpClient` + parsing de HTML.

### 2.1 Niveles de navegación

| Nivel | URL | Qué aporta | Frecuencia |
|---|---|---|---|
| 1. Calendario | `/es/turf/calendario-de-carreras/{año}` | Lista de reuniones del año (fecha + ID) | 1 vez por año |
| 2. Reunión completa | `/es/turf/ver-carreras/{verDiaId}` | **Toda la jornada**: carreras, participantes, resultados, odds | 1 vez por reunión |
| 3. Caballo (opcional) | `/es/turf/ver-caballo/{verCaballoId}` | Datos biográficos + historial completo | Bajo demanda |

**Decisión**: para el scraper diario se usan **solo los niveles 1 y 2**. El nivel 3 se reserva para enriquecer caballos que aparecen sin historial o para validaciones puntuales.

### 2.2 Estructura de `ver-carreras/{verDiaId}`

Cada carrera dentro de la página de reunión tiene **3 bloques**:

- **Bloque A** — Metadata de la carrera: fecha, hora, distancia, superficie, condición de pista, tiempo del ganador, clase de carrera, monto del premio.
- **Bloque B** — Tabla de participantes: posición final, número de caja, nombre del caballo, distancia al ganador, jockey, cuidador (entrenador), caballeriza, peso jockey/caballo, pagaria (odds).
- **Bloque C** — Datos del ganador: nombre, fecha de nacimiento, sexo, pelaje, padre, madre, abuelo materno, criador, caballeriza.

---

## 3. Decisiones de diseño

| Decisión | Justificación |
|---|---|
| **No usar Selenium** | El sitio es server-side rendered. `HttpClient` es más rápido, más liviano y menos frágil. |
| **Usar `ver-carreras/{id}` en lugar de `ver-carrera/{id}`** | Una sola petición devuelve toda la jornada (~10 carreras) en vez de una por carrera. 10x más eficiente. |
| **No modificar el DDL existente** | El scraper se adapta al esquema actual. Los datos que no tienen columna se descartan y se loguean. |
| **Scraping idempotente** | Los repositorios usan `IF NOT EXISTS` + `SCOPE_IDENTITY()` + `MERGE`. Se puede correr N veces sin duplicar. |
| **Clean Architecture** | El dominio no conoce Selenium, HtmlAgilityPack ni SQL. Toda dependencia externa vive en `Infrastructure`. |
| **Patrón Repository** | Desacopla la lógica de scraping del acceso a datos. |
| **Principios SOLID** | Cada parser hace una cosa. Cada repositorio maneja una entidad. Las interfaces viven en el dominio. |
| **Retry con Polly** | El sitio puede fallar transitoriamente. Polly reintenta con backoff exponencial. |
| **Rate limiting de 2s entre requests** | No tumbar el sitio. Ajustable por configuración. |

---

## 4. Stack tecnológico

| Componente | Tecnología |
|---|---|
| Framework | .NET 8 |
| IDE | Visual Studio 2022+ |
| Base de datos | SQL Server 2019+ |
| Acceso a datos | ADO.NET (`Microsoft.Data.SqlClient`) |
| HTTP | `HttpClient` (nativo) |
| Parsing HTML | HtmlAgilityPack |
| Resiliencia | Polly |
| Logging | Serilog (console + archivo) |
| Inyección de dependencias | `Microsoft.Extensions.DependencyInjection` |
| Configuración | `Microsoft.Extensions.Configuration.Json` |
| Tests | xUnit + FluentAssertions |

---

## 5. Estructura del repositorio

```
PalermoScraper.sln
│
├── src/
│   ├── PalermoScraper.Domain/          # Entidades, Value Objects, Interfaces. Sin dependencias.
│   │   ├── Entities/
│   │   ├── ValueObjects/
│   │   └── Abstractions/
│   │       ├── Repositories/
│   │       └── Scraping/
│   │
│   ├── PalermoScraper.Application/     # Servicios de orquestación.
│   │   └── Services/
│   │
│   ├── PalermoScraper.Infrastructure/  # HTTP, parsers, repositorios ADO.NET.
│   │   ├── Http/
│   │   ├── Parsing/
│   │   ├── Scrapers/
│   │   └── Repositories/
│   │
│   └── PalermoScraper.Console/         # Punto de entrada + configuración.
│       ├── Program.cs
│       └── appsettings.json
│
└── tests/
    └── PalermoScraper.Tests/           # Tests unitarios con HTML real como fixture.
        └── Fixtures/
```

### 5.1 Dependencias entre proyectos

```
Console ──► Infrastructure ──► Application ──► Domain
   │              │                  │
   └──────────────┴──────────────────┘
```

El dominio no depende de nadie. Las flechas solo van hacia adentro.

---

## 6. Entidades del dominio

Espejo de las tablas de `bill_benter_v2`. Los nombres y tipos coinciden con el esquema existente.

| Entidad | Tabla BD | Descripción |
|---|---|---|
| `Hipodromo` | `dbo.Hipodromos` | Lugar físico donde se corren las carreras |
| `Caballo` | `dbo.Caballos` | Caballo participante |
| `Jinete` | `dbo.Jinetes` | Jinete que monta |
| `Entrenador` | `dbo.Entrenadores` | Entrenador (cuidador) |
| `Carrera` | `dbo.Carreras` | Carrera individual |
| `Participacion` | `dbo.Participaciones` | Inscripción de un caballo en una carrera |
| `ResultadoCarrera` | `dbo.Resultados_Carreras` | Resultado deportivo de una participación |

> **Nota**: las entidades `Variables_Calculadas` y `Modelo_Coeficientes` **no** son parte del scraper. Se calculan/populan por fuera (`sp_RecalcularVariablesPointInTime` y entrenamiento en Python).

---

## 7. Value Objects

Estructuras que representan **lo que sale del HTML**, antes de persistirse. Viven en el dominio y no dependen de HtmlAgilityPack.

| Value Object | Qué representa |
|---|---|
| `ReunionDescubierta` | Una reunión descubierta en el calendario (fecha + ID del sitio) |
| `CarreraScrapeada` | Una carrera completa extraída de `ver-carreras/{id}` |
| `ParticipanteScrapeado` | Un caballo participante con todos sus datos de la tabla de resultados |
| `GanadorScrapeado` | Datos biográficos del ganador extraídos del bloque C |

---

## 8. Interfaces

### 8.1 Repositorios (`Domain/Abstractions/Repositories`)

```csharp
IHipodromoRepository     → ObtenerOInsertarAsync, ObtenerIdPorNombreAsync
ICaballoRepository       → ObtenerOInsertarAsync, ActualizarDatosBiograficosAsync
IJineteRepository        → ObtenerOInsertarAsync
IEntrenadorRepository    → ObtenerOInsertarAsync
ICarreraRepository       → ObtenerOInsertarAsync
IParticipacionRepository → ObtenerOInsertarAsync
IResultadoRepository     → UpsertAsync
```

Todas devuelven el `ID` generado para poder encadenar inserciones respetando las FKs.

### 8.2 Scrapers (`Domain/Abstractions/Scraping`)

```csharp
ICalendarioScraper → ObtenerReunionesAsync(int anio)
IReunionScraper    → ScrapearAsync(int verDiaId, DateOnly fecha)
```

---

## 9. Mapeo HTML → Base de datos

### 9.1 Bloque A — Metadata de carrera

| Celda HTML | Campo BD | Notas |
|---|---|---|
| `FECHA` | `Carreras.Fecha` | dd/MM/yyyy |
| `HORA` | *no mapeado* | Sin columna en BD |
| `DISTANCIA` | `Carreras.Distancia` | int, en metros |
| `PISTA` | `Carreras.Superficie` + `Carreras.CondicionPista` | split por `\|` |
| `TIEMPO` | *no mapeado* | Tiempo del ganador, sin columna |
| `CONDICIóN` | `Carreras.ClaseCarrera` | texto completo |
| `PREMIOS` | `Carreras.PremioTotal` | primer `$` = monto del 1° |
| `Nro. X - NOMBRE` | `Carreras.NombreCarrera` | del `<h2>` |

### 9.2 Bloque B — Tabla de participantes

| Columna HTML | Campo BD | Notas |
|---|---|---|
| `POS..` | `Resultados_Carreras.PosicionFinal` | `RET` → `NULL` |
| `NRO.` | `Participaciones.NumeroCaja` | int |
| `COMPETIDOR` | `Caballos.Nombre` | + `ver-caballo/{id}` en el href |
| `DISTANCIA.` | `Resultados_Carreras.DistanciaGanador` | texto → parseo tolerante |
| `JOCKEY` | `Jinetes.Nombre` | |
| `CUIDADOR` | `Entrenadores.Nombre` | |
| `CABALLERIZA.` | *no mapeado* | Sin columna en BD |
| `PESO JOCKEY / CABALLO` | `Participaciones.PesoAsignado` | split por `/`, se toma el primero |
| `PAGARIA` | `Participaciones.Odds` | decimal |

### 9.3 Bloque C — Datos del ganador

| Celda HTML | Campo BD | Notas |
|---|---|---|
| `Nombre completo` | `Caballos.Nombre` | confirmación |
| Fecha de nacimiento | `Caballos.FechaNacimiento` | dd-MM-yyyy |
| `Sexo` | `Caballos.Sexo` | MACHO → M, HEMBRA → H |
| `Pelaje` | `Caballos.Color` | |
| `Criador` | `Caballos.Criador` | |
| `Caballeriza` | *no mapeado* | Sin columna en BD |
| `Padre`, `Madre`, `Abuelo Materno` | *no mapeado* | Sin columna en BD |

### 9.4 Datos descartados (sin columna en BD)

Los siguientes datos se extraen del HTML pero **no se persisten** porque el DDL no tiene columna. Se loguean en nivel `Debug` para auditoría:

- Hora de la carrera
- Tiempo del ganador
- Peso del caballo (segundo valor de `PESO JOCKEY / CABALLO`)
- Caballeriza
- Padre, madre, abuelo materno
- Dividendos (Exacta, Trifecta, etc.)

> Si en el futuro se decide persistir alguno, solo hay que agregar la columna al DDL y extender el repositorio. El parser ya los extrae.

---

## 10. Estrategia de scraping

### 10.1 Flujo general

```
┌────────────────────────────────────────────────┐
│ FASE 1: Descubrir reuniones                    │
│ GET /es/turf/calendario-de-carreras/{año}      │
│ → [{ fecha, verDiaId }]                        │
└───────────────────┬────────────────────────────┘
                    ▼
┌────────────────────────────────────────────────┐
│ FASE 2: Scrapear cada reunión                  │
│ GET /es/turf/ver-carreras/{verDiaId}           │
│ → por cada carrera:                            │
│   ├── Bloque A → Carrera                       │
│   ├── Bloque B → Participantes + Resultados    │
│   └── Bloque C → Caballo (update)              │
└───────────────────┬────────────────────────────┘
                    ▼
┌────────────────────────────────────────────────┐
│ FASE 3: Persistir respetando FKs               │
│ Hipodromo → Caballo/Jinete/Entrenador          │
│ → Carrera → Participacion → Resultado          │
└────────────────────────────────────────────────┘
```

### 10.2 Idempotencia

Cada repositorio implementa `ObtenerOInsertar` siguiendo el patrón:

```sql
IF NOT EXISTS (SELECT 1 FROM dbo.Tabla WHERE clave = @clave)
BEGIN
    INSERT INTO dbo.Tabla (...) VALUES (...);
    SET @id = SCOPE_IDENTITY();
END
ELSE
BEGIN
    SELECT @id = ID FROM dbo.Tabla WHERE clave = @clave;
    -- opcional: UPDATE con ISNULL para completar campos faltantes
END
SELECT @id;
```

Esto permite correr el scraper tantas veces como sea necesario sin duplicar.

### 10.3 Rate limiting

- **2 segundos** entre requests por defecto.
- Configurable en `appsettings.json` → `Scraper:DelayEntreRequestsMs`.
- Se aplica tanto entre páginas de reunión como entre años del calendario.

### 10.4 Manejo de errores

- **Retry con Polly**: 3 intentos, backoff exponencial (2s, 4s, 8s) ante errores de red o HTTP 5xx.
- **Fallos por reunión**: se loguean y se continúa con la siguiente. No se aborta el scraping completo.
- **HTML crudo guardado**: cada HTML fallido se guarda en `logs/html-fallidos/` para reprocesar sin volver a pegarle al sitio.

---

## 11. Configuración

`src/PalermoScraper.Console/appsettings.json`:

```json
{
  "ConnectionStrings": {
    "BillBenter": "Server=localhost;Database=bill_benter_v2;Trusted_Connection=True;TrustServerCertificate=True;"
  },
  "Scraper": {
    "BaseUrl": "https://old.palermo.com.ar",
    "DelayEntreRequestsMs": 2000,
    "AnioDesde": 2021,
    "AnioHasta": 2026
  },
  "Serilog": {
    "MinimumLevel": "Information"
  }
}
```

> Marcar `appsettings.json` con **Copy to Output Directory: Copy if newer**.

---

## 12. Cómo ejecutar

### 12.1 Modos disponibles

```bash
# Scrapear el calendario del año configurado (no persiste, solo lista)
dotnet run --project src/PalermoScraper.Console -- calendario

# Scrapear y persistir una reunión completa
dotnet run --project src/PalermoScraper.Console -- reunion 24834
```

### 12.2 Requisitos previos

1. Base `bill_benter_v2` creada con el DDL completo.
2. Cadena de conexión configurada en `appsettings.json`.
3. .NET 8 SDK instalado.

### 12.3 Logs

- Consola (Serilog).
- Archivo diario en `logs/scraper-YYYYMMDD.log`.

---

## 13. Roadmap

### Fase 1 — Esqueleto ✅ *(en progreso)*
- [x] Estructura de la solución (Clean Architecture)
- [x] Entidades del dominio
- [x] Value Objects
- [x] Interfaces de repositorio y scraper
- [ ] Configuración + DI + `Program.cs`

### Fase 2 — Parsers y Scrapers
- [ ] `CalendarioParser` + tests con HTML real
- [ ] `CalendarioScraper`
- [ ] `ReunionParser` (3 bloques) + tests
- [ ] `ReunionScraper`
- [ ] `CaballoParser` + `CaballoScraper` (opcional)

### Fase 3 — Persistencia
- [ ] `Database` (ya creado)
- [ ] `HipodromoRepository`
- [ ] `CaballoRepository`
- [ ] `JineteRepository`
- [ ] `EntrenadorRepository`
- [ ] `CarreraRepository`
- [ ] `ParticipacionRepository`
- [ ] `ResultadoRepository`

### Fase 4 — Orquestación
- [ ] `ScrapeoCalendarioService`
- [ ] `ScrapeoReunionService`
- [ ] Modo `reunion` funcional end-to-end
- [ ] Modo `historico` (batch de varios años)
- [ ] Ejecución de `sp_RecalcularVariablesPointInTime` al finalizar

### Fase 5 — Operación
- [ ] Programación diaria (Task Scheduler / cron)
- [ ] Métricas de scraping (requests, éxitos, fallos)
- [ ] Alertas ante cambios de estructura HTML

---

## 14. Convenciones de código

### 14.1 Nombres

- **Entidades y propiedades**: PascalCase en español, igual que las columnas de la BD (`HipodromoID`, `NombreCarrera`).
- **Interfaces**: prefijo `I` + PascalCase (`ICaballoRepository`).
- **Métodos async**: sufijo `Async` (`ObtenerOInsertarAsync`).
- **Namespaces**: siguen la estructura de carpetas (`PalermoScraper.Domain.Entities`).

### 14.2 Estilo

- `file-scoped namespace` (C# 10+).
- `var` cuando el tipo es evidente por el lado derecho.
- Nullable reference types activado (`<Nullable>enable</Nullable>`).
- `CancellationToken` como último parámetro en todos los métodos async.
- Sin `Thread.Sleep`: usar `Task.Delay`.

### 14.3 Manejo de nulos

- Las propiedades que pueden no venir en el HTML se tipan como `nullable` (`string?`, `int?`).
- En el parser, usar `?.` y `?? null` en lugar de `try/catch`.
- Los `DBNull.Value` se manejan explícitamente al persistir: `(object?)valor ?? DBNull.Value`.

### 14.4 Tests

- Un test por parser, con el HTML real como `Fixture`.
- Los fixtures se guardan como archivos `.html` en `tests/PalermoScraper.Tests/Fixtures/`.
- Nombres de tests: `Metodo_Condicion_ResultadoEsperado()`.

---

## Licencia

Proyecto privado. No distribuir.
