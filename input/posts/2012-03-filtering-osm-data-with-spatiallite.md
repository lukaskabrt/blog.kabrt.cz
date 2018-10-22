Title: Filtering OSM data with SpatialLITE
Published: 11/03/2012
Tags: [OpenStreetMap, SpatialLITE]
---

I'm a big fan of the [OpenStreetMap project](https://www.openstreetmap.org/), so when I released the first version of the [SpatialLITE library](https://github.com/lukaskabrt/SpatialLITE) last week, classes for working with OpenSteetMap data couldn't be missing in the project. Right now the library supports OSM XML files (without compression) and OSM PBF files, both for reading as well as writing. And how can you work with OSM data? How does SpatialLITE library compare with other tools in terms of speed? Let's find out

...  err wait, some introduction might be neccessary ... 

### IEntityInfo vs. IOsmGeometry

There are two different representation of the OSM entities in the library - `IEntityInfo` and `IOsmGeometry` objects. 

`IEntityInfo` represents a lightweight object that contains only data for the particular entity - relationships with other entities are described only by IDs of related entities.

<figure class="figure text-center w-100">
    <img src="/content/2012-03-filtering-osm-data-with-spatiallite/20120311-entityinfo.png" class="figure-img img-fluid" alt="WayInfo class">
    <figcaption class="figure-caption text-center">WayInfo class</figcaption>
</figure>

On the other hand a collection of `IOsmGeometry` objects represents tree of interconnected objects that implement `IGeometry` interface. This allows you to access properties of related entities directly, perform spatial analysis with methods from SpatialLite.Core library or use any other methods from the library that accepts `IGeometry` parameters. `IOsmGeometry` objects are pretty powerful, but the price is obvious - all objects must be held in the memory, and with large files it might easily become OutOfMemoryException kind of problem.

<figure class="figure text-center w-100">
    <img src="/content/2012-03-filtering-osm-data-with-spatiallite/20120311-osmgeometry.png" class="figure-img img-fluid" alt="Way class">
    <figcaption class="figure-caption text-center">Way class</figcaption>
</figure>

^^^
```csharp
Tag signalsTag = new Tag("highway", "traffic-signals");
IEnumerable waysWithSignals = ways.Where(w => w.Nodes.Any(n => n.Tags.Contains(signalsTag)));
```
^^^ Example of IOsmGeometry usage - find all ways that contain traffic signals{.text-center}

The decision whether use `IEntityInfo` objects or `IOsmGeometry` objects is up to you. Is some cases it might be better to stick with the simple `IEntityInf`o objects and sometimes you will need more complex `IOsmGeometry` objects. Fortunately it is possible to switch between these two representations of the OSM entities.


### Introducing OSM Readers / Writers

SpatialLITE library defines two interfaces related to IO tasks - `IOsmReader` and `IOsmWriter`. These interface are fairly simple:

<figure class="figure text-center w-100">
    <img src="/content/2012-03-filtering-osm-data-with-spatiallite/20120311-readerwriter.png" class="figure-img img-fluid" alt="IOsmWriter and IOsmReader interfaces">
    <figcaption class="figure-caption text-center">IOsmWriter and IOsmReader interfaces</figcaption>
</figure>

Right now there are two formats supported - OSM XML and OSM PBF. Both formats support reading as well as writing - so there is `OsmXmlReader`, `OsmXmlWriter`, `PbfReader` and `PbfWriter`.

Readers accepts a stream or a file in their constructors and provide forward only reading capabilities - pretty much the same behaviour you find in build-in readers (e.g. `BinaryReader`). Because of the structure of files it is impossible to create `IOsmGeometry` objects in the reader and thus readers returns `IEntityInfo` objects. If you need to work with `IOsmGeometry` objects, you can use `OsmDatabase` class that encapsulates process of creating full fledged objects from data read by `IOsmReader`.

The principle of the writers is pretty much the same - they also accepts a stream or a file in the constructor and then provide forward only, writing capabilities. Both `IOsmGeometry` and `IEntityInfo` objects contains all necessary information for serialization, so both object types are accepted as parameter of the `Write` method.

### Putting it together

OK, now when we have covered basics, let's go back to the title of this post - filtering of OSM data. 

For the purpose of this demo we would like to find all guide posts in an OSM file. In OSM a guide post is represented by a node with the `information:guidepost` tag. Writing an expression to choose such entities is simple:

```csharp
Tag required = new Tag("information", "guidepost");
if (info.EntityType == EntityType.Node && info.Tags.Contains(required)) {
 ...
}
```

When put together with the reader and writer we got the code of a simple utility for OSM data filtering:

```csharp
string inputFile = "input.osm";
string outputFile = "output.osm";

var readerSettings = new OsmXmlReaderSettings() { StrictMode = false };
var writerSettings = new OsmWriterSettings() { ProgramName = "OsmFilter_Demo" };

using (var reader = new OsmXmlReader(inputFile, readerSettings)) {
    using (var writer = new OsmXmlWriter(outputFile, writerSettings)) {
        var required = new Tag("information", "guidepost");
        IEntityInfo info = null;
    
        while ((info = reader.Read()) != null) {
            if (info.EntityType == EntityType.Node && info.Tags.Contains(required)) {
                writer.Write(info);
            }
        }
    }
}
```

### Comparison with Osmosis

I wanted to find out how fast the SpatialLITE library is - [Osmosis](https://wiki.openstreetmap.org/wiki/Osmosis) application was chosen as competitor, because it is probably the most popular tool for processing OSM data. Both tools were used to perform the same task: Extract all nodes with `information:guidepost` tag from the OSM files (5.25 GB OSM XML file, 245 MB PBF file). 

Let's see the results ...

^^^
|       |SpatialLITE (XML)  |SpatialLITE (PBF)  |Osmosis (XML)  |Osmosis (PBF)  |
|-------|------------------:|------------------:|--------------:|--------------:|
|Run 1  |5:01               |1:30               |5:58           |1:46           |
|Run 2  |4:48               |1:34               |6:40           |1:44           |
|Run 3  |4:56               |1:29               |6:01           |1:43           |
^^^ Speed comparison of SpatialLITE and Osmosis (all times in minutes){.text-center}

Not bad, I guess :-)

OK, it might not be fair comparison for the Osmosis, it has many additional features and can perform significantly more complex filtering, but it shows that the reader / writer classes in the SpatialLITE library are anything but slow.


For the sake of completeness - Osmosis parameters:
```
osmosis 
    --read-xml file="input.osm"
    --node-key-value keyValueList="information.guidepost"
    --write-xml file="output.osm"
```

```
osmosis 
    --read-pbf file="input.pbf"
    --node-key-value keyValueList="information.guidepost"
    --write-pbf file="output.pbf"
```