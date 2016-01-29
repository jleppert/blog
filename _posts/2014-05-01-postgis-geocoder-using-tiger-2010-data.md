---
layout: post
title: "PostGIS Geocoder using Tiger Data"
description: "Build your own geocoder"
tags: [geo, postgres, postgis]
oldurl: /postgis-geocoder-using-tiger-2010-data
---

What happens when you have half a million addresses and need to find out where in the world they actually are? You geocode them! Which isn’t the easiest thing to do — the difficulty increasing as your need for performance, quality and quantity goes up.

##Geocoding Options

The first, and probably the most obvious, is to simply use an external service. There are plenty to choose from, with many major companies offering up their Geocoder API for your personal enjoyment (or dis-enjoyment). So pick your poison — Google, Yahoo, Bing all have nice API’s that will happily take an address and give you something that looks like a lat and long.

Of course, can you really make 500,000 web requests and no one be the wiser? Of course not. There are limitations, both on the service and potentially on your process to make these requests. For one, it’s pretty expensive for most API’s I looked at to purchase a commercial license suitable for bulk geocoding. But it’s also slow. And in slow I mean it takes time to make 500,000 web requests, wait for them to complete, and put the data some place. That’s not counting the fact you’ll need to somehow address failures, which probably means some sort of queue/worker system, that’ll need supervision, management, et al. Yeah, not fun. So even if you manage to find an awesome deal on 500k web requests to someone’s API, you still need to manage the carrier pigeon effect. At least until connections get faster and it suddenly becomes cheap to buy expensive online data and services.

