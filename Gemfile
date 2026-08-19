# The site is built with Jekyll 4 rather than the github-pages gem (Jekyll
# 3.10) because lesson content contains literal {{ }} and {% %} — Prometheus
# alert templates, Go templates, Go struct literals — and only Jekyll 4
# honours the `render_with_liquid: false` default set in _config.yml.
# Under Jekyll 3 that key is silently ignored, Liquid parses the content, and
# the build fails on the first thing that looks like an unterminated tag.
source "https://rubygems.org"

gem "jekyll", "~> 4.3"
gem "jekyll-theme-cayman", "~> 0.2"

# Mirrors the plugins the github-pages gem loaded implicitly. jekyll-relative
# -links is load-bearing: it rewrites the .md links every lesson uses into
# .html so they resolve on the built site.
group :jekyll_plugins do
  gem "jekyll-relative-links"
  gem "jekyll-optional-front-matter"
  gem "jekyll-readme-index"
  gem "jekyll-default-layout"
  gem "jekyll-titles-from-headings"
  gem "jekyll-seo-tag"
  gem "jekyll-github-metadata"
end
