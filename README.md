# bill# Palermo Scraper

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
-benter-docs
