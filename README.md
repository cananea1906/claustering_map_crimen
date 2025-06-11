# claustering_map_crimen
This Python application performs geospatial security analysis. It identifies crime hot spots using DBSCAN. Calculates optimal escape routes via NetworkX road network analysis. Proposes strategic checkpoints. Transforms raw data into actionable tactical intelligence for military &amp; urban security.


import pandas as pd
import numpy as np
import geopandas as gpd
from sklearn.cluster import DBSCAN
from shapely.geometry import Point
import matplotlib.pyplot as plt
from mpl_toolkits.axes_grid1.anchored_artists import AnchoredSizeBar
import matplotlib.font_manager as fm
import folium
from folium.plugins import MiniMap, MeasureControl, MousePosition # Importa MousePosition
from datetime import datetime

file = r'file_to_read.csv'
pandas_file = pd.read_csv(file, encoding='Latin-1')
purgue_file = pandas_file.drop(columns=['Unnamed: 25', 'Unnamed: 21', 'Unnamed: 20'], errors='ignore') # Añadido errors='ignore'

def purgue_number_values(column):
    column = column.replace({'n/a': np.nan, 'menor': 17, 'n': np.nan, 'varias': np.nan, 'n\\a': np.nan,
                             'individual': 1, 'pareja': 2, 'unico': 1, 'dos': 2, 'tres': 3,
                             'varios': np.nan, 9: np.nan, 'cinco': 5, 'seis': 6, 'siete': 7,
                             'ocho': 8, 'nueve': 9})
    column = column.dropna()
    column = column.astype(int)
    return column

def purgue_string_values(column):
    column = column.dropna()
    column = column.astype(str)
    return column

edad_del_criminal = purgue_number_values(purgue_file['Edad del delincuente'])
num_de_victimas = purgue_number_values(purgue_file['numero de victima'])
participacion_de_delincuentes = purgue_number_values(purgue_file['participacion de delincuentes'])

average_edad = edad_del_criminal.mean()
average_num_victimas = num_de_victimas.mean()
average_participacion = participacion_de_delincuentes.mean()

mean_resultado = pd.DataFrame({
    'Columnas': ['average Edad del Criminal', 'Average Numero de Victimas', 'participacion de delincuentes'],
    'Valores': [average_edad, average_num_victimas, average_participacion]
})
print(mean_resultado)

# Diccionario de delitos a íconos FontAwesome (se mantiene original)
delito_icono = {
    'robo':                  ('fa-mask', 'blue'),
    'asalto':                ('fa-user-secret', 'orange'),
    'asesinato':             ('fa-skull-crossbones', 'black'),
    'feminicidio':           ('fa-venus', 'pink'),
    'violacion':             ('fa-venus-mars', 'purple'),
    'allanamiento':          ('fa-door-open', 'gray'),
    'portacion de droga':    ('fa-cannabis', 'green'),
    'portacion de arma blanca': ('fa-knife', 'darkred'),
    'portacion de arma de fuego': ('fa-gun', 'darkred'),
    'portacion de armas y droga': ('fa-biohazard', 'darkgreen'),
    'agresion':              ('fa-fist-raised', 'red'),
    'agresion sexual':       ('fa-venus-mars', 'purple'),
    'abuso sexual':          ('fa-venus-mars', 'purple'),
    'tentativa':             ('fa-question', 'gray'),
    'detencion':             ('fa-handcuffs', 'cadetblue'),
    'secuestro':             ('fa-user-lock', 'darkpurple'),
    'cuerpo':                ('fa-dna', 'black'),
    'hallazgo de cuerpos':   ('fa-dna', 'black'),
    'desaparicion':          ('fa-user-slash', 'lightgray'),
    'violencia familiar':    ('fa-house', 'brown'),
    'incautacion de droga':  ('fa-cannabis', 'green'),
    'atentado':              ('fa-bomb', 'darkred'),
    'tiroteo':               ('fa-bullseye', 'darkblue'),
    'intento de asesinato':  ('fa-skull-crossbones', 'black'),
    'posesion':              ('fa-cannabis', 'green'),
    'cadaver':               ('fa-dna', 'black'),
    'hallanamiento':         ('fa-door-open', 'gray'),
    'acoso':                 ('fa-exclamation', 'orange'),
    'nan':                   ('fa-question', 'gray'),
    # Genérico
    'otro':                  ('fa-circle', 'lightgray')
}

def icono_delito(delito):
    """Devuelve el ícono para el delito dado."""
    if pd.isna(delito):
        return ('fa-question', 'gray')
    delito = str(delito).strip().lower()
    for key in delito_icono:
        if key in delito:
            return delito_icono[key]
    return delito_icono['otro']

