# blog.kabrt.cz
This repository contains source files for [Wyam generator](https://wyam.io) for my blog.

## Development
Use: 
```
wyam build c:\Users\Lukas\Source\Repos\blog.kabrt.cz -w -p
```
to run `Wyam` preview server, that watches for file changes in the input directory a servers output files on `http://localhost:5080` 

## Blog components cheatsheet
### Lightbox
```
<a href="/content/about/mountains.jpg" data-toggle="lightbox">
    <img src="/content/about/mountains.jpg" class="img-fluid" alt="Mountains are calling">
</a>  
```

### Figure
```
<figure class="figure">
    <a href="/content/about/mountains.jpg" data-toggle="lightbox">
        <img src="/content/about/mountains.jpg" class="figure-img img-fluid" alt="Mountains are calling">
    </a>  
    <figcaption class="figure-caption text-center">A caption for the above image.</figcaption>
</figure>
```
