title: Polygon search with Sitecore and Solr
thumbnail: /images/london-polygon.png
date: 2019/07/03
categories:
- Sitecore
- Solr
tags:
- Sitecore
- spatial
- Solr

---
<img class="hero-img" src="/images/london-polygon-world.png" alt="Sitecore Solr GeoJson">
---
How to store and search on geojson polygons in sitecore.
<!-- more -->

*Versions used: Sitecore Experience Platform 9.0 rev. 180604 (Update-2).*

**Update:**
Make sure you read my [follow-up post](/2019/07/sitecore-polygon-search-part2) with some tips about this implementation.

On this post, I will be going through implementation on how to index fields that contain GeoJson polygons and a Linq extension that allows searching for points that intersect the polygons. Full source code [here](https://github.com/josedbaez/sitecore-spatial-geojson-polygons).

In order to follow the same approach taken by Sitecore, we need to create a new index provider. The provider uses a Search Context that exposes a GetQueryable function with the ability to trigger our custom intersect method. 

You'll need to define the spatial field type in Solr. See my [previous post](/2019/06/solr-dynamic-spatial-fields) on how to do this.

The [implementation](https://github.com/josedbaez/sitecore-spatial-geojson-polygons) looks long but it's mostly inheriting from base Sitecore classes. It is basically an extension from [ehabelgindy's WithinRadius implementation](https://github.com/ehabelgindy/sitecore-solr-spatial) which is now part of Sitecore code base. 

I decided to use a custom index and a computed field. You could create a new Sitecore field type and map it to a solr spatial field type instead, but I had issues with this when using custom indexes.

My custom index configuration looks like:
``` xml
<contentSearch>
    <indexConfigurations>
        <defaultSolrIndexConfiguration type="Sitecore.ContentSearch.SolrProvider.SolrIndexConfiguration, Sitecore.ContentSearch.SolrProvider">
        <fieldMap type="Sitecore.ContentSearch.SolrProvider.SolrFieldMap, Sitecore.ContentSearch.SolrProvider">
            <typeMatches hint="raw:AddTypeMatch">
                <typeMatch typeName="geojsonrptgeom" type="System.String" fieldNameFormat="{0}_grptgeom" settingType="Sitecore.ContentSearch.SolrProvider.SolrSearchFieldConfiguration, Sitecore.ContentSearch.SolrProvider" patch:before="*[1]"/>
            </typeMatches>
        </fieldMap>
        </defaultSolrIndexConfiguration>
    </indexConfigurations>

    <configuration type="Sitecore.ContentSearch.ContentSearchConfiguration, Sitecore.ContentSearch">
    <indexes hint="list:AddIndex">
        <index id="sc9xp0_spatial_master" type="Sitecore.Spatial.GeoJson.Providers.Solr.SolrSearchIndex, Sitecore.Spatial.GeoJson" role:require="Standalone OR ContentManagement">
        ...
        ...
        <configuration ref="contentSearch/indexConfigurations/defaultSolrIndexConfiguration" >
            <documentOptions ref="contentSearch/indexConfigurations/defaultSolrIndexConfiguration/documentOptions">
            <fields hint="raw:AddComputedIndexField">
                <field fieldName="polygon" returnType="geojsonrptgeom">Sitecore.Spatial.GeoJson.ComputedFields.Polygon, Sitecore.Spatial.GeoJson</field>
            </fields>
            ...
            ...
            </documentOptions>
        </configuration>
       ...
       ...
        </index>
    </indexes>
    </configuration>
</contentSearch>
```
Here, I'm adding a typeMatch `geojsonrptgeom` and mapping it to solr dynamic type `grptgeom`. I'm then adding a computed field that returns this typeMatch. As you can [see](https://github.com/josedbaez/sitecore-spatial-geojson-polygons/blob/master/src/ComputedFields/Polygon.cs), the computed field is just returning the value entered on the field.

Note the custom provider `Sitecore.Spatial.GeoJson.Providers.Solr.SolrSearchIndex, Sitecore.Spatial.GeoJson` set on index type property.

As stated before, there are couple of custom classes that are basically inheriting from base and initialising instances of the custom ones: [SolrSearchIndex.cs](https://github.com/josedbaez/sitecore-spatial-geojson-polygons/blob/master/src/Providers/Solr/SolrSearchIndex.cs), [SolrSearchContext.cs](https://github.com/josedbaez/sitecore-spatial-geojson-polygons/blob/master/src/Providers/Solr/SolrSearchContext.cs), [LinqToSolrIndex.cs](https://github.com/josedbaez/sitecore-spatial-geojson-polygons/blob/master/src/Providers/Solr/LinqToSolrIndex.cs), [ExpressionParser.cs](https://github.com/josedbaez/sitecore-spatial-geojson-polygons/blob/master/src/Linq/ExpressionParser.cs).

[LinqToSolrIndex.cs](https://github.com/josedbaez/sitecore-spatial-geojson-polygons/blob/master/src/Providers/Solr/LinqToSolrIndex.cs) initialises a new QueryMapper [SolrQueryMapper.cs](https://github.com/josedbaez/sitecore-spatial-geojson-polygons/blob/master/src/Providers/Solr/SolrQueryMapper.cs) that contains the new spatial function that will generate the Solr `Intersects` query. 
``` csharp
protected virtual AbstractSolrQuery VisitInsidePolygon(InsidePolygonNode node, SolrQueryMapperState state)
{
    AbstractSolrQuery abstractSolrQuery = new SolrQueryByField(node.Field, $"Intersects({node.Coordinate.Latitude} {node.Coordinate.Longitude})");
    if (!abstractSolrQuery)
    {
        return abstractSolrQuery & this.Visit(node.SourceNode, state);
    }

    return abstractSolrQuery;
}
```

Now that we have the SolrQuery method, we just need to create a Linq Extension so we can easily call this method when querying the index.
``` csharp
public static class QueryableExtensions
{
    public static IQueryable<TSource> InsidePolygon<TSource, TKey>(this IQueryable<TSource> source, Expression<Func<TSource, TKey>> keySelector, Coordinate center)
    {
        var method = ((MethodInfo)MethodBase.GetCurrentMethod()).MakeGenericMethod(typeof(TSource), typeof(TKey));
        var arguments = new Expression[3]
                                        {
                                            source.Expression,
                                            Expression.Quote(keySelector),
                                            Expression.Constant(center, typeof(Coordinate))
                                        };
        return source.Provider.CreateQuery<TSource>((Expression)Expression.Call(null, method, arguments));
    }
}
```

This will allow us to build queries that use this extension like:
`var queryable = context.GetQueryable<PolygonResultItem>().InsidePolygon(s => s.Polygon, new Coordinate(lat, lng));`

__Note:__ `Intersects` can be done via Queries or Filter Queries. This implementation uses Queries. For more information on filter queries, see [solr documentation](https://github.com/SolrNet/SolrNet/blob/master/Documentation/Querying.md#filter-queries).

The demo on the [source code](https://github.com/josedbaez/sitecore-spatial-geojson-polygons) provides a template with the spatial field, couple of sample items with polygons from [openstreetmap](https://nominatim.openstreetmap.org/search.php) that cover some cities; and a controller rendering that reads `lat` and `lng` from the querystring and returns the polygon of the item found (e.g. Big Ben's coordinates http://sc9xp0.sc/?lat=-0.1268141&lng=51.5007292 will return London).

<img class="hero-img" src="/images/london-polygon.png" alt="Greater London Polygon">

====================
References:
https://github.com/ehabelgindy/sitecore-solr-spatial
https://developers.google.com/maps/documentation/javascript/datalayer
https://developers.google.com/maps/documentation/javascript/importing_data
https://stackoverflow.com/a/42127338/991459
https://lucene.apache.org/solr/guide/6_6/spatial-search.html
https://github.com/ehabelgindy/sitecore-solr-spatial
http://polygons.openstreetmap.fr/index.py
https://nominatim.openstreetmap.org/search.php

---

Please let me know what you think and/or if you can spot any errors.
*/eom*
