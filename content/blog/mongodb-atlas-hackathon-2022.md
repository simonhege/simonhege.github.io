---
title: Varmomapo - MongoDB Atlas Hackathon 2022 on DEV
date: 2022-12-07T08:40:51+02:00
draft: false
description: A serverless web application, using MongoDB Atlas to display heat maps based on OpenStreetMap data. My submission to the MongoDB Atlas Hackathon 2022 on DEV!
tags: ['atlashackathon22', 'mongodb', 'googlecloud', 'go']
---

## What I built
*Varmomapo* is a serverless web application, using MongoDB Atlas to display heat maps based on OpenStreetMap data.

### Category Submission:
- Think Outside the JS Box
- Google Cloud Superstar

### App Link
https://varmomapo.xdbsoft.com

### Screenshots

#### Bar and other places to have a drink, near Cannes, France
<p>
  <img src="https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ye7s5n3br9dit38n1h1p.png" class="img-fluid" style="width: 36rem" alt="Heatmap of bars near Cannes">
</p>

#### Restaurants near me, on my phone
<p>
  <img src="https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o2np07t4limbo3qpihy4.png" class="img-fluid" style="width: 18rem" alt="Heatmap of restaurants near Peymeinade, France">
</p>

#### Wind turbines in northern west Europe
<p>
  <img src="https://dev-to-uploads.s3.amazonaws.com/uploads/articles/umsbix0ep8htaxu5vc6j.png" class="img-fluid" style="width: 36rem" alt="Heatmap of wind turbines in northern Europe">
</p>


### Description
*Varmomapo* is composed of 4 main parts: 
- a web application, usable from desktop or mobile to display heat maps based on OpenStreetMap data. Multiple layers are available: restaurants, bars, playgrounds, ... Geolocalisation is possible to find local information. This is build with plain HTML and JavaScript, using [Leaflet](https://leafletjs.com/) and [Bootstrap](https://getbootstrap.com) frameworks.
- a backend, computing and serving tiled heat maps. This is a Go application build via [Google Cloud Build](https://cloud.google.com/build) and executed via [Google Cloud Run](https://cloud.google.com/run). It connects to MongoDB Atlas, hosted in same GCP region, to retrieve the features to be displayed.
- a data store, hosted in [MongoDB Atlas](https://www.mongodb.com/atlas). Two collections are used: one for all the OpenStreetMap items (stored as GeoJSON features) and one to cache the most viewed tiles. It uses [geospatial indexes](https://www.mongodb.com/docs/manual/core/2dsphere/) to ensure great performance.
- a tool, written in Go, to import the data from the OpenStreetMap protobuf file into the MongoDB data store.

### Link to Source Code
https://github.com/xdbsoft/varmomapo

### Permissive License
[MIT](https://choosealicense.com/licenses/mit/)

## Background
I'm the happy father of 2 little girls. They really, really, like to go to playgrounds. I know the few ones in our own area, but as soon as we travel a bit, I have no clue where to find them. 

Heatmaps are great to visualize data and have a global view of precise data. This is where *Varmomapo* originated.

And then, once you have the data, the code to generate such maps and the infrastructure that scales, it becomes easy to display nice things, such as tourism information (restaurants, hotels, art work), or energy infrastructure (wind turbines, nuclear reactors). The possibilities are unlimited.


### How I built it

#### Importing the data
The first part that I built was the tool to import the data. I selected Go and used the official MongoDB driver to connect to the Atlas data store.
The first dataset I imported was the Monaco extract. As it is a really small one (less than 10 000 objects), it was easy to iterate, dropping and recreating the database each time. Then I moved to a bigger data set, for one of the french region. I improved the tool by using bulk import. My local internet bandwidth allowed me to import the 250 000 features in around 20 minutes.
But this would not scale to France or full planet. Looking at cluster metrics in MongoDB Atlas, the issue was not on that side, thus definitely on my network. 

I decided to spawn a VM in Google Cloud Platform and launched the data import from there. As the tool is written in Go cross compiling from my local Windows laptop to a Linux OS is only a matter a single environment variable (`set GOOS=linux`). The binary was uploaded to the cloud hosted VM and performance drastically improved. At that time I switched from a free cluster to a larger one, in order to be able to import the full planet (around 40 millions features imported in few hours).

Using a managed service such as MongoDB Atlas helped to build fast. The free cluster allows to experiment and later switch to a more adequate for production workload. Even more, the auto scaling feature popped up at the right time to enable the import of the full planet file, and scaled down few hours later when import was done. Also the Atlas interface helps to explore data, and can even provide a preview of the heatmap for a given query thanks to [MongoDB Atlas Charts](https://www.mongodb.com/products/charts).

<p>
  <img src="https://dev-to-uploads.s3.amazonaws.com/uploads/articles/111c52spe193uyt8h7i6.png" class="img-fluid" alt="Example usage of MongoDB Atlas Chart">
</p>

#### Querying and serving the heat map tiles

The second aspect was to implement the backend. In order to ease integration in the UI, I decided to implement a [Tile Map Service](https://en.wikipedia.org/wiki/Tile_Map_Service), which is basically a server that given a zoom level and coordinates of a tile must provide a png for the area. 

I was already aware of some packages that allows to read from OpenStreetMap protobuf format and geospatial aspects. I took the official MongoDB client for Go to interract with the data store. I also found a package that generates heat maps, forked it and glued all things together. In order to be able to write proper Go code that uses the MongoDB driver. I first experimented the queries in the MongoDB Atlas web interface and exported the code from here. Once I understood the logic to build the filter with `bson.D` data type (thanks to the example) I was able to write them on my own.

<p>
  <img src="https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s6oqbqn7vrphml1lij1d.png" class="img-fluid" alt="Example of generated code from the web interface">
</p>


With the small "Monaco" dataset, performance were good, but I had to add [2dsphere compound indexes](https://www.mongodb.com/docs/manual/core/2dsphere/#create-a-compound-index-with-2dsphere-index-key) for the larger datasets.

Thanks to Google Cloud Build and Google Cloud Run, I was able to run the exact same code in my local environment and deployed in production. Furthermore, thanks to the build-pack standardization, the repository does not contain anything related to the deployment. A few click in the Google cloud console allows to define the CI/CD pipeline and have the executable serving request from the cloud. Custom domain setup with SSL is another few more clicks.
<p>
  <img src="https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wis1m7vqhjy8kw1914nb.png" class="img-fluid" alt="Google Cloud Build interface">
</p>


### Additional Resources/Info

- [OpenStreetMap](https://www.openstreetmap.org/copyright), as the best source of reusable geospatial data
- [MongoDB Go driver guide](https://www.mongodb.com/docs/drivers/go/current/)
- [Quickstart to deploy a Go service to Cloud Run ](https://cloud.google.com/run/docs/quickstarts/build-and-deploy/deploy-go-service)
- [github.com/paulmach/osm](https://github.com/paulmach/osm): a Go package to read and transform OpenStreetMap PBF format.
- [github.com/dustin/go-heatmap](https://github.com/dustin/go-heatmap): a Go package to generate heatmaps

__This post was originally posted on https://dev.to/xeonx/varmomapo-mongodb-atlas-hackathon-2022-on-dev-2ddp__