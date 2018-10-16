# blog.kabrt.cz
This repository contains source files for [Wyam generator](https://wyam.io) for my blog.

## Development
Use: 
```
wyam build c:\Users\Lukas\Source\Repos\blog.kabrt.cz -w -p
```
to run `Wyam` preview server, that watches for file changes in the input directory a servers output files on `http://localhost:5080` 

## Deployment
Use: 
```
wyam build c:\Users\Lukas\Source\Repos\blog.kabrt.cz
```
to generate output files.