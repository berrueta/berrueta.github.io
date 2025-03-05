## Requirements

```bash
brew install rbenv ruby-build

export PATH="$HOME/.rbenv/bin:$PATH"

eval "$(rbenv init -)"

rbenv install 3.3.7
rbenv global 3.3.7
rbenv versions
rbenv rehash

ruby --version
gem env

gem install bundler
bundle install # or bundle update
```

## Running the website

```bash
bundle exec jekyll serve
```

Then open http://localhost:4000/ in your browser.