+++
date = "2012-08-26T17:34:34+09:00"
draft = false
title = "MySQLでインデックスあり・なしの検索速度を比較してみました。"
tags = ["mysql"]
+++

usersとfavoritesが１対多になるようにして、  
indexの設定あり・なしの2つのDBを用意して比較しました。  

<!--more-->

予め、  
usersは 100,000件  
favoritesに300,000件のレコードを用意しました。


#### indexなし
・テーブル作成
```
create table users(
id int(11) not null primary key auto_increment,
name varchar(40) not null
) engine=innodb;

create table favorites(
user_id int(11) not null,
favorite_name varchar(40) not null
) engine=innodb;
```

・select  
![select](https://dl.dropbox.com/u/22083548/octopress/20120826/select_noindex.png)

・explain  
![explain](https://dl.dropbox.com/u/22083548/octopress/20120826/explain_noindex.png)

#### indexあり
・テーブル作成
```
create table users(
id int(11) not null primary key auto_increment,
name varchar(40) not null
) engine=innodb;

create table favorites(
user_id int(11) not null,
favorite_name varchar(40) not null,
foreign key(user_id) references users(id)
) engine=innodb;
```

・select  
![select](https://dl.dropbox.com/u/22083548/octopress/20120826/select_index.png)

・explain  
![explain](https://dl.dropbox.com/u/22083548/octopress/20120826/explain_index.png)

#### 結果
- indexなし
クエリ実行時間 0.38 秒
- indexあり
クエリ実行時間 0.0005秒

explainの結果から、indexがないと  
テーブルのフルスキャンが発生してしまっていることがわかりました。
