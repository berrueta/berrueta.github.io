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

## Using GitHub Codespaces

You can develop and preview this Jekyll site in [GitHub Codespaces](https://docs.github.com/en/codespaces) with no local setup required.

### Getting Started

1. Click the "Code" button in your repository and select "Open with Codespaces".
2. Wait for the Codespace to build and install dependencies automatically.
3. The Jekyll server will be available at port 4000. You can preview your site in the Codespaces browser.

### Customization

The development environment is defined in `.devcontainer/devcontainer.json` and `.devcontainer/Dockerfile`. You can modify these files to add tools or change settings as needed.