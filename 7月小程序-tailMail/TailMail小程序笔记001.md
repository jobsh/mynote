## TailMail小程序笔记 1-2-7 

### 修改小程序默认首页

![image-20210129193418869](E:\mynote\7月小程序-tailMail\TailMail小程序笔记001.assets\image-20210129193418869.png)

新建home后首页仍然是index

需要去修改app.json文件 的pages属性，小程序**普通编译模式下**默认pages第一行是首页， 需要把`"pages/home/home" `放到第一行

```json
{
  "pages": [
    "pages/index/index",   // 默认第一行首页
    "pages/logs/logs",
    "pages/home/home"
  ],
  "window": {
    "backgroundTextStyle": "light",
    "navigationBarBackgroundColor": "#fff",
    "navigationBarTitleText": "Weixin",
    "navigationBarTextStyle": "black"
  },
  "style": "v2",
  "sitemapLocation": "sitemap.json"
}
```

## 小程序page中的js文件里面应该写什么?

小程序中的Page中的js**不应该去写业务逻辑**，而应该只是去做**数据绑定**，页面的js应该充当view视图层的角色，相当于Model和Html（小程序中就是wxml）的桥梁（中间层），可以类比于后端的Controller层

model层：业务对象层，可以类比于后端的model层，存放的是一个个的bean
theme、 banner、spu、sku、 address、user

