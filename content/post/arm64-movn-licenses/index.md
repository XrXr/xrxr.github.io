+++
# Override this since this file is under
# the "post" directory, and post list filter by `.Type`.
# <https://gohugo.io/templates/lookup-order/>
# Matches up with `layouts/licenses/single.html`
type = "licenses"
url = "/post/arm64-movn/licenses"
publishResources = false
disable_feed = true # exclude fom rss feed

# Why is this not in the arm64-movn directory? because with the post as "index.md",
# this page would be just a resource and not rendered in the end.
# Using "_index.md" makes the post page a Page with `.Kind == "section"` and
# breaks the listing operations like finding all posts. Section posts are not
# in ".RegularPages".
+++

# Licenses for toy on [`arm64-movn`](/post/arm64-movn) post
