## Purpose
Generate [GitHub Pages] for https://www.relg.uk.

## Fixing Mistakes or Adding Other Value
Feel free to open a pull request against `main`, if you found a mistake,
want to improve phrasing, or anything otherwise useful.
If you don't want to do it yourself, you can open an issue in this repository. 

## Interaction
Currently, I did not implement comments. If you want to ask questions
or interact otherwise with anything on the website, you can open an issue.

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

[GitHub Pages]: https://docs.github.com/en/pages
