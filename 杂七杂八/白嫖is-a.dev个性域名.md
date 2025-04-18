# 白嫖is-a.dev个性域名 

## 0.前言

日常冲浪，发现可以白嫖两个个性域名。都是GitHub上的项目，is-a-dev和is-a-good-dev，顾名思义，一个可以注册`.is-a.dev`域名，另一个可以注册`.is-a-good.dev`域名。

## 1.注册步骤

首先，需要fork对应的GitHub仓库。

然后，在domains文件夹中创建一个名为your-domain-name.json的新文件，以注册your-domain-name.is-a.dev或your-domain-name.is-a-good.dev。

提交一个Pull request以供审核。审核通过后即可生效。

听起来很简单，操作也不复杂。但是，亲测is-a-dev的PR审核比较严苛。

两个项目下，我都是添加的CNAME解析，对应的json文件如下

```json
{
  "owner": {
    "username": "Mrliduanyang",
    "email": "duanyangchn@gmail.com"
  },
  "record": {
    "CNAME": "duanyang.cool"
  },
  "proxied": false
}
```

```json
{
  "owner": {
    "username": "Mrliduanyang",
    "email": "duanyangchn@gmail.com"
  },
  "target": {
    "CNAME": {
      "name": "duanyang",
      "value": "duanyang.cool"
    }
  },
  "proxied": false
}
```

再说is-a-dev的PR审核。

- 一开始网页上放了个Hello World，不行，网页不完整，让修改。

- 又让AI随便生成一个页面放上去，还不行，不完整，并且网页内容和开发无关，继续修改。

- 没办法，找了之前写的几篇博客文章，用Hugo生成静态页，放上去，审核通过。

相比之下，is-a-good-dev的审核就宽松多了。

原因可能有两点吧。一是is-a-good-dev没有is-a-dev知名，从PR数量上也能反映，is-a-good-dev只有700+的PR，而is-a-dev有20000+；二可能和审核人员有关，他觉得行就给通过。

PR合并后，域名很快就生效，几乎没等多长时间。

## 2.最后

一直白嫖一直爽，希望项目不要倒了。感恩！
