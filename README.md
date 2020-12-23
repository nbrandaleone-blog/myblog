# [blog](https://nickaws.net/index.html)
This repo holds my blog, powered by Jekyll.

## Features
- [x] RSS Feed. What URI?
- [x] Disqus comments. Admin = @nbrand
- [ ] Google Analytics
- [x] SEO. What version?
- [ ] Examine Category code/functionality
- [ ] new index page / reorder?

## Install Google Analytics:
- https://desiredpersona.com/google-analytics-jekyll/
- https://michaelsoolee.com/google-analytics-jekyll/
- https://curtisvermeeren.github.io/2016/11/18/Jekyll-Google-Analytics.html
- https://learn.cloudcannon.com/jekyll/google-analytics/

Where is the Jekyll defaults stored?
bundle info --path minima

## Building and deploying the static Jekyll website
1) To run a local web server:
``` sh
	 $ bundle exec jekyll serve
  
   # If I want to serve with Disqus comments
   $ JEKYLL_ENV=production bundle exec jekyll serve
```
2) To build the website
``` sh
	 $ bundle exec jekyll build
	 $ JEKYLL_ENV=production bundle exec jekyll build
```
3) To push to production
```
	 # See rake/Rakefile
	 $ rake deploy
```
4) To create new page. Go into \_posts.
``` sh
	 https://jekyllrb.com/docs/posts/
```
5) If creating a draft post, put it in \_drafts.
``` sh
   $ bundle exec jekyll build --drafts
   $ bundle exec jekyll serve --drafts
```

### Git commands
```sh
git commit -a -m "comment"
git push origin master
git pull
```
