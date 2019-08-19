title: Add dynamic geojson fields to Solr
thumbnail: /images/solr-spatial.jpg
date: 2019/06/30
categories:
- Solr
tags:
- spatial

---
<img class="hero-img" src="/images/solr-spatial.jpg" alt="Solr Spatial">
---
How to add dynamic geojson fields to Solr 6 to index and search geospatial data.
<!-- more -->

*Versions used: Solr 6.6.3.*

Solr supports indexing spatial fields on industry standards: [WKT](https://en.wikipedia.org/wiki/Well-known_text_representation_of_geometry) and [GeoJSON](https://geojson.org/). In this post, I'll be going through indexing spatial fields on GeoJSON format, especially to index polygons.

First, you need to choose between [SpatialRecursivePrefixTreeFieldType (RPT)](https://lucene.apache.org/solr/guide/6_6/spatial-search.html#SpatialSearch-RPT) or [RptWithGeometrySpatialField (RPT with Geometry)](https://lucene.apache.org/solr/guide/6_6/spatial-search.html#SpatialSearch-RptWithGeometrySpatialField). Both support indexing polygons but there are a couple of technical differences that can suit your needs better. I chose RPT with Geometry.

For polygon support, you will need [JTS Topology Suite](https://sourceforge.net/projects/jts-topo-suite/). Navigate to this [URL](https://repo1.maven.org/maven2/com/vividsolutions/jts-core/), click on required version and download file __jts-core-VERSION#.jar__. This file should be copied to: __Solr Installation Folder\server\solr-webapp\webapp\WEB-INF\lib\__

Add the new field type to your Solr index schema. 
``` xml
<!-- RPT -->
<fieldType name="geojson_rpt" class="solr.SpatialRecursivePrefixTreeFieldType" spatialContextFactory="JTS" geo="true" format="GeoJSON" autoIndex="true" validationRule="repairBuffer0" distErrPct="0.025" maxDistErr="0.001" distanceUnits="kilometers" />
<!-- RPT with geometry -->
<fieldType name="geojson_rptgeom" class="solr.RptWithGeometrySpatialField" spatialContextFactory="JTS" geo="true" format="GeoJSON" autoIndex="true" validationRule="repairBuffer0" distErrPct="0.15" maxDistErr="0.001" distanceUnits="kilometers"/>
```

Then, add a dynamic field that maps to the newly defined field type.
``` xml
<dynamicField name="*_grpt" type="geojson_rpt" indexed="true" stored="true" allowMultiOverlap="true"/>
<dynamicField name="*_grptgeom" type="geojson_rptgeom" indexed="true" stored="true" allowMultiOverlap="true"/>
```
Once you have done this, you can store GeoJson data on these fields and run Spatial searches against them. The best way to test if it works is by storing a value. If it's invalid, Solr will throw and error than can be seen in the lgos.

e.g. The following filter query will return documents that intersect the given coordinate.
```
polygon_grpt:"Intersects(19.0497665 47.4916717)"
```

__Notes:__
- distErrPct and maxDistErr can be tweaked for less accuracy and faster performance.
- Solr recommends to not store these field types as it would be redundant and will increase index size.
> When using this field type, you will likely not want to mark the field as stored because it’s redundant with the DocValues data and surely larger because of the formatting (be it WKT or GeoJSON). [Source](https://lucene.apache.org/solr/guide/6_6/spatial-search.html#SpatialSearch-RptWithGeometrySpatialField)
- On the same link, Solr mentions an in-memory cache that can be enabled to improve performance.
> An optional in-memory cache can be defined in solrconfig.xml, which should be done when the data tends to have shapes with many vertices. 


---

Please let me know what you think and/or if you can spot any errors.
*/eom*