def generar_cluster_general(file_path, eps_metros=500, min_samples=3):
    df = pd.read_csv(file_path, encoding='latin-1')

    gdf = gpd.GeoDataFrame(df, geometry=gpd.points_from_xy(df.x, df.y), crs="EPSG:4326")
    gdf_utm = gdf.to_crs(epsg=32614)

    coords = gdf_utm.geometry.apply(lambda geom: (geom.x, geom.y)).tolist()

    db = DBSCAN(eps=eps_metros, min_samples=min_samples, metric='euclidean')
    gdf_utm["cluster"] = db.fit_predict(coords)
    gdf["cluster"] = gdf_utm["cluster"]

    clusters_unicos = sorted(gdf["cluster"].unique())

    for clus in clusters_unicos:
        print(f"\n{'❌' if clus == -1 else '✅'} Clúster {clus if clus != -1 else 'fuera de clúster (noise)'}:")
        subset = gdf[gdf["cluster"] == clus]
        for _, row in subset.iterrows():
            print(f"   ➤ ID: {row['ID']} | Delito: {row['Delito']} | Colonia: {row['Colonia']}")

    # --- Visualización Matplotlib (sin cambios aquí) ---
    fig, ax = plt.subplots(figsize=(10, 10))
    gdf_utm.plot(ax=ax, column="cluster", categorical=True, legend=True, cmap='tab10', markersize=30)
    plt.title("Clústeres generales de crímenes con DBSCAN", fontsize=14)

    ax.grid(True, linestyle='--', alpha=0.5)

    for idx, row in gdf.iterrows():
        ax.annotate(
            text=str(row['ID']),
            xy=(gdf_utm.geometry[idx].x, gdf_utm.geometry[idx].y),
            xytext=(3, 3),
            textcoords="offset points",
            fontsize=7,
            color='black'
        )

    scalebar = AnchoredSizeBar(
        transform=ax.transData,
        size=500,
        label='500 m',
        loc='lower right',
        pad=0.5,
        color='black',
        frameon=False,
        size_vertical=5,
        fontproperties=fm.FontProperties(size=10)
    )
    ax.add_artist(scalebar)

    ax.set_xlabel("UTM Este (metros)")
    ax.set_ylabel("UTM Norte (metros)")
    plt.tight_layout()
    plt.show()

    # --- Visualización Folium con mini mapa, escala, grid y simbología personalizada ---
    lat_mean = gdf.geometry.y.mean()
    lon_mean = gdf.geometry.x.mean()
    m = folium.Map(location=[lat_mean, lon_mean], zoom_start=13, tiles='OpenStreetMap')

    # Título del mapa de Folium
    title_html = '''
             <h3 align="center" style="font-size:16px;margin-bottom:5px;"><b>Clústeres de Crímenes y Simbología</b></h3>
             '''
    m.get_root().html.add_child(folium.Element(title_html))

    # Mini mapa
    minimap = MiniMap(tile_layer='OpenStreetMap', toggle_display=True, position='bottomright')
    m.add_child(minimap)

    # Escala (regla de distancia)
    m.add_child(MeasureControl(primary_length_unit='meters', position='bottomleft'))

    # Grid (Muestra coordenadas en el ratón)
    MousePosition().add_to(m)

    # Colores para clusters (Tus colores originales)
    cluster_colors = [
        'red', 'blue', 'green', 'purple', 'orange', 'darkred', 'lightred',
        'beige', 'darkblue', 'darkgreen', 'cadetblue', 'darkpurple', 'white',
        'pink', 'lightblue', 'lightgreen', 'gray', 'black', 'lightgray'
    ]
    # Color para el ruido (cluster -1)
    noise_color = 'black'

    # Puntos dispersos con íconos personalizados y color por clúster (se mantiene original)
    for _, row in gdf.iterrows():
        cluster_id = row['cluster']
        cluster_color = cluster_colors[cluster_id % len(cluster_colors)] if cluster_id != -1 else 'black'
        icon_name, _ = icono_delito(row['Delito']) # Obtenemos el nombre del icono

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
            icon=folium.Icon(icon=icon_name, prefix='fa', color=cluster_color), # Usa el color del cluster
            popup=folium.Popup(resumen_html, max_width=350)
        ).add_to(m)

    # Leyenda personalizada con scroll bar
    legend_html = (
        '<div style="'
        'position: fixed; bottom: 50px; left: 50px; width: 300px; max-height: 400px; overflow-y: auto; z-index:9999; ' # Aquí están los cambios
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
        '<i class="fa fa-venus-mars"></i> Agresión sexual<br>' # Añadido para consistencia si es diferente al de abuso sexual
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

    folium.LayerControl().add_to(m)

    fecha_hora = datetime.now().strftime('%Y%m%d_%H%M%S')
    mapa_html = f"clusters_acercamientos_{fecha_hora}.html"
    m.save(mapa_html)
    print(f"\n🗺️ Mapa interactivo guardado en: {mapa_html}")

# Ejecutar función
ruta_csv = r"C:\Users\velaz\Downloads\Crimenes base de datos A (1)(in).csv"
generar_cluster_general(ruta_csv, eps_metros=500, min_samples=3)
