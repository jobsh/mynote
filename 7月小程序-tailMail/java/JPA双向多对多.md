# JPA双向多对多

theme 和 Spu是多对多的关系

## Theme Entity

```java
//去孤子
@Entity
public class Theme {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    private String name;

    @ManyToMany
    @JoinTable(name = "theme_spu", joinColumns = @JoinColumn(name = "theme_id"),
            inverseJoinColumns = @JoinColumn(name = "spu_id"))
    private List<Spu> spuList;
}
```

## Spu Entity

```java
/**
 * @作者 7七月
 * @微信公号 林间有风
 * @开源项目 $ http://talelin.com
 * @免费专栏 $ http://course.talelin.com
 * @我的课程 $ http://imooc.com/t/4294850
 * @创建时间 2020-02-11 19:46
 */
package online.loopcode.tailmall.model;

import javax.persistence.*;
import java.util.List;

@Entity
public class Spu {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    private String subtitle;

    /**
     * mappedBy必须放在关系的被维护端，
     * mappedBy与@JoinTable或@JoinColume不能同时存在，必须分开，否则会报错
     */
    @ManyToMany(mappedBy = "spuList")
    private List<Theme> themeList;
}
```

单向多对多，只需在一侧配置@ManyToMany就可以了