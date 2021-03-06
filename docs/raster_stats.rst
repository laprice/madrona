.. _raster_stats:

``madrona.raster_stats`` -  Raster Statistics
==============================================

The `madrona.raster_stats` app allows you to analyze raster datasets based on a polygon geometry. For example, for a given polygon, you may want to know the average elevation or the maximum temperature, etc. Essentially this performs a vector-on-raster intersection. 

The app uses an optional caching mechanism so that any geometry/raster combination only needs to be run once. 

The raster_stats app provides a model to register raster datasets, a utility function to quickly calculate stats programatically and a webservice which returns the statistics in json format.

Installation
------------
The raster_stats app relies on the `starspan` executable. This C++ app must be installed from src and requires GDAL and GEOS header files::

    hg clone https://code.google.com/p/starspan/
    cd starspan
    ./configure
    make
    sudo make install

The above technique should be sufficient if you have GEOS 3.2.2 or earlier. However, starspan uses geos C++ api which is a moving target and the 3.3 version of GEOS is not fully compatible with starspan.
This means if your system is build on GEOS 3.3+, you must compile starspan against seperate versions of GEOS 3.2.2 and GDAL 1.8.1 and keep them isolated (the starspan shell script uses environment variables to ensure the correct libaries are loaded at runtime). The procedure below should cover all the bases::

    #geos 3.2.2 
    cd /usr/local/src
    wget http://download.osgeo.org/geos/geos-3.2.2.tar.bz2
    tar -xjvf geos-3.2.2.tar.bz2
    cd geos-3.2.2
    ./configure --prefix=/usr/local/starspan
    make
    sudo make install

    #gdal 1.8.1
    cd /usr/local/src
    wget http://download.osgeo.org/gdal/gdal-1.8.1.tar.gz
    tar -xzvf gdal-1.8.1.tar.gz
    cd gdal-1.8.1
    ./configure --prefix=/usr/local/starspan --with-geos=/usr/local/starspan/bin/geos-config
    make
    sudo make install

    #starspan 1.2.06
    cd /usr/local/src
    hg clone https://code.google.com/p/starspan/
    cd starspan 
    ./configure --with-gdal=/usr/local/starspan/bin/gdal-config --with-geos=/usr/local/starspan/bin/geos-config --prefix=/usr/local/starspan
    make
    sudo make install

    # settings.py
    # Finally, in settings_local.py, set STARSPAN_BIN to '/usr/local/starspan/bin/starspan'

Settings
--------
You can set the `STARSPAN_BIN` setting to point to your starspan executable and the `RASTER_DIR` setting for the filepath to the directory containing your raster files.

If you want to keep the temporary files around after starspan is done, set `STARSPAN_REMOVE_TMP` to False (default is True which clears disc space after each run).

Then just add to your installed apps::

    INSTALLED_APPS += ( 'madrona.raster_stats' )

Adding a Raster
---------------
Simple as it gets. Type is used to define the raster as `continuous` (e.g. elevation)::
    
    from madrona.raster_stats.models import RasterDataset
    elev = RasterDataset.objects.create(name="test_elevation",filepath='/tmp/data/elevation.tif',type='continuous')  

or discrete categorical values (e.g. land use)::

    land = RasterDataset.objects.create(name="landuse",filepath='/tmp/data/landuse.tif',type='categorical')  

Running zonal stats
-------------------
Use the utility function which will first check the cache. If nothing is in the cache, the full starspan analysis is run. Otherwise the cache hit is returned::

    from madrona.raster_stats.models import zonal_stats
    stats = zonal_stats(geos_polygon_geom, elev)
    stats.from_cache # False
    
    stats = zonal_stats(geos_polygon_geom, elev)
    stats.from_cache # True
    stats.min 
    stats.max
    stats.median
    stats.mode
    stats.nulls
    stats.pixels
    stats.avg
    stats.stdev


Categorical rasters
-------------------
In addition to the zonal statistics calculated for a continuous raster, categorical rasters can access the pixel counts for each discrete category of raster values::

    stats = zonal_stats(geos_polygon_geom, landuse)
    total_pixels = stats.pixels
    stats.categories.all() # returns a queryset of ZonalCategories
    for cat in stats.categories.all():
        print "Category", cat.category, "has", cat.count, "pixels out of a total of", total_pixels
        # ex: "Category 42 has 1866 pixels out of a total of 7866"

It is the programmers responsibility to account for mapping the category raster code to a meaningful category name (i.e. 42 == 'Douglas Fir') as well as handling any null cells that might affect the total pixel count; check `stats.nulls` and adjust accordingly. For example if stats.pixels == 7866 and stats.nulls == 1000, you may consider the total pixel count to be 6866 depending on your analysis needs.

Specifying the pixel proportion
-------------------------------
Starspan allows you to define the threshold of cell inclusion based on the percentage of the pixel that is covered by the polygon. By default, a raster cell is included if the geometry overlaps >= 50% of the cell. You can adjust this value by assigning an alternate `pixprop` value between 0 and 1::

    stats = zonal_stats(geos_polygon_geom, landuse, pixprop=0.85) # cell must be 85% covered to be included

Using the web service
---------------------
The app provides a urls.py file; just point your main URLCONF file to it::

    (r'^zonal/', include('madrona.raster_stats.urls')),

You can get a json list of the rasters at this url::

	http://localhost/zonal/

And you can append the raster name and supply a `geom_txt` parameter (either wkt or json) which returns the rasters stats as json::

	http://localhost/zonal/sst/?geom_txt=POLYGON ((-122.735420504497029 37.238868044757552,-122.516579972608298 37.245550198403009,-122.50822728055148 37.043415050627928,-122.730408889262932 37.046756127450656,-122.735420504497029 37.238868044757552))

	[
         {"pk": 764, "model": "raster_stats.zonalstatscache", 
          "fields": {"raster": 23, "min": 0.0, "max": 1.5440739999999999, "geom_hash": "-8107990604081680573", 
                     "nulls": 0.0, "median": 0.28777199999999997, "mode": 0.0, "stdev": 0.44484400000000002, 
                     "date_modified": "2010-06-23 19:00:30", "avg": 0.40776400000000002, "pixels": 531.0}
         }
        ]
