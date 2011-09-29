---
layout: post
title: Generating Maps with Mapnik
categories:
- GIS
---

{{ page.title }}
================

Installation
------------

I followed [this excellent
tutorial](http://weait.com/content/build-your-own-openstreetmap-server)
for the bulk of the install process. Some changes I made were
installing `Mapnik2` instead of `release-0.7.1`. For that I had to go
[here](http://trac.mapnik.org/wiki/UbuntuInstallation) (I'm on Ubuntu
11.04) and [here](http://trac.mapnik.org/wiki/Mapnik2).

The rest of the tutorial is nearly perfect. One thing to watch out for
is copying the correct url for the `10m-populated-places.zip` and
`110m-admin-0-boundary-lines.zip` downloads. The full url is truncated
on screen so you'll have to right-click and "Copy Link Address". Also,
I learned from
[here](http://gevork.ru/2011/09/16/mapnikubuntu-and-10m-populated-places-zip/)
that it's necessary to rename the files in the 10m archive.

Here's where it gets a bit tricky:

The `generate_xml.py` and `generate_image.py` scripts are a bit
picky. I had to set a password for my postgres user to make the xml
output work.

    sudo -u postgres -i
    psql -d gis
    ALTER USER username WITH PASSWORD '123456';

Now this command should work:

    ./generate_xml.py --dbname gis --password 123456 --user reed --world_boundaries ./world_boundaries/ --port 5432 --host localhost

From
[here](http://trac.mapnik.org/wiki/OpenSolarisInstallation#Step8:TestingMapnik)
I figured out how to make the output of `generate_xml.py` compatible
with `Mapnik2`.

    upgrade_map_xml.py osm.xml osm2.xml
    nik2img.py osm2.xml image.png --mapnik-version 2

If everything worked well, at this point an image of the world should
pop up automatically. Now to go make some tile sets...
