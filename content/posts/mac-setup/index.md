---
title: "Setup Macbook Pro"
date: 2024-07-16
summary: "Quick setup for Macbook Pro M3 for backend dev"
tags: ["mac", "m3", "setup"]
---

### 1. Install homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Add below lines in ~/.zshrc file (update the {username} before pasting!)

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> /Users/{username}/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

### 2. Install git

```bash
brew install git
```

### 3. Install iTerm2

```bash
brew install --cask iterm2
```

For next step, switch to iTerm2.

### 4. Install Oh My Zsh

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### 5. Install PowerLevel10K Theme

```bash
git clone https://github.com/romkatv/powerlevel10k.git $ZSH_CUSTOM/themes/powerlevel10k
```

Update ~/.zshrc

```bash
ZSH_THEME="powerlevel10k/powerlevel10k"
```

To configure P10K

```bash
p10k configure
```

Increase the font size in the terminal to 20 :)

### 6. Plugins for iTerm2

- Auto-suggestions

```bash
    git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

- Syntax Highlighting

```bash
    git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

- Add the plugins in ~/.zshrc

```
plugins=(git zsh-autosuggestions zsh-syntax-highlighting web-search)
```

- Load the changes

```bash
    source ~/.zshrc
```

Done!
