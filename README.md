# Vacation Planner
This project takes the form of three sections, detailed below.

## Weather Database - Weather_Database.ipynb

### Purpose
The Weather Database subproject uses randomized latitude and longitude coordinates to create a list of cities from the citipy library, then uses the OpenWeather API to append weather data for each valid city onto the DataFrame.  The DataFrame is the output as a *WeatherPy_Database.csv* file to be used by other subprojects.

## Vacation Search - Vacation_Search.ipynb
The Vacation Search subproject allows the user to select a temperature range and limit potential cities (from the previous project) to temperatures within that range at the current time.  Using the Google NearbySearch API, we can search for "lodging" to append the closest hotel to a city to the DataFrame, so that our consumer can know what hotel to look for.  After creating and cleaning a Hotel DataFrame, it is then output to *WeatherPy_vacation.csv*  Following that, we can then take all the hotel data and put it in a gmaps figure with markers and info-boxes.

![Hotels Marked, Global](/Vacation_Search/WeatherPy_vacation_map.png)

## Vacation Itinerary - Vacation_Itinerary.ipynb
The final subproject involves creating an itinerary of 4 nearby hotels, including the directions to get to and from each one.
### After inputting the results of Vacation Search, a confirmation figure is created to confirm the results of the previous Vacation Search:
![Hotels Marked, Global](/Vacation_Itinerary/WeatherPy_vacation_map.png)
### After selecting 4 cities, a direction-layer map is created to show a theoretical circuitious path between the cities.
![Directions-Layer](/Vacation_Itinerary/WeatherPy_travel_map.png)
### The four cities, with hotel info-boxes, are also displayed separately.
![Hotel Inforboxes](/Vacation_Itinerary/WeatherPy_travel_map_markers.png)

### *Completely Extraneous Bonus Project*
Rather than manually "selecting" 4 cities and using those, I wrote a function (using a class) to cluster the cities of a particular country into the tightest group of ***n*** cities that I could.  There are still a few caveats (such as directions between cities separated completely by a body of water, such as Hawaiian islands), and if this implementation is explored further, it should definitely be refactored to dynamically store distance data rather than constantly recalculating redundant distance data on the order of O(n<sup>2</sup>log(n)).  The main code for that is below, and there are additional comments there and in the implementation in the main file.
```
select_country = "In"
country_df = vacation_df[vacation_df["Country"] == select_country]
#Set the desired number of cities
group_size = 4

country_df["City_ID"] = country_df["City_ID"].astype(str)

# Class to hold a number of cities in a group, with a centralized latitude/longitude
class my_city_group:
    def __init__ (self, city_ids, lat, lng):
        self.city_ids = []
        for i in city_ids:
            self.city_ids.append(i)
        self.lat = lat
        self.lng = lng
    
    # Return the distance between two centroids of city groups
    def dist(self, city_group2):
        x = self.lng - city_group2.lng
        y = self.lat - city_group2.lat
        return(((x**2) + (y**2))**0.5)
    
    # Combine two city groups into one, merging the city_ID lists into one and using
    # a weighted average to calculate a new centroid.
    def combine(self, city_group2):
        len1 = len(self.city_ids)
        len2 = len(city_group2.city_ids)
        new_ids = []
        for i in self.city_ids:
            new_ids.append(i)
        for i in city_group2.city_ids:
            new_ids.append(i)
        avg_lat = (self.lat * len1 + city_group2.lat * len2) / (len1 + len2)
        avg_lng = (self.lng * len1 + city_group2.lng * len2) / (len1 + len2)
        return my_city_group(new_ids, avg_lat, avg_lng)

def get_closest_cities(df, group_size):
    hotel_count = df["Country"].count()
    # Verify that there are at least group_size number of hotels in the database
    if hotel_count < group_size:
        country = list(df["Country"])[0]
        err = f"ERROR: There are only {hotel_count} hotels in " \
                f"country {country}, but {group_size} were requested."   
        print(err)
        return ""
    cities = []
    # initialize city list
    for i in df.index:
        new_city = my_city_group([df.loc[i, 'City_ID']],
                                  df.loc[i, 'Lat'],
                                  df.loc[i, 'Lng'])
        cities.append(new_city)
    # initialize the current largest group size
    lar_group = 1
    while(lar_group < group_size):
        # Maximum distance is, in theory, sqrt(360^2 + 180^2), assuming we don't account for the international dateline
        min_dist = 403
        # Cycle through each city grouping
        for i, city1 in enumerate(cities):
            # Cycle through each city grouping again
            for j, city2 in enumerate(cities):
                # Ensure we're not comparing the same group
                if i != j:
                    dist = city1.dist(city2)
                    # Find the minimum distance for this cycle
                    if dist < min_dist:
                        min_dist = dist
                        min_cities = [j, i]
        # Make a new group of the two closest city groups, combined
        new_group = cities[min_cities[0]].combine(cities[min_cities[1]])
        lar_group = len(new_group.city_ids)
        # Remove the two previous city groups that were combined before adding the new one
        # Since min_cities is returned as [j ,i], after finding the first combination of those two city groups,
        # the first one (larger index) should be popped first, to preserve the index order for the second
        cities.pop(min_cities[0])
        cities.pop(min_cities[1])
        cities.append(new_group)
    return new_group
    
cluster = get_closest_cities(country_df, group_size)
```
