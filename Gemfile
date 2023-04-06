source "https://rubygems.org"

# This is the default theme for new Jekyll sites. You may change this to anything you like.
gem "minima"
gem "github-pages", group: :jekyll_plugins
gem "webrick"

group :jekyll_plugins do
  gem "jekyll-feed"
end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", :platforms => [:mingw, :x64_mingw, :mswin]
