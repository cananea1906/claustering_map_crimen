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
