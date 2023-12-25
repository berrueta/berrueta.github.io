---
layout: post
title:  "Terminal setup"
date:   2023-12-26 15:49:29 +1200
category: software
tags: mac zsh terminal
---

# My terminal setup

This is not a complete blogpost, just a collection of resources that I use
to setup my Mac terminal. This document may be updated at anytime.

* Homebrew

* iTerm2: `brew install iterm2`. [Additional color schemes can be downloaded](https://iterm2colorschemes.com).

* [mackup](https://github.com/lra/mackup) to keep the dotfiles in sync. Install with `brew install mackup`, and then [configure the location of the dotfiles repository](https://github.com/lra/mackup/blob/master/doc/README.md). Then execute `mackup restore` (assuming it is not the first install).

* [colorls](https://github.com/athityakumar/colorls). Installed as a Ruby gem. Then created an alias in `~/.zshrc` like this:

```zsh
# colorls can be in multiple locations
[[ -f /usr/local/bin/colorls ]] && alias ls="/usr/local/bin/colorls"
[[ -f /usr/local/lib/ruby/gems/3.2.0/bin/colorls ]] && alias ls="/usr/local/lib/ruby/gems/3.2.0/bin/colorls"
```

* [ohmyzsh](https://ohmyz.sh).

* ohmyzsh plugins that can be installed via Hombrew: `brew install zsh-autosuggestions zsh-syntax-highlighting`. Add them to the `.zsh`, like this:

```zsh
[[ -f $HOMEBREW_PREFIX/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh ]] && source $HOMEBREW_PREFIX/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh

[[ -f $HOMEBREW_PREFIX/share/zsh-autosuggestions/zsh-autosuggestions.zsh ]] && source $HOMEBREW_PREFIX/share/zsh-autosuggestions/zsh-autosuggestions.zsh
```

* [powerlevel10k](https://github.com/romkatv/powerlevel10k). Make sure you download the fonts.