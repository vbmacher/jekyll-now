source 'https://rubygems.org'

require 'json'
require 'open-uri'
versions = JSON.parse(open('https://pages.github.com/versions.json').read)

group :jekyll_plugins do
  gem 'github-pages', versions['github-pages']
  gem 'jekyll', versions['jekyll']
  gem 'jekyll-gist', versions['jekyll-gist']
  gem 'jekyll-commonmark-ghpages', versions['jekyll-commonmark-ghpages']
end
