# jupyter-ml-tensorflow and jupyter-ml-pytorch

This repository contains the source code for two o²S²PARC Services: jupyyter-ml-tensorflow and jupyter-ml-pytorch

Building the docker images:

```shell
make build
```


Test the built images locally:


**pytorch**
```shell
make run-pytorch-local
```

**tensorflow**
```shell
make run-tensorflow-local
```

Note that the `validation` directory will be mounted inside the service.



Raising the version can be achieved via one for three methods. The `major`,`minor` or `patch` can be bumped, for example:


**pytorch**
```shell
make version-pytorch-patch
```

**tensorflow**
```shell
make version-tensorflow-patch
```


If you already have a local copy of **o<sup>2</sup>S<sup>2</sup>PARC** running and wish to push data to the local registry:

```shell
make publish-local
```

