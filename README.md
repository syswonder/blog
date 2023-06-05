# syswonder report

## deploy

Syswonder report uses [docsify](https://docsify.js.org) , just start a web
server at the docs directory is ok.

For example

```
git clone https://github.com/syswonder/report

cd report/docs

python -m http.server

```

then open a browser and visit http://localhost:8000

## write a report

1. place your report file, assume it be 202306_hello.md, at current year
   directory, e.g., 2023.

2. write 2023/README.md, add your report tile, abstract, etc.

3. add an item to 2023/_sidebar.md.

4. check it is displayed as expected.

## contribute

commit to your cloned repo and make a PR to https://github.com/syswonder/report
