# Repo for my personal digital garden

Created using [Quartz](https://quartz.jzhao.xyz/)

Uses GitHub Actions to build and deploy the site to https://diwashrai.github.io/digital-garden/

# Setup

Personal notes to get the project setup for myself

## Fedora/Mac
```
sudo dnf install go # fedora
sudo brew install go # mac
go install github.com/jackyzha0/hugo-obsidian@latest
```
Append the following lines in the .zshrc file
```
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```

