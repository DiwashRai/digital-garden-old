# Repo for my personal digital garden

Created using [digital-garden-jekyll-template](https://github.com/maximevaillancourt/digital-garden-jekyll-template)

Uses GitHub Actions to build and deploy the site to https://diwashrai.github.io/digital-garden/

# Setup

Personal notes to get the project setup for myself

## Fedora
```
sudo dnf install ruby ruby-devel
bundle config set --local path 'vendor/bundle'
bundle install
# Copy libsass.so to correct location
bundle exec jekyll serve
```

