task default: %w[build deploy]

task :build do
  puts "Building static web site using jekyll generator"
  `bundle exec jekyll build`
end

task :deploy do
  puts "Deploying to S3"
  `cd _site; aws s3 sync . s3://www.nickaws.net --profile personal`
  `aws cloudfront create-invalidation --distribution-id "E31FMCWWZS8AG1" --paths "/*" --profile personal`
end

task :web do
  `bundle exec jekyll serve`
end

