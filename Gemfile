# 로컬 미리보기 + CI 빌드 공용 Gemfile.
# CI(.github/workflows/pages.yml)가 setup-ruby + `bundle exec jekyll build` 로 이 Gemfile 을 쓴다.
# 로컬은 gem 을 vendor/bundle 에 가둬 설치(vendor/ 와 .bundle/ 은 .gitignore).
source "https://rubygems.org"

gem "jekyll", "~> 4.3"
gem "kramdown-parser-gfm"   # _config.yml 의 markdown: kramdown + input: GFM 에 필요
gem "webrick"               # Ruby 3.x 에서 `jekyll serve` 에 필요 (stdlib 에서 빠짐)
