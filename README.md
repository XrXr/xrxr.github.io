## How to build and deploy

- `git submodule init && git submodule update --recursive` to grab the theme
- `hugo server --buildDrafts --disableFastRender --renderToMemory` to preview changes
- `hugo` to build
- `ruby -run -ehttpd docs/` to preview the built site
- commit and push
