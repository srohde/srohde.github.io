---
layout: post
title: "Location Based Services with MongoDB"
date: 2013-06-10 11:27
comments: true
categories: 
published: false
---

[MongoDB](http://www.mongodb.org/) is an open-source NoSQL database. It offers great features to build [Location Based Services](https://en.wikipedia.org/wiki/Location-based_service) with build in support for [Geospatial Indexes and Queries](http://docs.mongodb.org/manual/applications/geospatial-indexes/).

I've been using MongoDB with [Node.js](http://nodejs.org) on [Heroku](http://heroku.com) so here a quick walk through on how to set it up and use it:

## Setup

### Local

Install and Run:

    $ brew install mongo
    $ mongod

On lesson I've learned is to always try to use MongoDB from the shell first before you start with actual code. __After__ having the command right in the shell translate it into code and test your actual implementation. Here some shell commands to get you started:

    $ mongo
    > use mydbname

For this example we store locations in a collection called `locations`
Make sure you have set the geo spacial index:

    > db.locations.ensureIndex({loc: '2d'})
    > db.locations.getIndexKeys()

Should output `[ { "_id" : 1 }, { "loc" : "2d" } ]`

### Heroku

On the Heroku side it's pretty easy to set up as well:

    $ heroku addons:add mongohq:sandbox

To make sure the location index is also working on Heroku, login, select your app and select the MongoHQ add-on. This brings you to the MongoHQ admin interface. Then select your locations collection and make sure the `Indexes` are configured correctly. In my case it shows as `{ key: { loc: "2d" }, v: 1, ns: "appXXX.locations", name: "loc_2d" }`

## MongoDB Shell

### Insert Location

When you insert a new location it's essential to store the location in an array with [lon, lat] and __not__ [lat, lon]. This was a mistake I made first and it took me a while to figure that out because all my queries were returning the wrong stuff.

Let's insert some example locations (I've been using [LATLONG.net](http://www.latlong.net/)):

    > db.locations.insert({name: "Ferry Building", city: "San Francisco", loc:[-122.3937, 37.7955]})
    > db.locations.insert({name: "Union Square", city: "San Francisco", loc:[-122.407437, 37.787994]})
    > db.locations.insert({name: "Duboce Park", city: "San Francisco", loc:[-122.433453, 37.769422]}) 
    > db.locations.insert({name: "Parque Ibirapuera", city: "São Paulo",loc:[-46.657490, -23.586996]})
    > db.locations.insert({name: "Big Ben", city: "London",loc:[-0.124575, 51.500705]})
    > db.locations.insert({name: "Fischmarkt", city: "Hamburg",loc:[9.937740, 53.544527]})
    > db.locations.insert({name: "Opera House", city: "Sydney",loc:[151.214993, -33.857767]})


Now you can query all locations with:

    > db.locations.find()

Just as a sanity check: Everything west of Greenwich has a longitude < 0 and east > 0. Everything in the northern hemisphere has a latitude > 0 and in the southern hemisphere < 0 or as Wikipedia puts it:

{% blockquote http://en.wikipedia.org/wiki/Latitude %}
In geography, latitude (φ) is a geographic coordinate that specifies the north-south position of a point on the Earth's surface. Latitude is an angle (defined below) which ranges from 0° at the Equator to 90° (North or South) at the poles.
{% endblockquote %}

{% blockquote http://en.wikipedia.org/wiki/Longitude %}
Longitude (/ˈlɒndʒɨtjuːd/ or /ˈlɒŋɡɨtjuːd/),[1] is a geographic coordinate that specifies the east-west position of a point on the Earth's surface. [...] The longitude of other places is measured as an angle east or west from the Prime Meridian, ranging from 0° at the Prime Meridian to +180° eastward and −180° westward.
{% endblockquote %}

### Query Locations

After we have inserted some test locations let's try to make a geo-spatial query. In my case I wanted to find locations close to me so let's pretend we are at the Ferry Building in San Francisco.

    > db.runCommand({geoNear: "locations", near: [-122.3937, 37.7955], num: 10, spherical:true})

The result is ordered by distance and the Ferry Building should be on top with a `dis` of 0. To have the distance in a human readable unit you can use the [distanceMultiplier attribute](http://docs.mongodb.org/manual/tutorial/calculate-distances-using-spherical-geometry-with-2d-geospatial-indexes/#geospatial-indexes-distance-multiplier) and select for instance the earth radius in miles which is 3959:

    > db.runCommand({geoNear: "locations", near: [-122.3937, 37.7955], num: 10, spherical:true,
    distanceMultiplier: 3959})

If you want to restrict the search to a radius like 10 miles you'll have to use the maxDistance attribute:

    > db.runCommand({geoNear: "locations", near: [-122.3937, 37.7955], num: 10, spherical:true,
    distanceMultiplier: 3959, maxDistance:10/3959})

If you want kilometers use __6371__m as the radius of the earth.

Now that everything is working in the shell we can start with actual code.

## Node.js Code

This example is written with CoffeeScript and is better pseudo code. It has been extracted from an actual project and usually you would do more validation and parsing but you should get the idea on how the [Express](http://expressjs.com/) route is working:

``` coffeescript
mongo = require 'mongodb'
assert = require 'assert'

ObjectID = mongo.BSONPure.ObjectID

dbConnection = process.env.MONGOHQ_URL or 'mongodb://localhost/mydbname'

locations = null
db = null

mongo.Db.connect dbConnection, (err, database) ->
  db = database
  assert.equal null, err

  database.collection 'locations', (err, collection) ->
    assert.equal null, err
    locations = collection

exports.location = (req, res) ->
  switch req.method
    when 'POST'
      # In this example I am using upsert to update the record with the existing _id when present or create a new ObjectID if not:

      location = req.body
      location.loc = [parseFloat(location.loc[0]), parseFloat(location.loc[1])]

      isUpdate = location._id?
      id = if isUpdate then location._id else new ObjectID()
         
      locations.update {_id:id}, location, {upsert:true, safe:true}, (err, result) ->
        assert.equal null, err
        res.send JSON.stringify(location)
    when 'GET'
      lat = req.query["lat"]
      lon = req.query["lon"]

      if lat and lon
        lat = parseFloat lat
        lon = parseFloat lon

        # validate that lat/lon are numeric
        return res.send 400 if isNaN(lat) or isNaN(lon)

        # validate that lat/lon are in the right range
        return res.send 400 if lon < -180 or lat > 180

        geoNear = 
          geoNear: locations
          near: [lon, lat]
          spherical: true
          maxDistance: 10 / 3959
          distanceMultiplier: 3959

        db.command geoNear, (err, locations) ->
          assert.equal null, err
          res.send locations
```

