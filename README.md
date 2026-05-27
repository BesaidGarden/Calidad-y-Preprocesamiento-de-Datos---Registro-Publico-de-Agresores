# Calidad y Preprocesamiento de Datos - Registro Público de Agresores

## Objetivos
Integrar y vincular los registros de agresores del INE y BANAVIM (2020–2022) mediante limpieza, consolidación por individuo (Golden Record) y dos
estrategias de record linkage para producir:
1. Perfiles estadísticos de violencia por municipio y estado.
2. Un conjunto documentado de pares candidatos que aparezcan en ambas bases de datos.

## Requisitos
- Python 3.8 - 3.12
- Paquetes:
  - recordlinkage
  - fuzzywuzzy
  - python-Levenshtein
  - jellyfish
  - unidecode
  - networkx
  - ydata-profiling
  - ftfy
  - pandas, numpy, matplotlib, seaborn
  - geopy
  - scikit-learn
  - openpyxl
  - requests

## Estructura

```
├── Código/
│   ├── perfilado.ipynb          # EDA de INE y BANAVIM
│   ├── limpieza.ipynb           # Limpieza, normalización y Golden Record → exporta 3 archivos
│   ├── fusion.ipynb             # Record linkage (SNM + exacto), perfiles y visualización
│   ├── analisis.pkl             # Creación de gráficas para el análisis de datos limpios y fusionados
│   ├── bv_2020.pkl              # BANAVIM 2020 pre-parseado desde Excel
│   ├── bv_2021.pkl              # BANAVIM 2021 pre-parseado desde Excel
│   └── bv_2022.pkl              # BANAVIM 2022 pre-parseado desde Excel
│
├── Data/
│   ├── Registro-nacional-de-personas-sancionadas (INE).xlsx   # Fuente INE
│   ├── 2020.xlsx / 2021.xlsx / 2022.xlsx                      # Fuentes BANAVIM
│   │
│   ├── ine_incidencias_2020_2022.csv                   # INE limpio y filtrado (salida de limpieza.ipynb)
│   ├── banavim_fix_2020_2022.pkl/csv                   # BANAVIM con codificación reparada (salida de limpieza.ipynb)
│   ├── df_golden.pkl                                   # Golden Record INE por individuo (salida de limpieza.ipynb)
│   ├── ine_fusion_geo.csv                              # INE con coordenadas geocodificadas
│   ├── golden_g_geo.csv                                # Golden Record con coordenadas geocodificadas
|   ├── Clusters geográficos - INE fusion.png           # Imagen de los clusters generados (salida de fusion.ipynb)
|   ├── Clusters geográficos - INE Golden record.png    # Imagen de los clusters golden record (salida de fusion.ipynb)
│   │
│   ├── graficas/                        # Figuras exportadas (salida de analisis.ipynb)
│   │
│   └── fusion_outputs/                  # Resultados del record linkage (salidas de fusion.ipynb)
│       ├── ine_fusion_2020_2022.csv
│       ├── banavim_fusion_2020_2022.csv.gz
│       ├── matches_snm.csv
│       ├── matches_exactos_alta_confianza.csv
│       ├── tabla_revision_manual_alta_exacta.csv
│       ├── perfil_municipal_integrado.csv
│       ├── perfil_estado_integrado.csv
│       ├── resumen_matching_exacto.csv
│       └── catalogo_inegi_municipios_usados.csv
│
└── Terminos y Condiciones - BANAVIM/
```

Los notebooks deben ejecutarse en orden: `perfilado.ipynb` → `limpieza.ipynb` → `fusion.ipynb`→ `analisis.ipynb` 

---

**Presentación del proyecto (Canva):** <https://www.canva.com/design/DAHKiNaOcwE/jFqs8YRy3IOL6kK3_En9Hg/>

**Reporte del proyecto (Overleaf / LaTeX):** <https://www.overleaf.com/project/6a1337b580bb716e3cdc5af5>
