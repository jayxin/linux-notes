# Font

## 基础

### 核心组件

`fontconfig` 是 Linux 系统的字体配置和管理框架，
负责字体的发现、匹配、渲染等功能。
几乎所有 Linux 发行版都预装了 `fontconfig`，
它通过读取配置文件 (如 `/etc/fonts/fonts.conf`) 和扫描字体目录，
为应用程序提供统一字体访问接口。

### 常用字体目录

Linux 字体存储在特定目录，`fontconfig` 会自动扫描这些目录。

| 路径                     | 作用范围         | 权限         | 适用场景                         |
|--------------------------|------------------|--------------|----------------------------------|
| `~/.local/share/fonts`   | 当前用户(用户级) | 普通用户权限 | 个人字体，不影响其他用户         |
| `/usr/local/share/fonts` | 所有用户(系统级) | root 权限    | 本地管理员添加的系统字体         |
| `/usr/share/fonts`       | 系统级别         | root 权限    | 系统预装或通过包管理器安装的字体 |

### 支持的字体格式

- TrueType(`.ttf`): 广泛使用的轮廓字体格式，兼容性好
- OpenType(`.otf`): 基于 TrueType 扩展，支持更复杂的排版功能(如连笔、多语言字符)
- Type 1(`.pfb`, `.pfa`): 较早的专业字体格式
- Web Open Font Format(`.woff`, `.woff2`): 主要用于网页

## 准备

## 安装 fontconfig

```sh
# Debian
apt install fontconfig

# Fedora
dnf install fontconfig

# Arch
pacman -S fontconfig
```

### 检查现有字体

```sh
# 查看已安装的中文字体
fc-list :lang=zh
```

### 获取字体文件

字体来源:
- **官方软件源**: 通过包管理器安装
- **可信第三方平台**
    + [Google Fonts](https://fonts.google.com/)
    + [Adobe Fonts](https://fonts.adobe.com/)
    + [文泉驿](http://wenq.org/)
    + etc.
- 个人备份

## 字体安装

### 用户级安装

```sh
# 1. 创建用户字体目录
mkdir -p ~/.local/share/fonts

# 2. 复制字体文件到用户字体目录
mkdir -p ~/.local/share/fonts/chinese/{sans,serif}
cp ~/Downloads/SourceHanSansCN-Regular.ttf \
    ~/.local/share/fonts/chinese/sans

# 3. 更新字体缓存
# -f : force
# -v : verbose
# -r : recurse
fc-cache -fvr
```

### 系统级安装

```sh
# 1. 复制字体到系统目录
sudo mkdir -p /usr/local/share/fonts/sans
sudo cp ~/Downloads/Roboto-Regular.ttf /usr/local/share/fonts/sans/

# 2. 修改文件和目录权限，保证所有用户可读
sudo chmod 644 /usr/local/share/fonts/sans/Roboto-Regular.ttf
sudo chmod 755 /usr/local/share/fonts/sans

# 3. 更新系统字体缓存
sudo fc-cache -fvr
```

### 包管理器安装

```sh
# Debian
apt search fonts-noto
sudo apt install fonts-noto-sans
sudo apt install fonts-wqy-microhei

# Fedora
dnf search google-noto-serif
sudo dnf install google-noto-serif-simplified-chinese-fonts

# Arch
yay -S ttf-google-fonts-git
yay -S ttf-fangzheng-lantinghei
```

## 验证字体安装

### CLI 验证

```sh
# 查看输出是否包含字体路径
fc-list | grep "Source Han Sans CN"
```

### GUI 验证

打开文字处理软件 (如 LibreOffice Writer) 或图像软件 (如 GIMP)，
在字体选择栏中搜索目标字体，确认能否正常显示。
