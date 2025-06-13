# Claustering Map Crimen

**Geospatial Crime Analysis with Python**

---

This Python application performs geospatial security analysis by:

- **Identifying crime hot spots** using DBSCAN clustering.
- **Calculating optimal escape routes** via NetworkX road network analysis.
- **Proposing strategic checkpoints** for law enforcement.

---

## Features

- **Data cleaning and preparation** for hemerographic crime datasets.
- **Cluster detection** for crime hot spots using DBSCAN.
- **Interactive mapping** of clusters and crime details using Folium.
- **Custom symbology and legend** for crime types and cluster IDs.
- **Detailed popups** with contextual crime information.
- **Distance scale and auxiliary controls** for map usability.

---

## Quick Start

### 1. Install Required Libraries

Install Python dependencies:

```bash
pip install pandas numpy geopandas scikit-learn matplotlib folium
```

If using the scale bar in Matplotlib, you may also need:

```bash
pip install matplotlib-scalebar
```

---

### 2. Data Preparation

#### 2.1. Load Your CSV

```python
import pandas as pd

file = r'file_to_read.csv'
pandas_file = pd.read_csv(file, encoding='Latin-1')
```

#### 2.2. Clean Unnecessary Columns

```python
purgue_file = pandas_file.drop(
    columns=['Unnamed: 25', 'Unnamed: 21', 'Unnamed: 20'],
    errors='ignore'
)
```

#### 2.3. Clean Erroneous Values

Define functions for cleaning numeric and string columns:

```python
import numpy as np

def purgue_number_values(column):
    column = column.replace({
        'n/a': np.nan, 'menor': 17, 'n': np.nan, 'varias': np.nan, 'n\\a': np.nan,
        'individual': 1, 'pareja': 2, 'unico': 1, 'dos': 2, 'tres': 3,
        'varios': np.nan, 9: np.nan
    })
    column = column.dropna()
    column = column.astype(int)
    return column

def purgue_string_values(column):
    column = column.dropna()
    column = column.astype(str)
    return column
```

#### 2.4. Extract Key Crime Data

```python
edad_del_criminal = purgue_number_values(purgue_file['Edad del delincuente'])
num_de_victimas = purgue_number_values(purgue_file['numero de victima'])
participacion_de_delincuentes = purgue_number_values(purgue_file['participacion de delincuentes'])
```

#### 2.5. Calculate Averages

```python
average_edad = edad_del_criminal.mean()
average_num_victimas = num_de_victimas.mean()
average_participacion = participacion_de_delincuentes.mean()

mean_resultado = pd.DataFrame({
    'Columnas': [
        'average Edad del Criminal',
        'Average Numero de Victimas',
        'participacion de delincuentes'
    ],
    'Valores': [average_edad, average_num_victimas, average_participacion]
})
print(mean_resultado)
```

---

## Custom Symbology

### Define Icons for Crimes

```python
delito_icono = {
    'robo': ('fa-mask', 'blue'),
    'asalto': ('fa-user-secret', 'orange'),
    'asesinato': ('fa-skull-crossbones', 'black'),
    'feminicidio': ('fa-venus', 'pink'),
    'violacion': ('fa-venus-mars', 'purple'),
    # ... more crime types
    'otro': ('fa-circle', 'lightgray')
}

def icono_delito(delito):
    if pd.isna(delito):
        return ('fa-question', 'gray')
    delito = str(delito).strip().lower()
    for key in delito_icono:
        if key in delito:
            return delito_icono[key]
    return delito_icono['otro']
```

---

## Clustering and Mapping

### 1. Generate Clusters

