+++
title = "Static websites with Zola"
date = 2020-07-24
+++

# [Zola](https://www.getzola.org) a static site generator

This post is about Zola, a static website generator and will show you how to make your first Zola post.
  
Zola is a static site engine written in Rust. The big advantage of static websites is, that they are fast and take advantage of caching. That's because the webserver only has to serve static rarely changing files and not make dynamic lookups against a database or execute scripts.
OpenBSD's `httpd` is a good choice for a server serving static files.


With Zola all commands are compiled into one binary, so there are no dependencies except Rust.
I like it because it is very easy to use and very fast, especially for small sites. The CLI is intuitive and building your site takes just some milliseconds. In essence Zola just stays out of my way, that's how i like it . That way I can focus on content rather than being distracted by an unintuitive CLI or reading docs.

Featurewise it is almost the same as Hugo, for a comparison see this [link](https://github.com/getzola/zola). Zola mainly uses a different [template engine](https://tera.netlify.app/) and a different file format for configurations files, namely [TOML](https://github.com/toml-lang/toml) instead of [YAML](https://yaml.org/), quick comparison [here](https://gist.github.com/oconnor663/9aeb4ed56394cb013a20).
Also Zola supports [Sass](https://sass-lang.com/) which makes CSS suck a little less. Zola's commands are pretty self-explanatory:

<br></br>
## Zola commands

Initialize a directory for Zola:

```sh
zola init
```

Build the site:

```sh
zola build
```

Check for errors: 

```sh
zola check
```

Build and serve the site locally, useful for debugging (by default at 127.0.0.1:1111):

```sh
zola serve
```
<br></br>
## Building a site with Zola

First install Zola:

```sh
sudo apt install zola
```

To use Zola you have to initialize a directory first:

```sh
zola init mywebsite
```

For convenience install a [theme](https://www.getzola.org/themes/):

```sh
cd website/themes
git clone https://github/myzolatheme
```

Edit the `config.toml` and set the preferences to your liking.
And write your first post:

```sh
vim content/myfirstpost
```
You don't have to add metadata to your posts but the opening/closing `+++` (brackets) are required. `myfirstpost`:
```md
+++                                 
title = "My first post with Zola"   #optional
+++                                 
```

You can serve the site locally to see what it looks like:

```sh
cd website
zola serve
```
Finally build the site with:

```sh
zola build 
```

Now host your `website` directory with your favorite webserver and enjoy the loading time of a static website.
