# Projet Python : Analyse des Espaces Urbains et Verts à Tanger

Ce projet Python utilise des scripts pour collecter, analyser, et visualiser les données géographiques de la ville de Tanger. À l'aide des données d'OpenStreetMap (OSM) et de diverses bibliothèques Python comme **osmnx**, **folium**, et **pandas**, nous analysons la répartition des espaces verts et urbains.


import osmnx as ox
import geopandas as gpd
import pandas as pd
from pyproj import CRS

# Définir la zone d'étude
city = "Tanger, Morocco"

# Tags pour les espaces verts
tags = {
    'leisure': ['park', 'garden', 'nature_reserve', 'playground'],
    'landuse': ['grass', 'forest', 'meadow', 'recreation_ground'],
    'natural': ['wood', 'grassland', 'scrub']
}

print("Collecting green spaces...")

# Créer une zone tampon autour de la ville
gdf = ox.geocode_to_gdf(city)
gdf = gdf.to_crs(32630)  # Projeter en UTM Zone 30N
gdf['geometry'] = gdf.geometry.buffer(3000)  # Tampon de 3 km
gdf = gdf.to_crs(4326)  # Revenir à WGS84

# Collecter les données pour chaque tag
all_geometries = []
for key, values in tags.items():
    for value in values:
        try:
            geom = ox.features_from_polygon(
                gdf.iloc[0].geometry, tags={key: value}
            )
            if not geom.empty:
                all_geometries.append(geom)
            else:
                print(f"No features found for {key}={value}.")
        except Exception as e:
            print(f"Error for {key}={value}: {e}")

# Fusionner toutes les géométries collectées
if all_geometries:
    espaces_verts = gpd.GeoDataFrame(pd.concat(all_geometries, ignore_index=True))

    # Projeter en UTM Zone 30N
    espaces_verts = espaces_verts.to_crs(32630)

    # Calculer la superficie en km²
    espaces_verts['Superficie'] = espaces_verts.geometry.area / 1_000_000

    # Calculer les centroïdes dans le système de coordonnées projeté
    espaces_verts['centroid'] = espaces_verts.geometry.centroid

    # Reprojeter les centroïdes en WGS84 pour obtenir la latitude et la longitude
    centroids_wgs84 = espaces_verts.set_geometry('centroid').to_crs(4326)

    # Extraire les informations nécessaires
    data = {
        "Nom": espaces_verts.get("name", "Espace vert"),
        "Latitude": centroids_wgs84.geometry.y,
        "Longitude": centroids_wgs84.geometry.x,
        "Superficie": espaces_verts["Superficie"]
    }

    # Créer le DataFrame
    df_espaces_verts = pd.DataFrame(data)

    # Remplacer les valeurs NaN de la colonne 'Nom' par 'nature_reserve'
    df_espaces_verts["Nom"] = df_espaces_verts["Nom"].fillna("nature_reserve")

    # Sauvegarder les données dans un fichier CSV
    df_espaces_verts.to_csv("/content/sample_data/espaces_verts.csv", index=False)
    print("Green spaces collected and saved.")
else:
    print("No green spaces found for the specified tags.")
