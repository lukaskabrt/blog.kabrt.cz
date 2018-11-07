Title: Clipping OpenStreetMap data with SpatialLITE
Published: 03/26/2012
Tags: [OpenStreetMap, SpatialLITE]
---
In the [previous post](/posts/2012-03-filtering-osm-data-with-spatiallite) I have shown, how to do simple filtering of the OSM data with the [SpatialLITE library](https://github.com/lukaskabrt/SpatialLITE). Today I will go a step further and do something more complicated - extract data from OSM file within specific bounding box.

It seems to be pretty straightforward task so, how hard can it be?

There is one problem, we need to solve - some ways and relations near edges of the bounding box would spread across bounding box boundaries - part of the entity would lie inside the bounding box and the other part outside. We need to decide what to do with these split entities. Should we include them? Should we exclude them?

### Split entities
Basically there are four way how to deal with them:

1) exclude ways and relations that have any part outside the bounding box - this leaves isolated nodes in the output - see picture
<a href="/content/2012-03-clipping-osm-data-with-spatiallite/20120323-osmosis-cropped.png" data-toggle="lightbox">
    <img src="/content/2012-03-clipping-osm-data-with-spatiallite/20120323-osmosis-cropped.png" class="figure-img img-fluid" alt="Mountains are calling">
</a>  


2) include ways and relations that lies partially outside the bounding box, but do not add any nodes that lies outside the bounding box - this leaves ways and relations incomplete - again see picture
<a href="/content/2012-03-clipping-osm-data-with-spatiallite/20120323-osmosis-cropped-clip.png" data-toggle="lightbox">
    <img src="/content/2012-03-clipping-osm-data-with-spatiallite/20120323-osmosis-cropped-clip.png" class="figure-img img-fluid" alt="Mountains are calling">
</a>  


3) include all entities with all nodes that lies partially inside the bounding box - this produces most the most complete map, but on the other hand you will end up with many entities outside the bounding box
<a href="/content/2012-03-clipping-osm-data-with-spatiallite/20120323-osmosis-cropped-ways.png" data-toggle="lightbox">
    <img src="/content/2012-03-clipping-osm-data-with-spatiallite/20120323-osmosis-cropped-ways.png" class="figure-img img-fluid" alt="Mountains are calling">
</a>  


4) clip entities at the edges of bounding box

The option 4 might seem as the best solution, but we would need to add new nodes on the edges of the bounding box, split ways into two, or do other complicated (with OSM data model) things.

Options 1 - 3 look very similar to each other and it depends on your needs, which one you should choose. But from the performance point of view option 3 is significantly more complex. Entities in OSM files are sorted by entity type and then by ID, so with options 1 and 2 we can create output file in single pass, but with option 3 it is impossible. During first pass we find all nodes in the bounding box, and all ways and relations that contain any of the selected nodes. In the second pass we need to add nodes that are outside bounding box but belong to the included ways and relations.

### Tracking IDs
For all options we need to track IDs of entities inside bounding box (or you can keep all entities in the memory - but with millions of entities it might be impossible). There are couple ways of doing that - for example Osmosis is using either bit board or list of integers. I'd like to try something else - as mentioned above OSM files are sorted by entity type and then by ID, another observation tells us that entities with consecutive ID often lies near each other. We can take advantage of that - we will store IDs as list of `Range` objects.

```csharp
struct IdRange {
  public int From;
  public int To;
}
```

Every `IdRange` structure will represent consecutive range of IDs. If there are at least 2 IDs in `IdRange` object in average, this representation of used nodes will use less memory then simple list of IDs. 

The code for tracking IDs isn't complicated ...

```csharp
public class IdTracker {
    private List<IdRange> _storage = new List<IdRange>();
    private IdRange _range = IdRange.Empty;

    public void Add(int id) {
        idCount++;

        if (_range.Equals(IdRange.Empty)) {
            _range.From = id;
            _range.To = id;
        } else {
            if (_range.To == id - 1) {
                _range.To++;
            } else {
                _storage.Add(_range);
                _range = new IdRange() { From = id, To = id };
            }
        }
    }
}
```

As mentioned above, entities are sorted by type and ID, so when checking whether specific ID has been added into the `IdTracker`, we can use binary search to speed things up. 

### Results
With tricky parts solved, writing whole application for clipping isn't too interesting, so I won't describe it in details, but the final code is available for [download](/content/2012-03-clipping-osm-data-with-spatiallite/20120326_osm-extracting.zip).

Finally another speed comparison - again SpatialLITE vs. Osmosis. The test file was  245MB PBF file with approx. 28 millions entities. The output file has approx. 300 000 entities.

^^^
|       |SpatialLITE (1)    |SpatialLITE (3)    |Osmosis (1)    |Osmosis (3)    |
|-------|------------------:|------------------:|--------------:|--------------:|  
|Run 1  |1:24               |3:29               |1:57           |8:51           |
|Run 2  |1:25               |3:27               |1:55           |8:50           |
|Run 3  |1:24               |3:27               |1:58           |8:43           |
^^^ Speed comparison - all times are in minutes{.text-center}

Again not bad :-)

When it comes to the memory consumption the results are worse for SpatialLITE. .NET applications are usually a little bit memory intensive and this sample application isn't exception. While peak memory consumption of Osmosis was 53MB, SpatialLITE took whole 84MB of the memory. It seems that the biggest memory consumption was caused by PbfWriter, which is weird and I will have take a look on it. The IdTracker, that should occupy the most of the memory, actually reached size of only 19MB. An internal  stats from the IdTracker shows, that there are approx. 4.5 IDs in every Range object - so it isn't actually bad idea.