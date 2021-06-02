I am using hello-friend-ng for this website. I forked my own version to modify
the theme a little bit. The usual workflow is to push from the submodule dir
and commit that update in the parent directory.

When pushing, there's a github action that generates the site and puts it in
the gh-pages branch. The site is served from there. Note the CNAME record.