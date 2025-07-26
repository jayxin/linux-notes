<!-- vim-markdown-toc GFM -->

* [Input Method](#input-method)
    * [Install fcitx5 in Fedora](#install-fcitx5-in-fedora)

<!-- vim-markdown-toc -->

# Input Method

## Install fcitx5 in Fedora

```bash
sudo dnf install fcitx5-configtool fcitx5-gtk{,2,3,4} \
    fcitx5-qt fcitx5-data fcitx5-rime fcitx5-autostart

sudo dnf install gnome-shell-extension-appindicator
gnome-extensions enable appindicatorsupport@rgcjonas.gmail.com
```
