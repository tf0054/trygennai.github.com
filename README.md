genn.ai Website
===============

About
------

This repository has Markdown syntax files compiled into http://pages.genn.ai/ website.

Prerequisites
--------------

* [Jekyll](http://jekyllrb.com/) (2.5 or greater)

How to Build
--------------

Clone this repository.

```
$ git clone git@github.com:TryGennai/trygennai.github.com.git
```

Add a needed gem.

```
$ sudo gem install jekyll
$ sudo gem install jekyll-redirect-from
```
 
Edit the source files.

```
$ emacs index.md
```

Build the modified source files into html files.

```
$ jekyll build
```

See the compiled pages.

```
$ jekyll server
```
