---
layout: post
title:  "Quasi-automatic Tagging"
date:   2016-09-02 01:32:30 +0200
tags:   github-pages jekyll
---

[Jekyll][jekyll] works really great with [GitHub Pages][gh-pages] but once you start playing around with plugins the realization quickly dawns that only a [small subset is supported][gh-plugins]. Therefore, it's easiest to use the `safe: true` [option][jekyll-conf] in "_config.yml" to ensure that when you view it locally it resembles what it will look like on GitHub Pages, because it disables custom plugins and ignores symbolic links.

After having tried several different tagging solutions (like [Jekyll Tagging][jekyll-tagging]) before realizing the above incompatibility, I finally found [an article][minddust] about how to do it without needing any custom plugin.

It was actually really easy:

1. The `post` layout was extended to detect if any tags are associated the current post, and links are generated to point to a new `post_by_tag` layout at `/tag/THETAG/`.

2. For each tag used by posts, the following had to be added to `_data/tags.yml`:

        - slug: thetag
          name: The Proper Name of The Tag

3. Finally, for each tag an empty template had to be created at `tag/thetag.md` containing:

        ---
        layout: post_by_tag
        tag: thetag
        permalink: /tag/thetag/
        ---

As you can see at the bottom of this post it says "_Posted with: .._" with each of the tags associated the post. Clicking such a link takes one to the empty template file `tag/thetag.md` which uses the `post_by_tag` layout to display the information.

Whenever a new post is written two other files must be edited for each of the post's tags, but it's still a small price to pay for a **working, quasi-automatic tagging system!**

[gh-pages]: https://pages.github.com
[gh-plugins]: https://pages.github.com/versions/

[jekyll]: https://jekyllrb.com
[jekyll-conf]: https://jekyllrb.com/docs/configuration/
[jekyll-tagging]: https://github.com/pattex/jekyll-tagging

[minddust]: http://www.minddust.com/post/tags-and-categories-on-github-pages/