```python
import geopandas as gpd
from sklearn.cluster import DBSCAN

def generar_cluster_general(file_path, eps_metros=500, min_samples=3):
    df = pd.read_csv(file_path, encoding='latin-1')
    gdf = gpd.GeoDataFrame(df, geometry=gpd.points_from_xy(df.x, df.y), crs="EPSG:4326")
    gdf_utm = gdf.to_crs(epsg=32614)
    coords = gdf_utm.geometry.apply(lambda geom: (geom.x, geom.y)).tolist()

    db = DBSCAN(eps=eps_metros, min_samples=min_samples, metric='euclidean')
    gdf_utm["cluster"] = db.fit_predict(coords)
    gdf["cluster"] = gdf_utm["cluster"]
    return gdf, gdf_utm
```

### 2. Visualize Clusters with Matplotlib

```python
import matplotlib.pyplot as plt
from mpl_toolkits.axes_grid1.anchored_artists import AnchoredSizeBar
import matplotlib.font_manager as fm

fig, ax = plt.subplots(figsize=(10, 10))
gdf_utm.plot(ax=ax, column="cluster", categorical=True, legend=True, cmap='tab10', markersize=30)
plt.title("General Crime Clusters with DBSCAN", fontsize=14)
ax.grid(True, linestyle='--', alpha=0.5)

for idx, row in gdf.iterrows():
    ax.annotate(
        text=str(row['ID']),
        xy=(gdf_utm.geometry[idx].x, gdf_utm.geometry[idx].y),
        xytext=(3, 3), textcoords="offset points",
        fontsize=7, color='black'
    )

scalebar = AnchoredSizeBar(ax.transData, size=500, label='500 m',
    loc='lower right', pad=0.5, color='black', frameon=False,
    size_vertical=5, fontproperties=fm.FontProperties(size=10))
ax.add_artist(scalebar)

ax.set_xlabel("UTM East (meters)")
ax.set_ylabel("UTM North (meters)")
plt.tight_layout()
plt.show()
```

---

### 3. Interactive Folium Map

```python
import folium
from folium.plugins import MiniMap, MeasureControl, MousePosition

lat_mean = gdf.geometry.y.mean()
lon_mean = gdf.geometry.x.mean()
m = folium.Map(location=[lat_mean, lon_mean], zoom_start=13, tiles='OpenStreetMap')

# Add map title
title_html = '''
<h3 align="center" style="font-size:16px;margin-bottom:5px;"><b>Clústeres de Crímenes y Simbología</b></h3>
'''
m.get_root().html.add_child(folium.Element(title_html))

# Add auxiliary controls
minimap = MiniMap(tile_layer='OpenStreetMap', toggle_display=True, position='bottomright')
m.add_child(minimap)

# Define cluster colors
cluster_colors = ['blue', 'orange', 'green', 'red', 'purple', 'brown', 'pink', 'gray', 'cadetblue', 'black']
noise_color = 'black'

# Add crime points
for _, row in gdf.iterrows():
    cluster_id = row['cluster']
    cluster_color = cluster_colors[cluster_id % len(cluster_colors)] if cluster_id != -1 else noise_color
    icon_name, _ = icono_delito(row['Delito'])

    resumen_html = f"""
    <div style="font-family:Arial; font-size:13px;">
        <b>ID:</b> {row['ID']}<br>
        <b>Delito:</b> {row['Delito']}<br>
        <b>Colonia:</b> {row['Colonia']}<br>
        <b>Dirección:</b> {row.get("Calles o lugares especificos (Donde ocurrio)", "N/A")}<br>
        <hr>
        <b>Sexo delincuente:</b> {row.get('Sexo del delincuente', 'N/A')}<br>
        <b>Edad delincuente:</b> {row.get('Edad del delincuente', 'N/A')}<br>
        <b>Edad víctima:</b> {row.get('edad de la victima', 'N/A')}<br>
        <b>Género víctima:</b> {row.get('genero de la victima', 'N/A')}<br>
        <b>Número de víctima:</b> {row.get('numero de victima', 'N/A')}<br>
        <hr>
        <b>Drogas/Arma:</b> {row.get('Tipo de droga o arma', 'N/A')}<br>
        <b>Marca/modelo auto:</b> {row.get('Marca y modelo de auto robado  ', 'N/A')}<br>
        <b>Zona oscura:</b> {row.get('dark zone', 'N/A')}<br>
        <b>Participación delincuentes:</b> {row.get('participacion de delincuentes', 'N/A')}<br>
        <b>Nacionalidad:</b> {row.get('nacionalidad', 'N/A')}<br>
        <b>Observaciones:</b> {row.get('obssrvaciones', 'N/A')}<br>
    </div>
    """

    folium.Marker(
        location=[row.geometry.y, row.geometry.x],
        icon=folium.Icon(icon=icon_name, prefix='fa', color=cluster_color),
        popup=folium.Popup(resumen_html, max_width=350)
    ).add_to(m)
```

