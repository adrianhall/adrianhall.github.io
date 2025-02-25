source "https://rubygems.org"

# No longer bundled with ruby
gem "csv" 
gem "ostruct"
gem "logger"
gem "rake"
gem "html-proofer"

# Jekyll requirements
gem "jekyll", "~> 4.4.1"
gem "classifier-reborn"
gem "faraday-retry"
gem "webrick"

# The theme
gem "minimal-mistakes-jekyll", "~> 4.26.1"

# Jekyll plugins
group :jekyll_plugins do
  gem "jekyll-archives"
  gem "jekyll-default-layout"
  gem "jekyll-feed"
  gem "jekyll-gist"
  gem "jekyll-include-cache"
  gem "jekyll-optional-front-matter"
  gem "jekyll-paginate-v2"
  gem "jekyll-relative-links"
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
  gem "jekyll-titles-from-headings"
  gem "jekyll-toc"
  gem "jemoji"
end

platforms :mingw, :x64_mingw, :mswin, :jruby do
    gem "tzinfo", ">= 1", "< 3"
    gem "tzinfo-data"
end

platforms :mingw, :x64_mingw, :mswin do
  gem "wdm"
end

platforms :jruby do
  gem "http_parser.rb", "~> 0.6.0"
end
