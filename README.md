# syswonder tech blog

## deploy

Syswonder blog uses [docsify](https://docsify.js.org) , just start a web
server at the docs directory is ok.

For example

```
git clone https://github.com/syswonder/blog

cd blog/docs

python -m http.server

```

then open a browser and visit http://localhost:8000

## write a blog

1. place your blog file, assume it be 202306_hello.md, at current year
   directory, e.g., 2023.

2. modify 2023/README.md, add your report title, data, writer name, abstract, etc.

3. add an item of this blog to 2023/_sidebar.md.

4. check it is displayed as expected.

## contribute

commit to your cloned repo and make a PR to https://github.com/syswonder/blog
