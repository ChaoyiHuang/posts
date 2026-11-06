# posts

Personal blog posts, published on GitHub Pages with the [Just the Docs](https://github.com/just-the-docs/just-the-docs) theme.

**Site:** <https://chaoyihuang.github.io/posts/>

## Posts

- [Software Transformation in Agentic Era](blog/software-transformation-in-agentic-era.md) (2025-02-08)

## Layout

```
_config.yml   Jekyll config (Just the Docs theme, site URL/baseurl)
index.md      Home page
blog/         One file per post, with Just the Docs front matter
              (title / parent: Blog / nav_order / date / description)
blog/assets/  Images and downloadable files referenced by posts
```

To add a new post: create `blog/my-post.md` with front matter like the existing
post (`parent: Blog`, next free `nav_order`), then commit and push — the post
appears automatically in the sidebar and on the Blog page.

## Local preview (optional)

The site builds on GitHub Pages automatically; to preview locally:

```sh
gem install jekyll jekyll-remote-theme
jekyll serve
```

Then open <http://localhost:4000/posts/>.

## License

Content is licensed under the [Apache License 2.0](LICENSE).
