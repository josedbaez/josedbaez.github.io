title: Tips when indexing Polygons with Sitecore and Solr
date: 2019/07/15
categories:
- Sitecore
- Solr
tags:
- Sitecore
- spatial
- Solr

---
<img class="hero-img" src="/images/sitecore-solr-spatial-tips.png" alt="Sitecore Solr GeoJson">
---
How to store and search on geojson polygons in sitecore.
<!-- more -->

*Versions used: Sitecore Experience Platform 9.0 rev. 180604 (Update-2).*

This is a follow-up post on my [previous one](/2019/07/sitecore-polygon-search/) regarding polygon search with Sitecore and solr. It's a list of things I learned, whilst implementing on a project, that will hopefully save you some debugging time.

- If you get error `Unsupported expression node type: Quote` when running the extension function, it means you are still using one or 2 default classes. Make sure you are referring all classes from the [repo](https://github.com/josedbaez/sitecore-spatial-geojson-polygons).
- If you get an error at indexing time, make sure the data being stored is a valid [GeoJSON](https://geojson.org/) object. 
-- e.g. this is a valid polygon ```{ "type": "Feature", "geometry": { "type": "MultiPolygon", "coordinates": [ [ [ [ 20.18, 46.2 ], [ 20.2, 46.26 ], [ 20.32, 46.26 ], [ 20.34, 46.22 ], [ 20.38, 46.22 ], [ 20.38, 46.160000000000004 ], [ 20.22, 46.160000000000004 ], [ 20.18, 46.2 ] ] ] ] } }```
- Ross Cartwright pointed out `queryable.ToList()` works fine, but `queryable.GetResults()` fails. We actually avoided using `GetResults()` throughout the project because it was slower than `ToList()`. You'll need to debug to find out why it fails.
- Make sure you [read](https://lucene.apache.org/solr/guide/6_6/spatial-search.html) about the differences between `RPT` and `RptWithGeometrySpatialField`
-- If you end up using `RptWithGeometrySpatialField`, see `optional in-memory cache`
- There's a limit on the amount of data being sent -at once- to solr, and this can also decrease performance, so be mindful on the polygon size being used. On my project, we were indexing cities and a polygon could easily reach 5MB. We used [openstreemap](https://nominatim.openstreetmap.org/search.php) to generate [non-perfect versions](http://polygons.openstreetmap.fr/) of the polygons which are quite smaller.
- `RPT` and `RptWithGeometrySpatialField` can index bounding boxes as long as you store them as polygons. 
-- e.g. ```{ "type": "Feature", "geometry": { "type": "Polygon", "coordinates": [ [ [ 20.2024989, 46.2431191 ], [ 20.356019, 46.2431191 ], [ 20.356019, 46.1757964 ], [ 20.2024989, 46.1757964 ], [ 20.2024989, 46.2431191 ] ] ] } }``` would work but `[ "46.1757964", "46.2431191", "20.2024989", "20.356019" ]` won't.
- If storing multi-polygons that overlap, set `allowMultiOverlap` to true when defining your `dynamicField`.

---

Please let me know what you think and/or if you can spot any errors.
*/eom*
