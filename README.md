You can start a `nix-shell` and it will have Jekyll available in it for
rendering the markdown articles locally.

You can put your half done articles in a `./_drafts` folder first, but to
render them, you have to start jekyll with an
[extra option](https://jekyllrb.com/docs/drafts/):

```
jekyll serve --drafts
```

Articles should start with some summary paragraphs, which are shown on the blog
article index page. Jekyll call these _excerpts_. Their boundary is marked by
a `<!--more-->` comment tag.

The beginning of the article should start with a so-called
[YAML front matter](https://jekyllrb.com/docs/frontmatter/) to provide some
meta information about the blog post, mainly the title, so don't start your
text by repeating the title text as a markdown header.
