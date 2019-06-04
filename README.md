# imagebuilder

## `Dockerfile.image`

```
$ cd imagebuilder

$ docker build --force-rm --file Dockerfile.image -t imagebuilder .
```

## `/imagebuilder_run`

```
$ docker run --rm -ti -v /tmp:/tmp imagebuilder /imagebuilder_run
```

The generated files are in `/tmp/output`
