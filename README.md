# imagebuilder

## `Dockerfile.image`

```
$ cd imagebuilder

$ docker build --force-rm --squash --file Dockerfile.image -t imagebuilder .
```

## `/imagebuilder_run`

```
$ docker run --rm -ti -v /tmp:/tmp imagebuilder /imagebuilder_run
```

The generated files are in `/tmp/ppp3`
