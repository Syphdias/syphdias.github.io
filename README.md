## Purpose
Generate [GitHub Pages].

## Fixing mistakes
Feel free to open a pull request against `main`.

## Cloning
Make sure to get the submodule contents or otherwise [running the
webserver](#testing) will fail.
```
git clone --recursive git@github.com:Syphdias/syphdias.github.io.git
```
or
```
git clone git@github.com:Syphdias/syphdias.github.io.git
cd syphdias.github.io
git submodule update --init
```

## Testing
Use this to run static site generation locally and host a local webserver.
```
hugo server
```
