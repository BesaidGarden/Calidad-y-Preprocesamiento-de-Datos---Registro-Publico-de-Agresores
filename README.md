# Documentación del notebook: Calidad y Preprocesamiento de Datos

Resumen y guía corta sobre qué hace cada bloque del notebook y las salidas generadas.

## Objetivo
Producir una versión limpia, normalizada y deduplicada del Registro Nacional de Personas Sancionadas (INE) y enriquecerlo con información auxiliar de BANAVIM para generar perfiles municipales integrados.

## Requisitos
- Python 3.8+
- Paquetes (instalar en el notebook o entorno):
  - recordlinkage
  - fuzzywuzzy
  - python-Levenshtein
  - jellyfish
  - unidecode
  - networkx
  - ydata-profiling
  - ftfy
  - pandas, numpy, matplotlib, seaborn

Instalación rápida (ejemplo):

```bash
pip install recordlinkage fuzzywuzzy python-Levenshtein jellyfish unidecode networkx ydata-profiling ftfy
```

## Estructura y qué hace cada bloque

0) Bibliotecas
- Instala (opcional) y importa librerías necesarias para procesamiento, perfilado y vinculación.

1) Ingesta de datos
- Lee `df_ine_raw` desde un Excel del INE y las hojas `AGRESORES YYYY` de BANAVIM.
- Función `leer_banavim_agresores()` normaliza el header real de las hojas AGRESORES.

2) Análisis exploratorio (EDA)
- Calcula completitud por campo, analiza duplicados exactos por `Nombre`, parsea fechas, extrae etiquetas XML de `Tipo de violencia` y muestra distribuciones básicas.
- Propósito: diagnóstico sin alterar las fuentes.

3) Limpieza y normalización (INE)
- Funciones clave:
  - `normalizar_nombre()`: lowercase, sin acentos, sin puntuación, espacios colapsados (retorna None para vacíos).
  - `nombre_canonico()`: tokens ordenados (útil para bloqueo exacto).
  - `normalizar_estado()`: normaliza entidades usando `MAPA_ESTADOS`.
  - `parsear_fecha()`: convierte múltiples formatos a datetime; 'Indeterminada' -> NaT.
  - `extraer_xml_lista()`: extrae valores entre etiquetas `<element>`.
- Crea `ine` con columnas derivadas: `nombre`, `entidad`, `fecha_resolucion`, `permanencia_fecha`, `tipo_violencia`, `medidas_reparacion`.
- Filtra `ine` a resoluciones 2020–2022 y elimina columnas con <20% completitud.

4) Limpieza BANAVIM
- `limpiar_bv_agresores()` normaliza columnas relevantes (estado, municipio, sexo, edad, escolaridad, fecha_registro) y convierte flags de drogas/armas a numéricos.
- Concatenación de años en `bv`.

5) Consolidación (golden) — INE
- Agrupa por `Nombre` y utiliza `consolidar_persona()` para producir un registro por persona con:
  - Campos invariantes (moda / valor más largo en empate).
  - Campos por incidencia: listas ordenadas por `fecha_resolucion` (conducta, sanción, expediente, fechas).
  - Derivados: `n_incidencias`, `es_reincidente`.
- Resultado: `df_golden`.

6) Reparación de codificación (BANAVIM)
- Detecta mojibake (caracteres de codificación rota) y aplica `ftfy.fix_text` en `reparar_codificacion()`.
- Resultado: `bv_fix` con texto corregido y reporte antes/después.

7) Perfilado y preparación para fusión
- Perfilado de campos categóricos candidatos (`sexo`, `estado`, `municipio`, `vinculo_victima`).
- Funciones de normalización para fusión:
  - `normalizar_categoria_fusion()` y `normalizar_sexo_fusion()` para crear campos canónicos.
- Crea `ine_fusion` y `bv_fusion` (versiones reducidas y canónicas con `id_ine_caso` / `id_bv_caso`).

8) Homologación semántica de `vinculo_victima`
- Mapas `MAPA_VINCULO_INE` y `MAPA_VINCULO_BANAVIM` que agrupan variantes textuales en categorías (PAREJA_EXPAREJA, FAMILIAR, PARES, INSTITUCIONAL_POLITICA, etc.).
- Función `homologar_vinculo()` asigna `MISSING` o `REVISAR` según corresponda; se itera el mapa para reducir `REVISAR`.

9) Ajustes finales de catálogo
- Funciones `limpiar_sexo_final()` y `limpiar_estado_final()` homologan valores residuales y placeholders a `NaN` o catálogos canónicos.

10) Vinculación (Record Linkage) — Sorted Neighbourhood Method (SNM)
- Indexación por `municipio` con ventana (`window=3`) para generar pares candidatos.
- Comparador con:
  - similitud jarowinkler en `municipio` (threshold 0.85),
  - match exacto en `vinculo_grupo`,
  - match exacto en `sexo`.
- Se suma la puntuación (`score`) y se filtran pares con `score >= 2.5`.
- `matches_full` es el dataset enriquecido con columnas de ambas fuentes.
- `perfil_municipal`: agregación por `municipio`, `estado`, `vinculo_grupo` con estadísticas (n sanciones INE, n casos BANAVIM, edad media, % portaba arma, % alcohol).

## Salidas generadas (paths relativos usados en el notebook)
- `../Data/ine_incidencias_2020_2022.csv` — INE limpio y recortado.
- `../Data/banavim_fix_2020_2022.csv` — BANAVIM con texto corregido.
- `../Data/fusion_outputs/ine_fusion_2020_2022.csv` — INE canónico para fusión.
- `../Data/fusion_outputs/banavim_fusion_2020_2022.csv.gz` — BANAVIM canónico (comprimido).
- `../Data/fusion_outputs/matches_snm.csv` — pares vinculados (SNM).
- `../Data/fusion_outputs/perfil_municipal_integrado.csv` — perfiles municipales agregados.
- Auditorías y CSVs de perfilado en `../Data/fusion_outputs/`.

## Notas importantes y supuestos
- Muchas funciones retornan `None` / `NaN` / `NaT` para preservar semántica de faltantes (no usar cadenas vacías).
- Consolidación por `Nombre` asume que el mismo texto identifica a la misma persona (riesgo de homónimos).
- La vinculación no prueba identidad absoluta; es una unión por similitud y contexto municipal.

## Próximos pasos sugeridos
- Ejecutar validación manual de una muestra de `matches_snm.csv` para verificar falsos positivos.
- Aplicar blocking difuso por `nombre_canonico` y usar comparadores string más ricos si se desea unir por identidad real.
- Versionar y commitear los CSVs de salida.

---
Generado automáticamente como documentación del notebook.
# Calidad-y-Preprocesamiento-de-Datos---Registro-Publico-de-Agresores