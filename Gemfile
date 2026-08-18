source "https://rubygems.org"

# GitHub Pages가 실제 배포에 쓰는 것과 동일한 gem 묶음을 사용합니다.
# 로컬에서 `bundle exec jekyll serve`로 미리보기할 때 필요합니다.
gem "github-pages", group: :jekyll_plugins

group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
end

# Windows 및 JRuby 환경 호환용 (GitHub Pages 기본 안내에 포함되는 설정)
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]
