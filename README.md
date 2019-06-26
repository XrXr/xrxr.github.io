## How to build and deploy

- `git submodule init && git submodule update --recursive` to grab the theme
- `hugo server --buildDrafts` to preview changes
- `hugo` to build
- `git chckout master` and `cp -r public/* .` to update the branch that's actually served
- commit and push to update the site
