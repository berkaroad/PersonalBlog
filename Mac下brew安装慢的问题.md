# Mac下brew安装慢的问题

brew默认源使用的是github，可以设置环境变量达到切换源的效果，见官网： https://github.com/Homebrew/install

```bash
export HOMEBREW_BREW_GIT_REMOTE="..."  # put your Git mirror of Homebrew/brew here
export HOMEBREW_CORE_GIT_REMOTE="..."  # put your Git mirror of Homebrew/homebrew-core here
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

可以切为清华大学源 (https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/)

```bash
# mac
export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew"
export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core"
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"

# linux
export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew"
export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/linuxbrew-core"
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

或 腾讯源

```bash
# mac
export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.cloud.tencent.com/homebrew/brew"
export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.cloud.tencent.com/homebrew/homebrew-core"
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"

# linux
export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.cloud.tencent.com/homebrew/brew"
export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.cloud.tencent.com/homebrew/linuxbrew-core"
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

这样不会因为github网络不稳定导致下载失败了。（PS：github的文件存储，使用的aws s3存储，国内访问比较慢）。

当然，如果之前已经安装了github的源，那么可以通过修改git的远端地址即可。

```bash
cd "$(brew --repo)"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git

cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git

cd "$(brew --repo)"/Library/Taps/homebrew/homebrew-cask
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/homebrew-cask.git

cd "$(brew --repo)"/Library/Taps/homebrew/homebrew-cask-versions
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-cask-versions.git
```

brew 软件包镜像地址

```bash
export HOMEBREW_BOTTLE_DOMAIN="https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles"
```

另附上brew的其他软件包源：

```bash
# mongodb
brew tap mongodb/brew
```