It’s also worth mentioning that many of these providers rely on the same sources of data that are freely available. In my tests, there were only marginal differences in quality across various solutions (both commercial API, private sources of data and home-brew systems). The largest factor in geocoding appears to be address quality first, and your data source second (and how well your parser can understand bad addresses). Soundex helps with address quality, but some addresses data is just bad and I suspect Google and others use some form of dynamic analysis based upon past searches to obtain higher quality and suggestions for incorrect cities, zip codes, etc. Here’s an [interesting report comparing various geocoding options](http://spatial.usc.edu/Users/dan/gislabtr10_Eight-Commonly-Used-Geocoding-Systems.pdf) and the effectiveness of each based upon testing with 100 addresses (pay close attention to shapefile based data sources).

With these facts of life in mind, I decided what any crazy person would — setup my own! Google be damned. I can do it, and in one week. Well, maybe two. I know what you’re thinking, pretty foolish and maybe even a little masochistic, and you’d be right on both counts.

Fortunately, PostGIS has in it’s latest version, buried under extras a [quite comprehensive geocoder project](http://postgis.refractions.net/documentation/manual-svn/Extras.html#Tiger_Geocoder) based upon freely available [US Census Shapefile data](http://www.census.gov/geo/www/tiger/). I’ll walk you through setting up your own geocoder, complete with soundex support and ability to bulk geocode from a db table.


##Postgres Setup

*If you already have Postgres setup, you can skip or skim over these steps. Just make sure you have a working installation and now the paths to all the relevant bits where things are installed*

###Download & Install Postgres

For maximum flexibility in the initial prototypying, I decided to compile Postgres from source, but most distributions also have a package available and Postgres maintains binary and graphical installer packages for some. To compile:

*You’ll need build-essential, gcc, etc. packages to compile*

{% highlight bash %}
export POSTGRES_VERSION=9.0.4
wget http://wwwmaster.postgresql.org/redir/180/f/source/v$POSTGRES_VERSION/postgresql-$POSTGRES_VERSION.tar.gz
tar -xvzf postgresql-$POSTGRES_VERSION.tar.gz
cd postgresql-$POSTGRES_VERSION
./configure --prefix=/usr/local
make
sudo make install
{% endhighlight %}

You’ll now need to compile and install PostGIS. Before you do this, it’s a good idea to spend some time setting up accounts on Postgres. The authentication model for Postgres relies on system accounts, specifically allowing root access to the postgres user by default, where you can administer other users as necessary. For my setup, I made sure to have a system account for every postgres user.

Create the postgres user:

{% highlight bash %}
sudo adduser postgres
{% endhighlight %}

You now need to create a directory to store the main Postgres DB files. I put mine under /usr/local, but you can put it generally anywhere as needed (the path is specified in configuration and/or during server startup).

{% highlight bash %}
sudo mkdir /usr/local/pgsql/data
sudo chown postgres /usr/local/pgsql/data
sudo chgrp postgres /usr/local/pgsql/data
{% endhighlight %}

Now you’ll need to create a postgres cluster and start the postmaster process. First, change to the postgres user:

{% highlight bash %}
su postgres
/usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data
/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l /var/log/postgres start
{% endhighlight %}

You can now login using the psql client (as the postgres system user):

{% highlight bash %}
/usr/local/pgsql/bin/psql
{% endhighlight %}

If all is well, you should be at a postgres command prompt like the following:
{% highlight bash %}
psql (9.0.4)
Type "help" for help.


postgres=#
{% endhighlight %}

To quit, type “\q”. Now we’re ready to install and setup PostGIS!

##Setup PostGIS

*You’ll need libxml2-dev, libgeos-dev and libproj-dev (xslt and convert are also needed for documentation).* 
I had to manually compile and install geos (which you can get from [osgeo](http://trac.osgeo.org/geos/)).

Since we’re interested in the Geocoder from 2.0, we’ll need to grab the latest development snapshot:

{% highlight bash %}
wget http://postgis.refractions.net/download/postgis-2.0.0SVN.tar.gz
tar -xvzf postgis-2.0.0SVN.tar.gz
./configure --with-pgconfig=/usr/local/pgsql/bin/pg_config
make
make install
ldconfig
{% endhighlight %}

###Create a spatial database template

So we can easily create spatially enabled databases, we’ll create one as a template and apply the spatial data types to it:

{% highlight bash %}
su postgis
/usr/local/pgsql/bin/createdb spatial_db_template
/usr/local/pgsql/bin/createlang plpgsql spatial_db_template
/usr/local/pgsql/bin/psql -d spatial_db -f /usr/local/pgsql/share/contrib/postgis-2.0/postgis.sql
/usr/local/pgsql/bin/psql -d spatial_db -f /usr/local/pgsql/share/contrib/postgis-2.0/spatial_ref_sys.sql
{% endhighlight %}

Now to create a new postgis enabled spatial database, we just do the following:

{% highlight bash %}
/usr/local/pgsql/bin/createdb -T spatial_db_template tiger
{% endhighlight %}

###Add soundex ability

To add soundex ability to the db (used in matching parsed address parts which includes soundex, metaphone and levenshtein abilities), first build the module under the contrib directory of your postgres source:

{% highlight bash %}
cd postgresql-$POSTGRES_VERSION
cd contrib/fuzzystrmatch
sudo make && make install
{% endhighlight %}

This will add the fuzzystrmatch under the contrib folder of your installation. To add it to your database:

{% highlight bash %}
/usr/local/pgsql/bin/psql -d tiger -f /usr/local/pgsql/share/contrib/fuzzystrmatch.sql
{% endhighlight %}

###Create a geocoder database

Now, we’ll need to create a new geocoder database:

{% highlight bash %}
/usr/local/pgsql/bin/createdb -T spatial_db_template geocoder
/usr/local/pgsql/bin/psql -d tiger -f /usr/local/pgsql/share/contrib/fuzzystrmatch.sql
/usr/local/pgsql/bin/psql -d geocoder
CREATE SCHEMA tiger;
CREATE SCHEMA tiger_data;
ALTER DATABASE geocoder SET search_path=public, tiger;
{% endhighlight %}

Now, we can begin to setup our geocoder. We first need to populate some street abbreviation and state lookup tables, and the tiger_loader.sql, which is the actual code to load tiger data into the geocoder table structure:

(I had my PostGIS sources under /tmp in this case):

{% highlight bash %}
/usr/local/pgsql/bin/psql -d geocoder -f /tmp/postgis-2.0.0SVN/extras/tiger_geocoder/tiger_2010/tables/lookup_tables_2010.sql
/usr/local/pgsql/bin/psql -d geocoder -f /tmp/postgis-2.0.0SVN/extras/tiger_geocoder/tiger_2010/tiger_loader.sql
{% endhighlight %}

Next, create the actual geocode function in the db (note you need to be in the tiger_2010 directory):

{% highlight bash %}
/usr/local/pgsql/bin/psql -d geocoder -f ./create_geocode.sql
{% endhighlight %}

You’re now ready to load US Census shapefile data into the geocoder! There’s a slightly contrived way of generated scripts from the db, but I found this cumbersome for loading many states, so I wrote my own with perl. Someone from the PostGIS mailing list also wrote their own:

{% gist forkandwait/885803 %}

For the original loader script and instructions, [check the original documentation](http://postgis.refractions.net/documentation/manual-svn/Loader_Generate_Script.html).

I found it useful to get all the files into a single directory, and then just process them from there (instead of having a separate script to do each state):

{% highlight bash %}
wget http://www2.census.gov/geo/pvs/tiger2010st/ --no-parent --relative --recursive --level=2 --accept=zip,txt --mirror --reject=html
{% endhighlight %}

The loading process takes anywhere from 30 minutes to over an hour, depending on how many points of interest are in the shape file and the complexity of the generated sql. Once it’s finished, it’s quite simple to geocode addresses with pretty good quality:

{% highlight bash %}
/usr/local/pgsql/bin/psql -d geocoder
geocoder=# SELECT g.rating, ST_X(geomout) AS lon, ST_X(geomout) AS lat, (addy).* FROM geocode('2500 Farmers Rd Columbus, Ohio') as g;
rating | lon | lat | address | predirabbrev | streetname | streettypeabbrev | postdirabbrev | internal | location | stateabbrev | zip | parsed
--------+------------+------------+---------+--------------+------------+------------------+---------------+----------+--------------+-------------+-------+--------
8 | -83.784252 | -83.784252 | 2500 | | Farmers | Rd | | | Sabina | OH | 45177 | t
9 | -83.784252 | -83.784252 | 2500 | | Farmers | Rd | | | Wilmington | OH | 45177 | t
11 | -83.784252 | -83.784252 | 2500 | | Farmers | Rd | | | Port William | OH | 45177 | t
13 | -83.087185 | -83.087185 | 2500 | | Farmers | Dr | | | Columbus | OH | 43235 | t
(4 rows)
{% endhighlight %}


What’s even more interesting is bulk geocoding by loading an entire table of addresses and selecting them back out (or to another table) geocoded. Here’s some code that does that (modified from (http://postgis.refractions.net/documentation/manual-svn/Geocode.html)):

{% highlight bash %}
CREATE TABLE addresses_to_geocode(addid serial PRIMARY KEY, address text,
lon numeric, lat numeric, new_address text, rating integer);


INSERT INTO addresses_to_geocode(address)
VALUES ('529 Main Street, Boston MA, 02129'),
('77 Massachusetts Avenue, Cambridge, MA 02139'),
('28 Capen Street, Medford, MA'),
('124 Mount Auburn St, Cambridge, Massachusetts 02138'),
('950 Main Street, Worcester, MA 01610');


-- only update the first two addresses (850 ms) --
-- for large numbers of addresses you don't want to update all at once
-- since the whole geocode must commit at once
UPDATE addresses_to_geocode
SET (rating, new_address, lon, lat)
= (g.rating, pprint_addy(g.addy),
ST_X(g.geomout), ST_Y(g.geomout) )
FROM (SELECT DISTINCT ON (addid) addid, (g1.geo).*
FROM (SELECT addid, (geocode(address)) As geo
FROM addresses_to_geocode As ag
WHERE ag.rating IS NULL ) As g1
ORDER BY addid, rating LIMIT 2) As g
WHERE g.addid = addresses_to_geocode.addid;

result
-----
2 rows affected, 850 ms execution time.

SELECT * FROM addresses_to_geocode WHERE rating is not null;


addid | address | lon | lat | new_address | rating
------+----------------------------------------------+-----------+----------+-------------------------------------------+--------
1 | 529 Main Street, Boston MA, 02129 | -71.07187 | 42.38351 | 529 Main St, Boston, MA 02129 | 0
2 | 77 Massachusetts Avenue, Cambridge, MA 02139 | -71.09436 | 42.35981 | 77 Massachusetts Ave, Cambridge, MA 02139 | 0
{% endhighlight %}

##Performance Considerations

What’s it like to use it for real life stuff? Well, your mileage may vary. I loaded all US States and performance ranged anywhere from 40 ms to over 3000 ms for some difficult addresses on a dual quad core machine with 32 GB of ram and fast disks. Part of the complexity is in the address parsing and attempting to match address parts to many tables, resulting in table scans and a lot of time and disk complexity. If you have many millions of addresses to geocode, this could be a problem.

I was able to increase performance by “pre-normalizing” my addresses, and then sorting them by zip code. This seemed to help the most, along with increases shared memory cache in Postgres configuration. I’m also looking at optimizing away some unnecessary table scans, and trying to make more use of partitioned tables and different indexes. So far, it’s worked for my purposes, and I haven’t tried to cluster it yet, which may be an option for some (e.g. partition multiple states or sets of data over many machines).

Happy geocoding!
