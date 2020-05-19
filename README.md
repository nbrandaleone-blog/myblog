# [blog](https://nickaws.net/index.html)
This repo holds my blog, powered by Jekyll.

1) To run a local web server:
	 $ bundle exec jekyll serve
  
   If I want to serve with Disqus comments
   $ JEKYLL_ENV=production bundle exec jekyll serve

2) To build the website
	 $ bundle exec jekyll build
	 $ JEKYLL_ENV=production bundle exec jekyll build

3) To push to production
	 # see rake/Rakefile
	 $ rake deploy

4) To create new page. Go into \_posts.
	 https://jekyllrb.com/docs/posts/

5) If creating a draft post, put it in \_drafts.
   $ bundle exec jekyll build --drafts
   $ bundle exec jekyll serve --drafts

# Git commands
git commit -a -m "comment"
git push origin master
git pull
