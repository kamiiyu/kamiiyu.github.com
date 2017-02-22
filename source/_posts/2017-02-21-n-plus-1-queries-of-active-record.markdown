---
layout: post
title: "浅谈ActiveRecord的N + 1查询问题"
date: 2017-02-21 11:37:27 +0800
comments: true
categories: ruby
---

## ORM框架的性能小坑

在使用ActiveRecord这样的ORM工具时，常会嵌套遍历model。  
例如，有两个model，Post、Comment，关系是一对多。  

```
class Post < ApplicationRecord
  has_many :comments
end

class Comment < ApplicationRecord
  belongs_to :post
end
```

 总共有4个post。  

```
> Post.count
(0.1ms)  SELECT COUNT(*) FROM "posts"
=> 4
```
获取每个post的所有comment，我们可以：

```
> Post.all.map{|post| post.comments}
  Post Load (0.3ms)  SELECT "posts".* FROM "posts"
  Comment Load (0.2ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 1]]
  Comment Load (0.4ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 2]]
  Comment Load (0.6ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 3]]
  Comment Load (0.6ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" = ?  [["post_id", 4]]
```

可以看到为了得到4条数据，我们执行了5（4 + 1）次的查询，这就是所谓N + 1查询问题。  

## 发现问题

除了凭经验外，有一些gem也可以帮助我们提早发现N + 1查询问题。  
例如收费的[New Relic](https://newrelic.com/)，免费的[Bullet](https://github.com/flyerhzm/bullet)。

## 解决问题

### 预加载

简单来说，就是提前加载model关系，让ActiveRecord预先加载所需要的数据。  
ActiveRecord提供了以下三个方法预加载。  

* `includes`
* `preload`
* `eager_load`

他们的区别可以参考[这里](http://blog.bigbinary.com/2013/07/01/preload-vs-eager-load-vs-joins-vs-includes.html)或[这里](https://ruby-china.org/topics/17866)。  
以最常用的`includes`方法为例。  

```
> Post.includes(:comments).map{|post| post.comments}
  Post Load (0.2ms)  SELECT "posts".* FROM "posts"
  Comment Load (0.5ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN (1, 2, 3, 4)
```
得到的结果一样，但执行的查询只有两次。  

### 傻瓜式预加载(Goldiloader)

#### 传统预加载的“问题”

`includes`方法的确很惊艳，但……  

第一，代码不够优雅。  
例如，假设我们现在想找的是id在1到3之间的post的comment。  
一般的我们的逻辑是，查找id在1到3之间的post，获取各post的comment然后合并。  
而预加载后的逻辑是，查找id在1到3之间的post，<font color='red'>**关联comment，再**</font>获取各post的comment然后合并。  
总觉得有点冗余。  

```
> Post.where(id: 1..3).includes(:comments).map{|post| post.comments}
  Post Load (0.5ms)  SELECT "posts".* FROM "posts" WHERE ("posts"."id" BETWEEN ? AND ?)  [["id", 1], ["id", 3]]
  Comment Load (0.5ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN (1, 2, 3)
```

第二，不符合DRY。  
既然我们都不喜欢N + 1，那就应该从源头上杜绝，而不是每次查询时都要主动`includes`一次。  
虽然我们也可以用`scope`把他藏在角落里，但总还是有点小疙瘩。  

#### Goldiloader

懒癌程序员的救星[Goldiloader](https://github.com/salsify/goldiloader)——几乎完美的解决了以上两个问题。  

```
gem 'goldiloader'
```

`bundle install`以后，就可以用最直接（傻瓜）的方式点点点……  

```
> Post.where(id: 1..3).map{|post| post.comments}
  Post Load (0.2ms)  SELECT "posts".* FROM "posts" WHERE ("posts"."id" BETWEEN ? AND ?)  [["id", 1], ["id", 3]]
  Comment Load (0.3ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN (1, 2, 3)
```

##### auto_include

Goldiloader默认自动加载所有关联数据，用`auto_include: false`可以方便地关闭自动加载。  

```
class Post < ApplicationRecord
  has_many :comments, auto_include: false
end
```

##### fully_load

以下的方法比较特殊，如果关系已经加载了，则会直接返回已缓存在内存中的值，如果没被加载，则会通过SQL查询。  

* first
* second
* third
* fourth
* fifth
* forty_two
* last
* size
* ids_reader
* empty?
* exists?

假设现在我们需要获取每个post的最新的comment。  

```
> Post.all.sum{|post| [post.id, post.comments.last&.content]}
  Post Load (0.1ms)  SELECT "posts".* FROM "posts"
  Comment Load (0.1ms)  SELECT  "comments".* FROM "comments" WHERE "comments"."post_id" = ? ORDER BY "comments"."id" DESC LIMIT ?  [["post_id", 1], ["LIMIT", 1]]
  Comment Load (0.1ms)  SELECT  "comments".* FROM "comments" WHERE "comments"."post_id" = ? ORDER BY "comments"."id" DESC LIMIT ?  [["post_id", 2], ["LIMIT", 1]]
  Comment Load (0.1ms)  SELECT  "comments".* FROM "comments" WHERE "comments"."post_id" = ? ORDER BY "comments"."id" DESC LIMIT ?  [["post_id", 3], ["LIMIT", 1]]
  Comment Load (0.1ms)  SELECT  "comments".* FROM "comments" WHERE "comments"."post_id" = ? ORDER BY "comments"."id" DESC LIMIT ?  [["post_id", 4], ["LIMIT", 1]]
```
这不是我们想要的。  

添加选项`full_load: true`后,当调用上述方法时，Goldiloader会强制自动加载所需的关系。  

```
class Post < ApplicationRecord
  has_many :comments, fully_load: true
end
```

这才是我们想要的。  

```
> Post.all.sum{|post| [post.id, post.comments.last&.content]}
  Post Load (0.3ms)  SELECT "posts".* FROM "posts"
  Comment Load (0.2ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN (1, 2, 3, 4)
```

#### Goldiloader也不是万能的

##### has_one使用SQL limit时的隐患

Goldiloader是ActiveRecord的衍生工具，所以ActiveRecord预加载的副作用也一并继承了。  
现在我们自定义一个`has_one`关系，用以获取最新的一条comment。  

```
class Post < ApplicationRecord
  has_many :comments, fully_load: true
  has_one :latest_comment, -> { order(created_at: :desc) }, class_name: 'Comment'
end
```

遍历post获取最新的comment。  

```
> Post.all.map{|post| post.latest_comment}
```

不使用Goldiloader或者预加载时，每条SQL自动回加上`limit 1`。  

```
Post Load (0.3ms)  SELECT "posts".* FROM "posts"
Comment Load (0.2ms)  SELECT  "comments".* FROM "comments" WHERE "comments"."post_id" = ? ORDER BY "comments"."created_at" DESC LIMIT ?  [["post_id", 1], ["LIMIT", 1]]
Comment Load (0.2ms)  SELECT  "comments".* FROM "comments" WHERE "comments"."post_id" = ? ORDER BY "comments"."created_at" DESC LIMIT ?  [["post_id", 2], ["LIMIT", 1]]
Comment Load (0.1ms)  SELECT  "comments".* FROM "comments" WHERE "comments"."post_id" = ? ORDER BY "comments"."created_at" DESC LIMIT ?  [["post_id", 3], ["LIMIT", 1]]
Comment Load (0.1ms)  SELECT  "comments".* FROM "comments" WHERE "comments"."post_id" = ? ORDER BY "comments"."created_at" DESC LIMIT ?  [["post_id", 4], ["LIMIT", 1]]
```

使用Goldiloader或者预加载时，世界变清净了，但同时会有性能隐患，因为post的数据可能非常大量。  

```
      Post Load (0.5ms)  SELECT "posts".* FROM "posts"
      Comment Load (0.2ms)  SELECT "comments".* FROM "comments" WHERE "comments"."post_id" IN (1, 2, 3, 4) ORDER BY "comments"."created_at" DESC
```

##### 其他限制

遇到以下的关系，Goldiloader会自动关闭自动预加载。  

* limit
* offset
* finder_sql
* group (due to a Rails bug)
* from (due to a Rails bug)
* joins (only Rails 4.0/4.1 - due to a Rails bug)
* uniq (only Rails 3.2 - due to a Rails bug)

## 本文结束之前

N + 1查询问题是一个容易被忽略的问题，`includes`已经够用，Goldiloader更是锦上添花，对新手足够友好。  
不过对于我这种被Rails“坑”习惯的斯德哥尔摩症候群患者来说，没有`includes`反而没安全感了>_<|||  

## 参考文档

* [Eager Loading Associations](http://guides.rubyonrails.org/active_record_querying.html#eager-loading-associations)
* [Goldiloader](https://github.com/salsify/goldiloader)
* [AUTOMATIC EAGER LOADING IN RAILS](http://www.salsify.com/blog/engineering/automatic-eager-loading-rails)
* [本文例子](https://github.com/kamiiyu/n_plus_1_demo)
