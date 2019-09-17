# 简介

一直很疑惑为什么很少有人用lombok。自从毕设的时候遇到lombok，越来越觉得这货用起来真TMD方便。

lombok通过在**编译前对代码进行处理**，自动生成代码，使得java开发变得更为简便。

# 用途

在使用java做开发的时候，我们经常需要定义这样的格式对象用于表示对业务中实体的抽象

```java
public class User {

    private Long id;

    private String name;

    private Integer age;

}
```

并给他们创建一系列的get，set，hashcode，equals以及toString这些方法，于是最终代码就成为了这个样子：

```java
public class User {

    private Long id;

    private String name;

    private Integer age;

    public User(){

    }

    public User(Long id,String name,Integer age){
        this.id = id;
        this.name = name;
        this.age = age;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User user = (User) o;
        return Objects.equals(getId(), user.getId()) &amp;&amp;
                Objects.equals(getName(), user.getName()) &amp;&amp;上瘾
                Objects.equals(getAge(), user.getAge());
    }

    @Override
    public int hashCode() {
        return Objects.hash(getId(), getName(), getAge());
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

代码从9行，膨胀到了66行，而且这是只有三个property的情况下，如果有更多的property，情况只会更糟。

于是lombok上场，通过lombok提供的getter，setter，toString以及EqualsAndHashCode注解，代码简化成了如下的样子：

```java
@ToString
@EqualsAndHashCode
public class User {

    @Getter
    @Setter
    private Long id;

    @Getter
    @Setter
    private String name;

    @Getter
    @Setter
    private Integer age;
}
```

甚至可以通过data注解将代码简化到如下：

```java
@Data
public class User {

    private Long id;

    private String name;

    private Integer age;

}
```

一个data注解，lombok就会帮我们生成所有的get/set/hashcode/toString/Equals等方法，极大简化了代码。

# lombok在idea中的安装

## 在项目的build.gradle或pom.xml中导入lombok

### 如果你是用gradle构建项目

```java
// https://mvnrepository.com/artifact/org.projectlombok/lombok
compileOnly group: 'org.projectlombok', name: 'lombok', version: '1.18.4'
```

### 如果你是用maven构建项目

```xml
<!-- https://mvnrepository.com/artifact/org.projectlombok/lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.4</version>
    <scope>provided</scope>
</dependency>

```

## 安装lombok插件

lombok会在编译前将所有的lombok注解进行处理，生成代码，如果不安装插件，那么ide会在代码编写过程中报错（虽然实际上可以编译通过），因此需要安装插件

- File > Settings > Plugins > Browse repositories… > Search for “lombok” > Install Plugin
- 或者可以直接[下载](https://github.com/mplushnikov/lombok-intellij-plugin/releases/latest)文件，然后在File > Settings > Plugins里点击小齿轮，选择Install plugin from disk.

## 编译错误的处理

如果程序在编写时，ide没有任何错误提示，但是编译无法通过，请注意检查以下两图中的设置：

打开enable annotation processing
![annotationProcess](https://www.yjgbg.me/img/lombok/annotationProcessors.png)

Use compiler 为 javac
![compiler](https://www.yjgbg.me/img/lombok/compiler.png)
原理：lombok在编译前将扫描所有的类，发现有lombok相关注解，就根据注解修改类，然后再将修改后的结果送进编译器进行编译。所以需要打开注解处理器(AnnotationProcessor)

# lombok常用注解

`@Setter`注解在成员变量上，表示为该成员变量生成set方法

`@Getter` 注解在成员变量上，表示为该成员变量生成set方法

`@HashCodeAndEquals` 注解在类上，表示为该类生成HashCode方法和Equals方法

`@ToString` 注解在类上，表示为该类生成toString方法，输出该类的所有成员变量。

```
@Data` 注解在类上，相当于对该类注解了`@ToString`,`@HashCodeAndEquals`,对该类的所有成员变量注解了`@Setter`,`@Getter
```

`@AllArgsConstructor` 注解在类上，表示为该类生成全参构造器

`@NoArgsConstructor` 注解在类上，表示为该类生成无参构造器

`@Log` 注解在类上，表示为该类注入一个Log对象

`@NonNull`注解在方法的参数上，表示该参数不该为空

`@Cleanup` 注解在方法内的方法内的变量（如inputStream）上，表示在方法结束时自动close该变量

`val` 可以当关键字用，表示不可变对象（常量）

`var` 当关键字用，表示可变对象（变量）(jdk11不可用，因为jdk11自带了var关键字)

**注**：jdk11中的var和lombok中的var是类型推断，而javascript或者python是弱类型语言，它们的var是变量声明。。。最大的区别是：java中的var只能在声明并赋值在同一行时可以用，而且java中的var在声明为一个对象之后，不可以在赋值为其他类型的对象。而js和python这些弱类型语言则无此限制。