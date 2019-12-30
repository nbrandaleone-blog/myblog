#
# Jekyll theme. https://github.com/yous/whiteglass
#

1) To run local web server:
	bundle exec jekyll serve
2) To build the website
	bundle exec jekyll build
3) To push to production
	# see rake/Rakefile
	rake deploy
4) To create new page. Go into _posts.
	https://jekyllrb.com/docs/posts/

5) If creating a draft post, put it in _drafts.
bundle exec jekyll build --drafts
bundle exec jekyll serve --drafts

# Git commands
git push origin master
git pull
