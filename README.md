# imagebuilder

## `Dockerfile.image`

```
$ cd imagebuilder

$ docker build --force-rm --file Dockerfile.image -t imagebuilder .
```

## `/imagebuilder_run`

```
$ docker run --rm --privileged=true -ti -v /tmp:/tmp -v /dev:/dev imagebuilder /imagebuilder_run
```

The generated files are in `/tmp/output`