---

## Custom Legend

```python
legend_html = (
    '<div style="'
    'position: fixed; bottom: 50px; left: 50px; width: 300px; max-height: 400px; overflow-y: auto; z-index:9999; '
    'font-size:14px; background:rgba(255,255,255,0.95); border:2px solid #444; border-radius:8px; padding:10px;">'
    '<h4 style="margin-top:0;"><b>Simbología</b></h4>'
    '<b>Clústeres de Crímenes</b><br>'
    +
    ''.join([
        f'<span style="background:{cluster_colors[i % len(cluster_colors)]}; border-radius:50%; display:inline-block; width:12px; height:12px; margin-right:5px;"></span> Clúster {i}<br>'
        for i in sorted([c for c in gdf["cluster"].unique() if c != -1])
    ])
    +
    f'<span style="background:{noise_color}; border-radius:50%; display:inline-block; width:12px; height:12px; margin-right:5px;"></span> Ruido (Clúster -1)<br>'
    '<hr>'
    '<b>Simbología de Delitos</b><br>'
    '<i class="fa fa-mask"></i> Robo<br>'
    '<i class="fa fa-user-secret"></i> Asalto<br>'
    '<i class="fa fa-skull-crossbones"></i> Asesinato/Intento<br>'
    '<i class="fa fa-venus"></i> Feminicidio<br>'
    '<i class="fa fa-venus-mars"></i> Violación/Abuso sexual<br>'
    '<i class="fa fa-door-open"></i> Allanamiento<br>'
    '<i class="fa fa-cannabis"></i> Portación/Poseción de droga<br>'
    '<i class="fa fa-knife"></i> Arma blanca<br>'
    '<i class="fa fa-gun"></i> Arma de fuego<br>'
    '<i class="fa fa-biohazard"></i> Armas y droga<br>'
    '<i class="fa fa-fist-raised"></i> Agresión<br>'
    '<i class="fa fa-user-lock"></i> Secuestro<br>'
    '<i class="fa fa-dna"></i> Cuerpo/Hallazgo<br>'
    '<i class="fa fa-user-slash"></i> Desaparición<br>'
    '<i class="fa fa-house"></i> Violencia familiar<br>'
    '<i class="fa fa-bomb"></i> Atentado<br>'
    '<i class="fa fa-bullseye"></i> Tiroteo<br>'
    '<i class="fa fa-question"></i> Otro/Desconocido<br>'
    '<hr>'
    '<span style="font-size:12px;color:#444;">La escala de distancia está en la esquina inferior izquierda.</span>'
    '</div>'
)

m.get_root().html.add_child(folium.Element(legend_html))
```

---

## Links

- [Editor.md](https://pandao.github.io/editor.md/)
- [GitHub Link](https://github.com)
- [Localhost Example](http://localhost/)

---

## Attribution

- Inspired by open-source mapping and data science tools.
- Icons from [FontAwesome](https://fontawesome.com/).
- Built with [GeoPandas](https://geopandas.org/), [Folium](https://python-visualization.github.io/folium/), and [scikit-learn](https://scikit-learn.org/).

---
