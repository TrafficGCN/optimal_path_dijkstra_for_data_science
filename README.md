# Plotting the Optimal Route in Python for Data Scientists using the Dijkstra Algorithm

<img src="https://github.com/ThomasAFink/optimal_path_dijkstra_for_data_science/blob/main/output/dijkstra_map.jpg?raw=true" width="450" align="right">

The following example uses [OSMnx](https://osmnx.readthedocs.io/en/stable/) to generate the optimal path between two geocoordinates and between one and many points and plotting the path(s) on a [Plotly](https://plotly.com/) map. This is a quick and simple example that data scientists can use to illustrate a path(s) for their professional or academic papers.

A brief file structure overview of the repository is provided. The dijkstra_map.py is in the root directory. The data folder houses a list of target geocoordinates in a csv file. The out map is generated in the output folder.

    /
    dijkstra_map.py

    - / data /
    geocoordinates.csv

    - / output /
    dijkstra_map.jpg
    
The geocoordinates.csv includes the following target geocoordinates.

    LATITUDE	LONGITUDE
    48.13485	11.5173913
    48.1348182	11.577103
    48.1492002	11.5592469
    48.11005	11.59344
    48.158903	11.5856
    
Before jumping into the code the following requirements and packages are needed to run the code:

    Python 3.10.6
    pip3 install osmnx==0.16.1 
    pip3 install geopandas==0.9.0
    pip install shapely==1.8.0
    pip3 install -U kaleido
    pip3 install networkx
    pip3 install plotly
    pip3 install pandas
    
First the packages that were just installed are imported into our file dijkstra_map.py

    import networkx as nx
    import plotly.graph_objects as go
    import osmnx as ox
    import pandas as pd
    import geopandas

Next a function is created which returns a list of geocoordinates belonging to the optimal path between two geocoordinates. The function is needed to iterate through several target points. The function takes three parameters an origin coordinate, a target coordinate, and a perimeter around those coordinates. The perimeter should be larger for points that a further from another, but a small as possible to keep processing time practical. For plotting routes in a city a perimeter value between 0.10 and 0.20 is should suffice.

    ##### Interface to OSMNX    
    def generate_path(origin_point, target_point, perimeter):
    
Inside the function a cache configuration is an optional method to store maps from OSM. This is especially resourceful for storing larger maps as it requires fewer requests to the api. For plotting a handful of paths this really does not impact processing time sigificantly.

    # Using the cache accelerates processing for a large map
    ox.config(log_console=True, use_cache=True)
    
Next inside the function the gecoordinates are spliced and the underlying structure of the graph network is set up. If the origin point is further from the equator than the target point, then the path starts from the north and ends in the south (and viceversa). If the origin point is further from the prime meridian than the target point, then the path starts in the east and ends in the west and (viceversa). This [logic](https://github.com/jasonmanesis/Vehicle-Navigation-Shortest-Path/blob/main/vehicle_navigation_shortest_path.py) was provided by [jasonmanesis](https://github.com/jasonmanesis) on [GitHub](https://github.com/jasonmanesis/Vehicle-Navigation-Shortest-Path).

    # Splice the geographical coordinates in long and lat
    origin_lat = origin_point[0]
    origin_long = origin_point[1]

    target_lat = target_point[0]
    target_long = target_point[1]

    # Build the geocoordinate structure of the path's graph

    # If the origin is further from the equator than the target
    if  origin_lat > target_lat:
        north = origin_lat 
        south = target_lat
    else:
        north = target_lat
        south = origin_lat

    # If the origin is further from the prime meridian than the target
    if  origin_long > target_long:
        east = origin_long 
        west = target_long
    else:
        east = target_long
        west = origin_long
After the underlying graph structure is determined the graph mode is set to drive. Drive captures the road network used by automobiles. Bike capture the path network used by bicycles. Walk capture the walkway network used by pedestrians. Finding an optimal walkway is usually too resources intensive when processing due to the fact that their are many more nodes on the walking graph.

    # Construct the road graph
    # Modes 'drive', 'bike', 'walk' (walk is usually too slow)
    mode = 'drive'
Finally the graph can be requested and downloaded from the OSMnx api. The graph perimeters are set in the parameters and passed to the api request. The graph mode is also passed along.

    # Create the path/road network graph via setting the perimeters
    roadgraph = ox.graph_from_bbox(north+perimeter, south-perimeter, 
    east+perimeter, west-perimeter, network_type = mode, simplify=False)
Alternatively a map can also be requested via the place instead of using coordinates. This may be useful for requesting larger maps and storing them locally in the cache.

    '''
    # Alternatively a road network can be determined via providing a place
    place  = 'Munich, Bavaria, Germany'
    roadgraph = ox.graph_from_bbox(place, network_type = 'drive', simplify=False )
    '''
Next the nearest node in the graph network is found for both the origin and target coordinates. It could for example be that the origin or target does not fall on the road, bike, or walk grid in the OSMnx graph network.

    # Get the nearest node in the OSMNX graph for the origin point
    origin_node = ox.get_nearest_node(roadgraph, origin_point) 

    # Get the nearest node in the OSMNX graph for the target point
    target_node = ox.get_nearest_node(roadgraph, target_point)
Using the shortest_path function provided by NetworkX the optimal route is found using the dijkstra as the method and the length between the nodes as the weight. Another method is the bellman-ford algorithm.

    # Get the optimal path via dijkstra
    route = nx.shortest_path(roadgraph, origin_node, target_node, weight =
    'length', method='dijkstra')
Finally the optimal path and its geocoordinates are returned via the route variable and stored in longitude and latitude arrays.

    # Create the arrays for storing the paths
    lat = []
    long = []

    for i in route:
        point = roadgraph.nodes[i]
        long.append(point['x'])
        lat.append(point['y'])
To conclude the function the optimal path is then returned as two arrays, longitude and latitude.

    # Return the paths
    return long, lat
Now that the function for finding the optimal paths between two points generate_path(origin_point, target_point, perimeter) is setup a function to plot the results on the Plotly map is created. This function takes the origin point the target points and the optimal paths provided by two lists as longitude and latitude.

def plot_map(origin_point, target_points, long, lat):
First inside the function a Plotly map figure is created and the origin point is defined as black marker.

    # Create a plotly map and add the origin point to the map
    print("Setting up figure...")  
    fig = go.Figure(go.Scattermapbox(
        name = "Origin",
        mode = "markers",
        lon = [origin_point[1]],
        lat = [origin_point[0]],
        marker = {'size': 16, 'color':"#333333"},
        )   

    )
Next the optimal paths are added to the Plotly map figure as a yellow color.

    # Plot the optimal paths to the map
    print("Generating paths...")   
    for i in range(len(lat)):
        #print(lat[i])
        #print(long[i])
        fig.add_trace(go.Scattermapbox(
            name = "Path",
            mode = "lines",
            lon = long[i],
            lat = lat[i],
            marker = {'size': 10},
            showlegend=False,
            line = dict(width = 4.5, color = '#ffd700'))
        )
The target points are also added as yellow markers.

    # Plot the target geocoordinates to the map
    print("Generating target...")  
    for target_point in target_points:
        fig.add_trace(go.Scattermapbox(
            name = "Destination",
            mode = "markers",
            showlegend=False,
            lon = [target_point[1]],
            lat = [target_point[0]],
            marker = {'size': 16, 'color':'#ffd700'}))
Next the style of the map layout is determined. The light theme from Mapbox was used. An API access token from [Mapbox](https://www.mapbox.com/) can be acquired for free after [signing up](https://account.mapbox.com/auth/signup/). Mapbox has many beautiful maps from a variety of contributors. These look nice in academic papers. Remember to attribute the map creator. Times New Roman was used and the shape was set to a square.

    # Style the map layout
    fig.update_layout(
        mapbox_style="light",       
        mapbox_accesstoken="############",
        legend=dict(yanchor="top", y=1, xanchor="left", x=0.83), #x 0.9
        title="<span style='font-size: 32px;'><b>The Shortest Paths Dijkstra
        Map</b></span>",
        font_family="Times New Roman",
        font_color="#333333",
        title_font_size = 32,
        font_size = 18,
        width=1000, #2000
        height=1000,
    )
Next the center of the map was manually added. There are methods for calculating this, but not included in this tutorial. An easy way to determine the map center is to look at Google Maps and gauge it that way. It may be better to include the map’s center with the function parameters.

    # Set the center of the map
    lat_center = 48.14
    long_center = 11.57

    # Add the center to the map layout
    fig.update_layout(margin={"r":0,"t":0,"l":0,"b":0},
        title=dict(yanchor="top", y=.97, xanchor="left", x=0.03), #x 0.75
        mapbox = {
            'center': {'lat': lat_center, 
            'lon': long_center},
            'zoom': 12.2}
    )
Finally the function saves the map as a jpg in the output folder as dijkstra_map.jpg and then generates an interactive Plotly map in the default OS browser.

    # Save map in output folder
    print("Saving image to output folder...");
    fig.write_image('output/dijkstra_map.jpg', scale=3)

    # Show the map in the web browser
    print("Generating map in browser...");
    fig.show()
Now that both functions are set up the main section of the script is written.

First in the main part of the script a list of target geocoordinates are fetched from the data folder in the geocoordinates.csv file and loaded into python using pandas’ dataframe method. The geocoordinates are then formatted as geocoordinates using geopandas.

    # Data import path
    SENSORS_CSV   = 'data/geocoordinates.csv'

    # Data Import
    df1 = pd.read_csv(SENSORS_CSV)

    # Keep only relevant columns
    df = df1.loc[:, ("LATITUDE", "LONGITUDE")]

    # Create point geometries
    geometry = geopandas.points_from_xy(df.LONGITUDE, df.LATITUDE)
    geo_df = geopandas.GeoDataFrame(df[['LATITUDE', 'LONGITUDE']], geometry=geometry)

    # Format the target geocoordinates from the csv file
    target_points = []
    for lo, la in zip(df["LONGITUDE"], df["LATITUDE"]):
       print(lo)
       target_points.append((la,lo))

Next the origin point is defined.

    # Set the origin geocoordinate from which the paths are calculated
    origin_point = (48.1372038, 11.565651)  
    Lists are declared to store the optimal paths.

    # Create the lists for storing the paths
    long = []
    lat = []

Using the list of target points from the data folder our generate_path function is called to return an optimal path for every target point. The perimeter can also be set here.

    i = 0
    for target_point in target_points:

        # Perimeter is the scope of the road network around a geocoordinate
        perimeter = 0.10

        # Process the optimal path
        print("Processing *************************************** " + str(i))
        x, y = generate_path(origin_point, target_point, perimeter)
    
        # Append the paths
        long.append(x)
        lat.append(y)

        i += 1

Finally our plot_map function is called after getting the optimal paths.

    plot_map(origin_point, target_points, long, lat)

Generating optimal paths may be particularly helpful for example when visualising and understanding graph neural networks, or GNNs, in for traffic forecasting.
