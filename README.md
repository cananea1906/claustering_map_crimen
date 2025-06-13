# claustering_map_crimen
This Python application performs geospatial security analysis. It identifies crime hot spots using DBSCAN. Calculates optimal escape routes via NetworkX road network analysis. Proposes strategic checkpoints. Transforms raw data into actionable tactical intelligence for military &amp; urban security.

### Claustering Map Crimen

- This Python application performs geospatial security analysis. It identifies crime hot spots using DBSCAN. Calculates optimal escape routes via NetworkX road network analysis. Proposes strategic checkpoints. Transforms raw data into actionable tactical intelligence for military &amp; urban security.;

# Editor.md

![](https://pandao.github.io/editor.md/images/logos/editormd-logo-180x180.png)

![](https://img.shields.io/github/stars/pandao/editor.md.svg) ![](https://img.shields.io/github/forks/pandao/editor.md.svg) ![](https://img.shields.io/github/tag/pandao/editor.md.svg) ![](https://img.shields.io/github/release/pandao/editor.md.svg) ![](https://img.shields.io/github/issues/pandao/editor.md.svg) ![](https://img.shields.io/bower/v/editor.md.svg)


**COMMIT CODE**

###Links

[Links](http://localhost/)

[Links with title](http://localhost/ "link title")

`<link>` : <https://github.com>

[Reference link][id/name] 

[id/name]: http://link-url/

GFM a-tail link @pandao

###Bodie Code Commented

####Install the libraries

`$ npm install marked`

####Import the libraries

import the libraries that we will use with `import`.

    import pandas as pd
    import numpy as np
    import geopandas as gpd
    from sklearn.cluster import DBSCAN
    from shapely.geometry import Point
    import matplotlib.pyplot as plt
    from mpl_toolkits.axes_grid1.anchored_artists import AnchoredSizeBar
    import matplotlib.font_manager as fm
    import folium
    from folium.plugins import MiniMap, MeasureControl, MousePosition
    from datetime import datetime

    
#####read the CSV file and encode to latin-1 because the original db was wrote in spanish

	file = r'file_to_read.csv'
	pandas_file = pd.read_csv(file, encoding='Latin-1')

####This is a example to how we clean the unnecesary columns, you can modify it, we add "errors" for some human errors in the data capture

	purgue_file = pandas_file.drop(columns=['Unnamed: 25', 'Unnamed: 21', 'Unnamed: 20'], errors='ignore')

####I created a function to clean the data capture mistakes, the data was capture manually, because is a hemerographic data base, then we can make mistakes.
I recommend adding custom changes, as everyone has different data capture errors.

	def purgue_number_values(column):
		column = column.replace({'n/a': np.nan, 'menor': 17, 'n': np.nan, 'varias': 		np.nan, 'n\\a': np.nan, 'individual': 1, 'pareja': 2, 'unico': 1, 'dos': 2, 'tres':    			3, 'varios': np.nan, 9: np.nan, 'cinco': 5, 'seis': 6, 'siete': 7, 'ocho': 8, 'nueve': 9})
		 column = column.dropna()
         column = column.astype(int)
         return column

####Convert certain values to strings only:

	def purgue_string_values(column):
		column = column.dropna()
		column = column.astype(str)
		return column

####Here we extracted some data on the crimes, such as the age of the offenders, the number of victims and the number of offenders involved in the crime.

	edad_del_criminal = purgue_number_values(purgue_file['Edad del delincuente'])
	num_de_victimas = purgue_number_values(purgue_file['numero de victima'])
	participacion_de_delincuentes = purgue_number_values(purgue_file['participacion de delincuentes'])


#### Get average values.

	average_edad = edad_del_criminal.mean()
	average_num_victimas = num_de_victimas.mean()
	average_participacion = participacion_de_delincuentes.mean()



#### Create a 'DataFrame' using the average values:
	mean_resultado = pd.DataFrame({
    'Columnas': ['average Edad del Criminal', 'Average Numero de Victimas', 'participacion de delincuentes'],
    'Valores': [average_edad, average_num_victimas, average_participacion]
	})
	print(mean_resultado)
	

#### Create icons using FrontAwesome
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
		'otro':                  ('fa-circle', 'lightgray')
	}
	

#### Add the respective icon to each type of crime
	def icono_delito(delito):
		if pd.isna(delito):
			return ('fa-question', 'gray')
		delito = str(delito).strip().lower()
		for key in delito_icono:
			if key in delito:
				return delito_icono[key]
		return delito_icono['otro']


####Here we generated the cluster into a Folium map, we use 500 meters because it is the average influence radio of the crimes 
	def generar_cluster_general(file_path, eps_metros=500, min_samples=3):
    df = pd.read_csv(file_path, encoding='latin-1')
	

#### Create a new DataFrame with the "X" and "Y" information and select the coords in this case is EPSG:4326 because the map was made in San Luis Potosi, Mexico. Create a list with the coords information with a LAMBDA function

	gdf = gpd.GeoDataFrame(df, geometry=gpd.points_from_xy(df.x, df.y), crs="EPSG:4326")
		gdf_utm = gdf.to_crs(epsg=32614)

		coords = gdf_utm.geometry.apply(lambda geom: (geom.x, geom.y)).tolist()
		

####Build the cluster with the data and create a DataFrame, filter the unique values.
	db = DBSCAN(eps=eps_metros, min_samples=min_samples, metric='euclidean')
	gdf_utm["cluster"] = db.fit_predict(coords)
	gdf["cluster"] = gdf_utm["cluster"]
	clusters_unicos = sorted(gdf["cluster"].unique())
	
	
#### Visualize each crime profile linking with the cluster
	for clus in clusters_unicos:
			print(f"\n{'❌' if clus == -1 else '✅'} Clúster {clus if clus != -1 else 'fuera de clúster (noise)'}:")
			subset = gdf[gdf["cluster"] == clus]
			for _, row in subset.iterrows():
				print(f"   ➤ ID: {row['ID']} | Delito: {row['Delito']} | Colonia: {row['Colonia']}")

####Creating the Plot Canvas
	fig, ax = plt.subplots(figsize=(10, 10))
`fig, ax = plt.subplots(figsize=(10, 10)):`
`plt.subplots()`: This function is the starting point for any Matplotlib figure. It creates a figure (fig) and a set of subplots (ax) with a single call.
`figsize=(10, 10)`: Sets the figure size in inches. In this case, a 10x10 inch square figure is created, which is ideal for maps as it helps maintain proper proportions and facilitates the visualization of geographical details.

####Plotting Geospatial Data 
	gdf_utm.plot(ax=ax, column="cluster", categorical=True, legend=True, cmap='tab10', markersize=30)

This is the primary plotting method for geopandas `GeoDataFrame` objects.
 `ax=ax`: Specifies that the plot should be drawn on the axes object (ax) created earlier. This ensures all elements are added to the same visualization.
`column="cluster"`: Indicates that the points (crimes) will be colored based on the values in the "cluster" column. This column is the result of the DBSCAN algorithm, where each value represents a different cluster (or noise if the value is -1).
categorical=True: Tells geopandas that the values in the "cluster" column should be treated as discrete categories, not a continuous range of numbers. This is crucial for correctly assigning colors to each cluster.
`legend=True:` Displays a legend on the map that associates each color with a cluster ID. This is fundamental for interpreting the different crime groupings.
``cmap='tab10'`: Defines the color map to use. 'tab10' is a qualitative Matplotlib colormap that provides 10 distinct colors, ideal for differentiating discrete categories. If you have more than 10 clusters, you should consider a different cmap like 'tab20' or generate a custom color palette.
`markersize=30:` Sets the size of the markers (points representing crimes) on the map. A size of 30 strikes a good balance, making the points visible without being overwhelming.

####  Map Title  
	plt.title("General Crime Clusters with DBSCAN", fontsize=14)

#### Grid Configuration 
	ax.grid(True, linestyle='--', alpha=0.5)
`ax.grid(True, linestyle='--', alpha=0.5)`
`True` Enables the display of the grid on the axes.
`linestyle='--'` Sets the gridline style to dashed (--).
`alpha=0.5`: Controls the transparency of the gridlines. A value of 0.5 makes them semi-transparent, aiding readability without saturating the map. The grid is useful for orientation and estimating relative distances on the projected map.

####Crime ID Annotation
	for idx, row in gdf.iterrows():
		ax.annotate(
			text=str(row['ID']),
			xy=(gdf_utm.geometry[idx].x, gdf_utm.geometry[idx].y),
			xytext=(3, 3),
			textcoords="offset points",
			fontsize=7,
			color='black'
		)

This loop iterates over each row of the original `GeoDataFrame (gdf)`.
ax.annotate(...): This function is used to add text annotations to a specific point on the plot.
`text=str(row['ID'])`: The text to annotate is the value from the 'ID' column for each crime. It's important to convert it to str if it isn't already.
`xy=(gdf_utm.geometry[idx].x, gdf_utm.geometry[idx].y)`: The (x, y) coordinates where the annotation will be placed. These are taken from the projected GeoDataFrame (gdf_utm) to ensure they match the point's location on the UTM map.
`xytext=(3, 3)`: Offset of the text from the xy point. A value of (3, 3) moves it slightly to the right and above the point, preventing the text from directly covering it.
`textcoords="offset points"`: Indicates that xytext is interpreted as an offset in points from xy.
`fontsize=7`: Font size of the annotation. A small size (7) is appropriate for individual IDs, preventing the map from becoming oversaturated with text.
`color='black'`: Color of the annotation text.

#### Adding the Scale Bar
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
AnchoredSizeBar(...): This is a class from matplotlib_scalebar that allows adding a scale bar to a Matplotlib figure. It is essential for maps, as it provides a visual reference for distances.
transform=ax.transData: Indicates that the scale bar should interpret units in data coordinates (UTM meters in this case).
`size=500`: The length of the scale bar in data units (500 meters).
`label='500 m'`: The text that will be displayed next to the scale bar.
`loc='lower right'`: The location of the scale bar on the plot (bottom right corner).
`pad=0.5`: Padding (space) between the plot edge and the scale bar.
`color='black'`: Color of the scale bar and its text.
`frameon=False`: Does not draw a frame around the scale bar, giving it a cleaner look.
`size_vertical=5`: The vertical thickness of the scale bar.
`fontproperties=fm.FontProperties(size=10)`: Defines the font properties for the scale bar text, such as its size.
`ax.add_artist(scalebar)`: Adds the created scale bar as an artist (element) to the axes object.

#### Axis Labels `ax.set_xlabel`, `ax.set_ylabel`

	ax.set_xlabel("UTM East (meters)")
	ax.set_ylabel("UTM North (meters)")
	
 `ax.set_xlabel `(...): Sets the label for the X-axis. On a UTM map, this represents the Easting coordinate in meters.
 `ax.set_ylabe `l(...): Sets the label for the Y-axis. On a UTM map, this represents the Northing coordinate in meters. These labels are crucial for understanding the units and projection of the map.
 
####Layout Adjustment and Plot Display `plt.tight_layout`, `plt.show`
	plt.tight_layout()
	plt.show()
`plt.tight_layout()`: Automatically adjusts subplot parameters for a tight layout. This helps prevent labels, titles, and other plot elements from overlapping.
`plt.show()`: Displays the generated figure. Without this call, the plot will not appear.

####Map Initialization and Centering
This section sets up the foundational interactive map using Folium, centering it based on the mean coordinates of your crime data in San Luis Potosí, Mexico.
	lat_mean = gdf.geometry.y.mean()
	lon_mean = gdf.geometry.x.mean()
	m = folium.Map(location=[lat_mean, lon_mean], zoom_start=13, tiles='OpenStreetMap')
	
`lat_mean = gdf.geometry.y.mean()`: Calculates the average latitude from the y coordinates of the geospatial data (gdf). This determines the central vertical point for the map's initial view.
`lon_mean = gdf.geometry.x.mean()`: Calculates the average longitude from the x coordinates of the geospatial data (gdf). This determines the central horizontal point for the map's initial view.
`m = folium.Map(...)`: Initializes the main Folium map object.
`location=[lat_mean, lon_mean]`: Sets the initial center of the map. By using the mean coordinates of your crime data, the map will automatically load focused on the area of interest (your crime dataset's geographical center).
`zoom_start=13`: Defines the initial zoom level when the map loads. A value of 13 is a good starting point, providing a balance between showing a broader area and retaining sufficient detail for urban-level analysis. You can adjust this based on the spatial spread of your data.
`tiles='OpenStreetMap'`: Specifies the base map layer. OpenStreetMap is a widely used, open-source mapping service providing detailed street-level data, which is excellent for crime mapping applications.


####Adding a Custom Map Title
This code snippet dynamically adds a custom HTML title that floats above the map, providing a clear and visually appealing header for the visualization.
	title_html = '''
				 <h3 align="center" style="font-size:16px;margin-bottom:5px;">			<b>Clústeres de Crímenes y Simbología</b></h3>
				 '''
	m.get_root().html.add_child(folium.Element(title_html))
	
`title_html = '''...'''`: Defines an HTML string for the map's title.
It uses an `<h3>` tag, which is a standard HTML heading.
`align="center"`: Centers the text horizontally.
`style="font-size:16px;margin-bottom:5px;"`: Applies inline CSS styles to control the font size (16px) and provide a small bottom margin (5px) for spacing.
`<b>Clústeres de Crímenes y Simbología</b>`: The actual title text, clearly stating the map's content in bold.
`m.get_root().html.add_child(folium.Element(title_html))`: This is the Folium method to embed custom HTML directly into the map's root HTML structure. `folium.Element()` converts the raw HTML string into a Folium-compatible element. This places the title prominently at the top of your interactive map.
#### Integrating Auxiliary Map Controls
Folium provides several useful plugins that enhance map usability. This section adds a mini-map for navigation, a measurement tool for distance calculation, and a coordinate display for precise location identification.

	minimap = MiniMap(tile_layer='OpenStreetMap', toggle_display=True, position='bottomright')
	m.add_child(minimap)

`minimap = MiniMap(...)`: Creates an instance of the MiniMap plugin. This adds a small, draggable overview map in a corner of the main map. It's incredibly useful for users to maintain spatial context when they zoom in or pan across large areas of the map.
`tile_layer='OpenStreetMap'`: Sets the base map tiles for the mini-map to OpenStreetMap for consistency with the main map.
`toggle_display=True`: Enables a button that allows users to toggle the visibility of the mini-map on or off.
`position='bottomright'`: Specifies the corner of the main map where the mini-map will be displayed.
`m.add_child(minimap)`: Adds the configured mini-map to the main Folium map object.

####Plotting Crime Points with Custom Icons and Popups

	for _, row in gdf.iterrows():
		cluster_id = row['cluster']
		cluster_color = cluster_colors[cluster_id % len(cluster_colors)] if cluster_id != -1 else 'black'
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
        icon=folium.Icon(icon=icon_name, prefix='fa', color=cluster_color), # Usa el color del cluster
        popup=folium.Popup(resumen_html, max_width=350)
    ).add_to(m)
	
`for _, row in gdf.iterrows():`: This loop iterates through each row (representing a single crime incident) in your `GeoDataFrame` (gdf).
`cluster_id = row['cluster']`: Extracts the `cluster_id` assigned to the current crime by the DBSCAN algorithm.
`cluster_color = cluster_colors[cluster_id % len(cluster_colors)] if cluster_id != -1 else 'black'`: This line dynamically determines the color of the marker based on its cluster ID:
`If cluster_id is -1` (indicating noise), it assigns the noise_color ('black').
Otherwise, it uses the modulo operator (%) to cycle through the `cluster_colors` list, ensuring that each cluster receives a distinct color from the predefined palette.
`icon_name, _=icono_delito(row['Delito'])` : Calls an external function `icono_delito` (which you would have defined elsewhere) to return a Font Awesome icon name (e.g., 'fa-mask' for 'Robo') based on the Delito (crime type) in the current row. This provides intuitive visual symbology for different crime categories.
`resumen_html = f"""..."""`: Constructs a detailed HTML string for the popup that appears when a user clicks on a marker.
It uses f-strings for easy embedding of crime attributes (ID, Delito, Colonia, Dirección, etc.) directly from the row data.
`.get("ColumnName", "N/A")`: This method safely retrieves column values, returning "N/A" if the column or value is missing, preventing errors and providing clear information.
The HTML is styled with basic CSS (font-family, font-size) for readability, and <hr> tags are used to separate sections within the popup.
`folium.Marker(...)`: Creates an individual marker object for the current crime incident.
`location=[row.geometry.y, row.geometry.x]`: Sets the marker's precise geographical location using the latitude (y) and longitude (x) from the gdf.
`icon=folium.Icon(icon=icon_name, prefix='fa', color=cluster_color):` Configures the marker's visual representation:
`icon=icon_name`: Uses the Font Awesome icon determined by icono_delito.
`prefix='fa'`: Specifies that the icon set is Font Awesome.
`color=cluster_color`: Sets the icon's background color to match its cluster, reinforcing the visual grouping.
`popup=folium.Popup`(resumen_html, max_width=350): Attaches the rich HTML resumen_html as a popup. When the marker is clicked, this detailed information window will `appear. max_width=350` ensures the popup is adequately sized.
`.add_to(m)`: Adds the fully configured marker to the main Folium map object m.

####Custom Legend with Scroll Bar
This section generates a sophisticated HTML legend that includes both cluster colors and crime type icons, designed to be visually appealing, informative, and manageable even with many entries via a scrollbar.

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
		'<i class="fa fa-venus-mars"></i> Agresión sexual<br>'
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

`legend_html = (...)`: Defines a multi-line HTML string representing the entire custom legend structure.
Positioning and Styling:
`position: fixed; bottom: 50px; left: 50px;: ` Uses fixed positioning to place the legend at a specific distance from the bottom and left edges of the map, ensuring it stays visible even when the map is scrolled.
`width: 300px; max-height: 400px; overflow-y: auto;:` Sets the legend's width, maximum height, and, crucially, adds a vertical scrollbar (overflow-y: auto) if the content exceeds the max-height. This is essential for legends with many entries, preventing them from overflowing the map.
`z-index:9999;:` Ensures the legend is rendered on top of all other map elements, making it always accessible.
Other styles like background, border, border-radius, and padding are applied for a clean, professional appearance.
"Clústeres de Crímenes" section:
Dynamically generates entries for each unique cluster ID found in your `gdf["cluster"] column (excluding -1 for noise)`.
Uses a `<span>` element styled as a small, colored circle` (border-radius:50%) `to visually represent each cluster's color.
`sorted([c for c in gdf["cluster"].unique() if c != -1])`: This ensures clusters are listed in numerical order for better readability.
`"Ruido (Clúster -1)"`: An explicit entry is added for the noise points, using the noise_color.
`'<hr>'`: Adds a horizontal rule to visually separate the cluster legend from the crime type symbology.
"Simbología de Delitos" section:
Lists common crime types with their corresponding Font Awesome icons `(<i class="fa fa-icon-name"></i>)`. This directly ties into the icono_delito function and provides a clear visual guide for understanding the markers on the map.
`Distance Scale Note`: A small, discreet message informing the user that the distance scale can be found in the lower-left corner.
`m.get_root().html.add_child(folium.Element(legend_html)):` Embeds this sophisticated custom HTML legend directly into the interactive Folium map.
