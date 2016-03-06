+++
date = "2013-02-02T17:11:18+09:00"
draft = false
title = "Octopressで記事が作れない(zsh)"

+++

zshを使うようになってからOctopressで記事を作成するときにエラーが出るようになってしまった。


```
$ rake new_post[hoge]
zsh: no matches found: new_post[hoge]
```

<!--more-->

ググったら２つ解決策を発見。  
<a href="https://github.com/imathis/octopress/issues/117" target="_blank">https://github.com/imathis/octopress/issues/117</a>

#### 1. aliasを設定する
```
$ alias rake="noglob rake"
```

#### 2. クォーテーションで囲む
```
$ rake "new_post[hoge]"
```
