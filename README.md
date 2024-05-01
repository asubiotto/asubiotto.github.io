## Workflow
After making changes, run
```
hugo server -D
```
To run a localhost server to preview the changes.

When pushing, there's a github action that generates the site and puts it in
the gh-pages branch. The site is served from there. Note the CNAME record.

## Theme

I am using hello-friend-ng for this website. I forked my own version to modify
the theme a little bit. The usual workflow is to update the theme (especially
if you're seeing build errors) by rebasing my master branch in the submodule
(./themes/hello-friend-ng) to include the commits in
https://github.com/rhazdon/hugo-theme-hello-friend-ng. Then commit that update in the parent directory of this repo.
