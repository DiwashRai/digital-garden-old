# Repo for my personal digital garden

Created using [digital-garden-jekyll-template](https://github.com/maximevaillancourt/digital-garden-jekyll-template)

Uses GitHub Actions to build and deploy the site to https://diwashrai.github.io/digital-garden/

# Setup

Personal notes to get the project setup for myself

## Fedora
```
sudo dnf install go
go install github.com/jackyzha0/hugo-obsidian@latest
go mod init github.com/jackyzha0/hugo-obsidian
```
`in .zshrc file add following lines
```
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```

