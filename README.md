Everts blog
===========

You're looking at the source code of my brand-spanking new blog.
Feel free to fork and mess around with it.

Installation instructions
=========================

Make sure you've got a clone of the source code.
After that, we need a few ruby dependencies:

  sudo gem install jekyll
  sudo gem install github-pages

On Ubuntu, you might need to run the following first:

  sudo apt install zlib1g-dev ruby ruby-dev
```

Then, from the root of the blog just run:

```
jekyll serve --watch
```

This will typically start a server on port 4000.
