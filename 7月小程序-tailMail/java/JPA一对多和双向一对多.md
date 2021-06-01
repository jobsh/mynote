# JPA一对多和双向一对多

Bannner和BannerItem是一对多的关系

## 单向一对多

### Banner Entity

```java
@Entity
public class Banner extends BaseEntity {
    @Id
    private Long id;
    private String name;
    private String description;
    private String title;
    private String img;

    @OneToMany(fetch = FetchType.LAZY)
    @JoinColumn(name="bannerId")
    private List<BannerItem> items;
}
```

### BannerItme Entity

```java
@Entity
public class BannerItem extends BaseEntity {
    @Id
    private Long id;
    private String img;
    private String keyword;
    private short type;
    private Long bannerId;
    private String name;
}
```

> 核心配置：
> @OneToMany(fetch = FetchType.LAZY)
>
> 指定关联id：否则JPA会生成第三张表，对这两张表的关系进行维护
> @JoinColumn(name="bannerId")

## Banner Entity

```java
//orm
//物理外键 逻辑外键 实体与实体关系配置 单表查询

@Entity
public class Banner {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(length = 16)
    private String name;

    @Transient
    private String description;
    private String img;
    private String title;

    @OneToMany(mappedBy = "banner", fetch = FetchType.LAZY)
    private List<BannerItem> items;
}
```

## BannerItem Entity

```java
/**
 * @作者 7七月
 * @微信公号 林间有风
 * @开源项目 $ http://talelin.com
 * @免费专栏 $ http://course.talelin.com
 * @我的课程 $ http://imooc.com/t/4294850
 * @创建时间 2020-02-08 05:31
 */
package online.loopcode.tailmall.model;

import javax.persistence.*;

//sql 2条sql语句 查询2次数据库
//sql 1次 join
//PHP 实体模型

@Entity
public class BannerItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String img;
    private String keyword;
    private Short type;
    private String name;

     /**
     * 双向一对多
     * 一 => 关系被维护端
     * 多 => 关系维护段
     * @JoinColumn 必须打在关系维护段(多端)，一端需要在@OneToMany(mappedBy = "banner")加上mappedBy
     * @JoinColumn打在多端的原因：
     * 要根据name = "bannerId"  生成表的外键字段banner_id， 因为这个原因 private Long bannerId; 是可以不写的
     * 多端可以通过@JoinColumn 的name 值知道它属于那个banner
     *
     * 一端可以通过@OneToMany(mappedBy = "banner") 的mappedBy 定位到banner导航属性，而banner是由JoinColume的
     * 也可以知道这个bannner下面有哪些bannnerItem
     *
     *
     * 纠错：@JoinColumn这行注释掉，同时也注释掉private Long bannerId;（不然会报错）
     *      也可以在banner_item表中生成banner_id
     *
     * 问题: 不加@JoinColumn也可以生成banner_id外键，那@JoinColumn作用是什么
     *
     */
//  private Long bannerId;
    
    @ManyToOne
//    @JoinColumn(foreignKey = @ForeignKey(value = ConstraintMode.NO_CONSTRAINT), insertable = false, updatable = false,name = "bannerId")
    private Banner banner;
}
```

## note

> 双向一对多
>      一 => 关系被维护端
>      多 => 关系维护段
>
> ​     @JoinColumn 必须打在关系维护段(多端)，一端需要在@OneToMany(mappedBy = "banner")加上mappedBy
>
> ​     @JoinColumn打在多端的原因：
> ​     要根据name = "bannerId"  生成表的外键字段banner_id， 因为这个原因 private Long bannerId; 是可以不写的，多端可以通过@JoinColumn 的name 值知道它属于那个banner，一端可以通过@OneToMany(mappedBy = "banner") 的mappedBy 定位到banner导航属性，而banner是有JoinColume的，也可以知道这个bannner下面有哪些bannnerItem
>
> ​     纠错：@JoinColumn这行注释掉，同时也注释掉private Long bannerId;（不然会报错）也可以在banner_item表中生成banner_id，这是JPA自身的机制生成的
>
> 问题: 不加@JoinColumn也可以生成banner_id外键，那@JoinColumn作用是什么

