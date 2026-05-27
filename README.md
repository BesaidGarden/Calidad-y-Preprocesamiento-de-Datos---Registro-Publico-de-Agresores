# Documentación del proyecto: Calidad y Preprocesamiento de Datos

Resumen y guía corta sobre qué hace cada notebook, qué decisiones metodológicas se tomaron y qué salidas se generan.

## Objetivo

Producir una versión limpia, normalizada y trazable del Registro Nacional de Personas Sancionadas del INE y preparar una vinculación exploratoria con BANAVIM.

El objetivo de la vinculación no es afirmar identidad absoluta entre registros, sino generar pares candidatos compatibles entre ambas fuentes a partir de campos normalizados y auditados. Dado que las fuentes no comparten un identificador único y BANAVIM no contiene nombre del agresor, los resultados de record linkage deben interpretarse como candidatos para revisión, no como matches definitivos.

## Requisitos

- Python 3.8+
- Paquetes utilizados o requeridos por los notebooks:
  - recordlinkage
  - fuzzywuzzy
  - python-Levenshtein
  - jellyfish
  - unidecode
  - networkx
  - ydata-profiling
  - ftfy
  - pandas, numpy, matplotlib, seaborn
  - requests

Instalación rápida (ejemplo):

```bash
pip install recordlinkage fuzzywuzzy python-Levenshtein jellyfish unidecode networkx ydata-profiling ftfy requests
```

## Notebooks principales

### `Código/agresores_record_linkage.ipynb`

Notebook principal de ingesta, limpieza, perfilado, normalización, consolidación del INE y preparación de bases canónicas para fusión.

### `Código/fusion_individuos.ipynb`

Notebook de fusión/record linkage. Parte de las bases ya preparadas (`ine_fusion` y `bv_fusion`) y construye escenarios de matching por nivel de confianza, validación geográfica contra INEGI y generación de pares candidatos.

---

## Estructura del notebook principal

### 0) Bibliotecas

- Instala e importa librerías necesarias para procesamiento, perfilado, reparación de codificación, normalización y vinculación.

### 1) Ingesta de datos

- Lee `df_ine_raw` desde un Excel del INE.
- Lee hojas `AGRESORES YYYY` de BANAVIM.
- La función `leer_banavim_agresores()` normaliza el encabezado real de las hojas AGRESORES.

### 2) Análisis exploratorio inicial

- Calcula completitud por campo.
- Analiza duplicados exactos por `Nombre`.
- Parsea fechas.
- Extrae etiquetas XML de campos como `Tipo de violencia`.
- Muestra distribuciones básicas.
- Propósito: diagnosticar sin alterar las fuentes originales.

### 3) Limpieza y normalización del INE

Funciones clave:

- `normalizar_nombre()`: normaliza nombres a minúsculas, sin acentos, sin puntuación y con espacios colapsados.
- `nombre_canonico()`: ordena tokens del nombre para apoyar comparaciones.
- `normalizar_estado()`: homologa entidades federativas usando un mapa de estados.
- `parsear_fecha()`: convierte múltiples formatos a datetime.
- `extraer_xml_lista()`: extrae valores desde etiquetas XML.

Resultados principales:

- Base `ine` con columnas derivadas:
  - `nombre`
  - `entidad`
  - `fecha_resolucion`
  - `permanencia_fecha`
  - `tipo_violencia`
  - `medidas_reparacion`
- Filtro temporal a resoluciones de 2020 a 2022.
- Eliminación de columnas con baja completitud cuando corresponde.

### 4) Limpieza BANAVIM

- `limpiar_bv_agresores()` normaliza columnas relevantes:
  - estado
  - municipio
  - sexo
  - edad
  - escolaridad
  - fecha de registro
- Convierte flags de drogas/armas a variables numéricas cuando aplica.
- Concatena los años 2020, 2021 y 2022 en `bv`.

### 5) Consolidación interna del INE: `df_golden`

- Agrupa por `Nombre` para generar un registro consolidado por persona dentro del INE.
- `consolidar_persona()` separa:
  - campos invariantes;
  - campos por incidencia;
  - derivados como `n_incidencias` y `es_reincidente`.

Nota metodológica: `df_golden` se usa como registro maestro interno del INE, pero no como base principal para vincular con BANAVIM, porque BANAVIM no contiene nombre del agresor ni identificador común.

### 6) Reparación de codificación en BANAVIM

- Detecta problemas de mojibake en texto categórico.
- Aplica reparación de codificación con `ftfy.fix_text`.
- Produce `bv_fix`, versión corregida de BANAVIM.
- Conserva `bv` como base previa para trazabilidad.

### 7) Perfilado y preparación para fusión

- Perfila campos categóricos candidatos:
  - `sexo`
  - `estado`
  - `municipio`
  - `vinculo_victima`
- Crea bases canónicas:
  - `ine_fusion`
  - `bv_fusion`
- Estas bases son versiones reducidas y normalizadas para comparación/fusión.
- Conservan identificadores internos:
  - `id_ine_caso`
  - `id_bv_caso`

### 8) Homologación semántica de `vinculo_victima`

- Inicialmente se creó `vinculo_grupo` para auditar equivalencias semánticas entre INE y BANAVIM.
- Se revisó que el campo `vinculo_victima` no significa exactamente lo mismo en ambas fuentes.
- Para el matching definitivo se construyó después una variable operativa más estricta:
  - `vinculo_match`
  - `confianza_vinculo_match`

### 9) Ajustes finales de catálogo

- `limpiar_sexo_final()` homologa sexo a valores comparables.
- `limpiar_estado_final()` corrige variantes residuales de entidad federativa y placeholders.
- Se decidió no corregir municipios manualmente en esta etapa sin validación externa, por su alta granularidad.

---

## Estructura del notebook de fusión: `fusion_individuos.ipynb`

### 1) Carga de bases canónicas

Carga las bases ya generadas por el notebook principal:

- `../Data/fusion_outputs/ine_fusion_2020_2022.csv`
- `../Data/fusion_outputs/banavim_fusion_2020_2022.csv.gz`

Estas son las entradas principales para la etapa de fusión/record linkage.

### 2) Revisión de categorías de vínculo

- Se auditan todos los valores originales de `vinculo_victima`.
- Se revisa qué valores de INE y BANAVIM caen en cada grupo semántico.
- Se decide no usar `vinculo_grupo` como variable definitiva de matching.

### 3) Clasificación definitiva de vínculo para matching

Se construyen directamente desde `vinculo_victima`:

- `vinculo_match`
- `confianza_vinculo_match`

Niveles definidos:

| Nivel | Uso |
|---|---|
| `ALTA` | Correspondencia semántica fuerte. |
| `MEDIA` | Correspondencia residual o débil, útil para revisión. |
| `NO_APTO` | Categorías no comparables o no informativas. |

Categorías principales:

| Fuente | Valor original | `vinculo_match` | Confianza |
|---|---|---|---|
| INE | `PARES` | `PARES` | ALTA |
| BANAVIM | `COMPANERO A` | `PARES` | ALTA |
| INE | `JERARQUICA`, `SUBORDINACION` | `JERARQUICA_SUBORDINACION` | ALTA |
| BANAVIM | `JEFE A O PATRON A` | `JERARQUICA_SUBORDINACION` | ALTA |
| INE/BANAVIM | `OTRO`, `OTRA`, `OTRA RELACION` | `OTRO_RESIDUAL` | MEDIA |
| INE/BANAVIM | `NINGUNA`, `DESCONOCIDO` | `SIN_RELACION_IDENTIFICABLE` | MEDIA |

Las demás categorías se marcan como `NO_APTO_MATCH`.

### 4) Bases por nivel de confianza

Se crean dos escenarios:

| Escenario | Descripción |
|---|---|
| Alta confianza | Solo registros con vínculo de confianza alta. |
| Alta + media confianza | Incluye confianza alta y media. |

Bases generadas:

- `ine_match_alta_export`
- `bv_match_alta_export`
- `ine_match_alta_media_export`
- `bv_match_alta_media_export`

En estas bases se elimina `vinculo_grupo`, porque ya no forma parte de la clasificación definitiva.

### 5) Validación geográfica contra INEGI

Antes del record linkage se valida el campo geográfico.

Procedimiento:

1. Reunir pares únicos `estado + municipio`.
2. Validarlos contra el catálogo oficial de municipios de INEGI.
3. Identificar municipios no encontrados.
4. Revisar manualmente casos no encontrados.
5. Aplicar únicamente correcciones justificadas.
6. Revalidar después de las correcciones.
7. Marcar la bandera `municipio_validado_inegi`.

### 6) Correcciones geográficas manuales

Se aplicaron solo correcciones con justificación clara:

| Estado | Valor original | Valor corregido | Tipo |
|---|---|---|---|
| `CAMPECHE` | `CIUDAD DEL CARMEN` | `CARMEN` | Localidad/cabecera a municipio |
| `CAMPECHE` | `SAN FRANCISCO DE CAMPECHE` | `CAMPECHE` | Localidad/cabecera a municipio |
| `CAMPECHE` | `XPUJIL` | `CALAKMUL` | Localidad a municipio |
| `COLIMA` | `CIUDAD DE ARMERIA` | `ARMERIA` | Localidad/cabecera a municipio |
| `COLIMA` | `CIUDAD DE VILLA DE ALVAREZ` | `VILLA DE ALVAREZ` | Localidad/cabecera a municipio |
| `NAYARIT` | `EL NAYAR` | `DEL NAYAR` | Variante a nombre oficial |

Se conservaron sin cambios valores revisados que corresponden a municipios oficiales o no requerían corrección, como:

- `CINTALAPA`
- `MAGDALENA APASCO`
- `SANTIAGO CHAZUMBA`
- `SOLIDARIDAD`

### 7) Bandera de calidad geográfica

Se crea:

- `municipio_validado_inegi`

Esta bandera permite conservar registros no validados sin eliminarlos, pero distinguiendo su calidad geográfica para la interpretación del record linkage.

Los municipios no validados se conservan en las bases generales; no se eliminan automáticamente.

### 8) Record linkage exacto de alta confianza

Se inicia con un escenario exacto y conservador.

Campos comparados:

- `estado_municipio`
- `sexo`
- `vinculo_match`

El resultado se interpreta como pares candidatos, no como matches definitivos.

Salida principal:

- `matches_exactos_alta_rl`

Este escenario produjo 8 pares candidatos de alta compatibilidad.

### 9) Revisión manual de candidatos

Los 8 pares candidatos se preparan para revisión manual, conservando:

- identificadores de INE y BANAVIM;
- columnas usadas en el matching;
- fechas disponibles;
- vínculo original;
- municipio corregido y municipio original antes de corrección;
- bandera de validación geográfica;
- score o criterios de coincidencia;
- campos de decisión manual.

La revisión manual es necesaria porque la coincidencia exacta en campos disponibles no prueba por sí sola que ambos registros representen el mismo caso.

---

## Salidas generadas

### Bases intermedias y canónicas

- `../Data/ine_incidencias_2020_2022.csv`
- `../Data/banavim_fix_2020_2022.csv`
- `../Data/fusion_outputs/ine_fusion_2020_2022.csv`
- `../Data/fusion_outputs/banavim_fusion_2020_2022.csv.gz`

### Bases por confianza

- `../Data/fusion_outputs/ine_match_alta_confianza_2020_2022.csv`
- `../Data/fusion_outputs/banavim_match_alta_confianza_2020_2022.csv.gz`
- `../Data/fusion_outputs/ine_match_alta_media_confianza_2020_2022.csv`
- `../Data/fusion_outputs/banavim_match_alta_media_confianza_2020_2022.csv.gz`

Los nombres exactos pueden variar si el notebook se ejecuta con nombres alternativos, pero deben conservar la lógica de fuente, escenario y etapa.

---

## Notas metodológicas y supuestos

- La consolidación `df_golden` agrupa registros del INE por nombre; puede tener riesgo de homónimos.
- La fusión INE--BANAVIM no se realiza a nivel identidad individual confirmada.
- BANAVIM no contiene nombre del agresor ni identificador común con INE.
- El record linkage produce pares candidatos compatibles, no matches definitivos.
- El campo `vinculo_victima` fue reinterpretado mediante `vinculo_match` para evitar falsas equivalencias semánticas.
- La validación geográfica se realizó contra INEGI para reducir errores en `estado + municipio`.
- Los municipios no validados se conservan, pero se interpretan con menor confianza.
- Las fechas disponibles pueden no referirse al mismo evento en ambas fuentes; por eso no se usan como condición dura inicial.
- La revisión manual de candidatos es obligatoria antes de afirmar coincidencia de caso.

---
Documentación actualizada con la sesión de preparación de bases, validación geográfica y primer record linkage exacto.
