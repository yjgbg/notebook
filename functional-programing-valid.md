# 以函数式编程思维设计一个通用的数据校验器

## 前言

### 关于校验器

#### JAVA中现有的数据校验机制

JSR303,JSR380,Hibernate Validator一整套解决方案

**使用方式：**Bean上使用注解来描述校验规则，注解对应的ConstraintValidator中的代码根据规则进行校验

##### 第一个问题：扩展困难

**JSR303的扩展机制：**新建一个注解，然后实现一个对应的ConstraintValidator

**问题描述：**这样的扩展机制过于繁琐， ，至少需要创建两个类，然而有效代码可能只有短短几行

**现状：**在我们现在的业务代码中，面对引入的库中没有的校验规则，需要扩展时，更多情况下会放弃这套扩展机制，而是如下图中这样，在代码中进行校验

![image-20210329142551881](C:\Users\12394\AppData\Roaming\Typora\typora-user-images\image-20210329142551881.png)

**结果：**而这种解决方式引发了新问题，校验结果不整齐，通过注解上的规则校验出的结果是BindingResult，自定义校验出的结果又有各式各样的结果返回方式，使得对代码的阅读，理解变得困难

##### 第二个问题：校验规则与数据本身耦合

**问题描述：**校验规则以注解的形式标注在实体上，而实体可能会用在多个接口上，而这多个接口又各自需要有不同的校验规则。

**JSR303的解决方案：**分组校验，在注解中使用group分组，然后校验时，只读取某组的注解进行校验。

**不足:**这使得代码的复杂度更进一步上升，阅读代码变得很困难

![image-20210329143215351](C:\Users\12394\AppData\Roaming\Typora\typora-user-images\image-20210329143215351.png)

##### 这两个问题的理想解决方案（类似下图这样）：

1. 校验规则和实体本身解耦
2. 校验规则可组合
3. 并且对校验规则本身的编写不应当比使用注解的方式复杂
4. 可以很好的描述复杂逻辑（至少强于注解）
5. 以及良好的可扩展性（加入新的校验规则）

![image-20210329143622530](C:\Users\12394\AppData\Roaming\Typora\typora-user-images\image-20210329143622530.png)

### 函数式编程思想

函数指数学中的“函数”，代表的是**从自变量到因变量的映射**

那么校验可以看作：**数据对象（自变量）到Errors类型（因变量）的映射**(有的对象校验完之后可能是没有错误的，应该用一个特定的错误类型对象，如ZERO或者NONE)

而**校验器本身是校验的规则**

## 正文

### 空校验器

```java
// 既然校验器是一个从数据对象到Errors类型的映射，那么：
public interface Validator<A> extends Function<A,Errors>
// 空校验器与空错误，定义如下：
// 空错误
public class Error {
  public static Error NONE = new Error(); 
  public static Error none() {
    return NONE;
  }
}
// 空校验器
Validator<A> none = obj -> Errors.none();
```

### 简单校验器

```java
// 更新Errors定义如下
public class Error {
  Object rejectValue;
  String message;
  static Error none();
  static Error simple(Object rejectValue,String message);
}
//创建一个简单字符串校验器，校验字符串长度不大于5
Validator<String> stringValidator = str -> {
  if(str.length() <= 5) {
     return Errors.none();
  } else {
     return Errors.simple(str,"长度大于5");// 第一个参数为错误的值，第二个参数为对应的信息
  }
}
//创建一个简单对象校验器，校验对象不为空
Validator<Object> stringValidator = obj -> {
  if(obj != null) {
     return Errors.none();
  } else {
     return Errors.simple(str,"对象不可为空");// 第一个参数为错误的值，第二个参数为对应的信息
  }
}
// 观察，发现，简单校验器具有都具有相似的结构，由一个布尔表达式判断是否有错，
// 如果有错则用被检验的目标对象和一个字符串创建一个错误，于是抽象简单校验器如下
public static Validator<A> simple(String message,Predicate<A> constraint) {
  return a -> constraint.test(a) ? Errors.none() : Errors.simple(a,message);
}
// 简单对象的校验
Validator<String> lengthGt3 = Validator.simple("长度应当小于3",str -> str.length <3);
lengthGt3.apply("alice"); // Errors.none();
lengthGt3.apply("xm"); // Errors.simple("xm","长度应当小于3");
```



### 两类复杂校验器

#### A类复杂校验器

**，已有一个校验规则为string长度小于3的校验器lengthGt3和规则为不带有空格的校验器havNotBlank,需要一个校验规则为string长度小于3，并且不带有空格的校验器**

```java
// 校验器lengthGt3：长度小于3
Validator<String> lengthGt3 = Validator.of("长度应当小于3",str -> str.length <3);
// 校验器havNotBlank:不含有空格
Validator<String> havNotBlank = Validator.of("应当没有空格",str -> Objects.equals(str,str.replaceAll(" ","")));
// 所需要的校验器应该是上面两个校验器相加
Validator<String> plus = Validator.plus(lengthGt3,noBlank);// 需要有这样一个函数，将两个校验器的规则加起来

// 思路：两个校验器（v0，v1）分别校验，然后将各自的校验结果相加。
public static Validator<A> plus(Validator<A> v0,Validator<A> v1) {
  return a -> Errors.plus(v0.apply(a),v1.apply(a));
}
// 前面定义的简单错误已经不够用了，我们需要可以相加的错误（A类复杂错误），需要可以容纳多条message
```

#### A类复杂错误

```java
//考虑Errors的结构
public class Errors {
  Object rejectValue;
  Set<String> message;
  // 提供如下构造方式:
  publib static Errors none(); // 表示该对象没有错误
  public static Errors simple(Object rejectValue,String message); // 表示该对象发生了一个错误，两个参数为该对象，和错误信息
  public static Errors plus(Errors e0,Errors e1) {
    assert rejectvalue0 == rejectvalue1;
    return new Error(rejectValue0,Set.union(message0,message1));
  }
}
// 将这种复杂错误成为A类复杂错误: 有rejectValue，且message集合中有多个元素的错误
```

#### B类复杂校验器

已有校验器`Validator<String> lengthGt3`,需要构造校验器`Validator<Entity1>`，

要求对Entity1中的字段String name的校验规则为lengthGt3

其中Entity1定义如下:

```java
public class Entity1 {
  String name;
}
Validator<String> lengthGt3 = Validator.of("长度应当小于3",str -> str.length <3); //这是已经有的校验器
求: Validator<Entity1>
```

```java
// 此时需要一个变换函数，可以使得校验器在不同的目标类型之间变换，关键在于转换结果的变换
// 要创建一个变化函数，则需要一个参数，可以表达出name这个属性的属性名以及如何取值
// 知识点:Getter.propertyName
// 如下：将A对象转化为B对象，使用origin进行校验，校验之后将返回的Errors结果转换回来
public static Validator<A> transform(Getter<A,B> prop,Validator<B> origin) {
  return a -> Errors.transform(prop.propertyName(),origin.apply(prop.apply(a)))
}
// 此时Errors的定义又不够用，需要一个可以进行transform运算的Errors
```

#### B类复杂错误

```java
//思考Errors的结构，需要可以表示属性错误
public class Errors { // 完全体的错误对象
  Object rejectValue;
  Set<String> message;
  Map<String,Errors> fieldErrors;
}
// 提供如下构造方式:
publib static Errors none(); // 表示该对象没有错误
public static Errors simple(Object rejectValue,String message); // 表示该对象发生了一个错误，两个参数为该对象，和错误信息
public static Errors plus(Errors e0,Errors e1);
public static Errors transform(String prop,Errors origin) { // 以上面的例子理解：String类型的校验错误，转换成
  return new Errors(null,Set.empty(),Map.of(prop,origin));
}
// 将这种复杂错误成为B类复杂错误，
```

### 阶段总结

Errors对象应该有4种构造方式

```java
publib static Errors none(); // 空错误
// 简单错误：只有rejectValue和message有值，且message为只有一个元素的set
public static Errors simple(Object rejectValue,String message); 
public static Errors plus(Errors err0,Errors err1); // A类复杂错误
public static Errors transform(String prop,Errors errors); // B类复杂错误

//其中transform和plus甚至是满足transform对plus的分配律的。即：
plus(transform(prop,a),transform(prop,b)) === transform(prop,plus(a,b))
```

对应校验器有4种构造方式:

```java
// 空校验器:无论如何校验结果都为Errors.none(),空错误
public static Validator<A> none(); 
// 简单校验器：校验结果为简单错误
public static Validator<A> simple(String message,Predicate<A> constraint);
// A类复杂校验器：校验结果为A类复杂错误
public static Validator<A> plus(Validator<A> validator0,Validator<A> validator1);
// B类复杂校验器：校验结果为B类复杂错误
public static Validator<A> transform(Getter<A,B> prop,Validator<B> validator0);

// transform和plus也是满足transform对plus的分配律的。即：
plus(transform(prop,a),transform(prop,b)) === transform(prop,plus(a,b))
```

### 新的问题

根据如上4个校验器的构造方式：

我们可以构造出各种各样的校验器，而且理论似乎接近完美，但是，像下面这样只通过这4个构造方式去构造复杂校验器，与我们理想中的校验器却相差甚远

```java
// 声明要校验的目标
    final var entity0 = new SampleEntity( 0L,"l", Gender.MALE,"12345678910","abc@def.com");
// 先声明若干个属性类型的校验器
		final var nonNull = Validator.<SampleEntity>simple(Objects::nonNull,"person不能为空");
		final var idValidator = Validator.<Long>simple( id -> id!=null && id > 0L,"id应该大于0");
		final var nameValidator = Validator.<String>simple(name -> name!=null && name.length() >= 2 && name.length() <5"name应该是2-5个字");
		final var genderValidator = Validator.<Gender>simple(gender -> Objects.equals(gender,Gender.FEMALE),"应该是女性");
		final var phoneValidator = Validator.<String>simple(x -> x == null || Pattern.matches("^\\d{11}$",x),"电话号码应该为11位数字");
		final var emailValidator = Validator.<String>simple(x -> x != null && Pattern.matches("^\\w+([-+.]\\w+)*@\\w+([-.]\\w+)*\\.\\w+([-.]\\w+)*$",x),"email格式非法");
// 将它们transform到目标实体类SampleEntity
		final var id = Validator.transform(SampleEntity::getId,idValidator);
		final var name = Validator.transform(SampleEntity::getName,nameValidator);
		final var gender = Validator.transform(SampleEntity::getGender,genderValidator);
		final var phone = Validator.transform(SampleEntity::getPhone,phoneValidator);
		final var email = Validator.transform(SampleEntity::getEmail,emailValidator);
// 再将这些校验器相加
		final var validator0 = Validator.plus(nonNull,id);
		final var validator1 = Validator.plus(validator0,name);
		final var validator2 = Validator.plus(validator1,gender);
		final var validator3 = Validator.plus(validator2,phone);
		final var validator4 = Validator.plus(validator3,email);
// 对目标对象进行校验
		final var errors0 = validator4.apply(entity0);
// 这样的代码既不好写，也不好读
```

因此，需要一些胶水代码（如下），封装一些易用的API，使我们生活可以不这么艰难

### 解决方法

[lombok的ExtensionMethod特性](https://projectlombok.org/features/experimental/ExtensionMethod) ：允许给已经存在的类添加实例方法

原理： 编译期修改代码，将不存在的实例方法调用转换为对应的静态方法调用

定义如下扩展：

```java
	public static <A> Validator<A> and(Validator<A> that, Validator<? super A> validator) {
		return Validator.plus(that, validator);
	}

	public static <A> Validator<A> and(Validator<A> that, String message, Predicate<@Nullable A> constraint) {
		return Validator.plus(that, Validator.simple(message, constraint);
	}


	public static <A, B> Validator<A> and(Validator<A> that, Getter<A, B> prop, Validator<B> another) {
		return Validator.plus(that, Validator.transform(prop, another));
	}

	public static <A, B> Validator<A> and(Validator<A> that, Getter<A, B> prop, String message,
	                                      Predicate<@Nullable B> constraint) {
		return Validator.plus(that, Validator.transform(prop, Validator.of(message,constraint)));
	}
```

通过这些扩展函数，我们可以将上面but下面的那一大段用如下的方式表达：

```java
		final var entity0 = new SampleEntity( 0L,"l", Gender.MALE,"12345678910","abc@def.com");
		final var validator0 = Validator.<SampleEntity>none()
				.and("person不能为空", Objects::nonNull)
				.and(SampleEntity::getId,"id应该大于0,而不是%s", id -> id!=null && id > 0L)
				.and(SampleEntity::getName,"name应该是2-5个字", name -> name!=null && name.length() >= 2 && name.length() <5)
				.and(SampleEntity::getGender,"应该是女性", gender -> Objects.equals(gender,Gender.FEMALE))
				.and(SampleEntity::getPhone,"电话号码应该为11位数字", x -> x == null || Pattern.matches("^\\d{11}$",x))
				.and(SampleEntity::getEmail,  "email格式非法", x ->
						x != null && Pattern.matches("^\\w+([-+.]\\w+)*@\\w+([-.]\\w+)*\\.\\w+([-.]\\w+)*$",x));
		final var errors0 = validator0.apply(false,entity0);
		System.out.println(errors0.toMessageMap());
```



常用的校验函数:

[LbkExtValidatorsStd.java](https://github.com/yjgbg/fun-valid/blob/simplify/src/main/java/com/github/yjgbg/fun/valid/ext/LbkExtValidatorsStd.java)

再次简化如上代码(从代码量上看，这一层的简化并没有啥效果，不过代码的可读性获得了比较高的提升):

```java
final var entity1 = new SampleEntity( 0L,"null", Gender.FEMALE,"1234567891011","asdqq.com");
		final var validator1 =Validator.<SampleEntity>none()
				.nonNull("person不能为空")
				.gt(SampleEntity::getId,"id应该大于0",0L)
				.length(SampleEntity::getName,"name应该是2-5个字",2,5)
				.equal(SampleEntity::getGender,"应该是男性",Gender.MALE)
				.regexp(SampleEntity::getPhone,"电话号码不合法", true, "^\\d{11}$")
				.regexp(SampleEntity::getEmail,"email不合法", false,
						"^\\w+([-+.]\\w+)*@\\w+([-.]\\w+)*\\.\\w+([-.]\\w+)*$");
		final var errors1 = validator1.apply(false,entity1);
		System.out.println(errors1.toMessageMap());
```

## 大功告成

### fun-valid 指北

#### 安装Lombok-EAP插件

[下载链接](https://github.com/mplushnikov/lombok-intellij-plugin/files/5505383/lombok-plugin-0.34-EAP.zip)

正式版本Lombok插件暂时不支持ExtensionMethod，需要使用EAP版(即上面链接里的版本)

#### 配置好公司的maven仓库

此处不再赘述,参见[Maven 和 Hypers Nexus 仓库](https://tech-blog.hypers.cc/article/15)，注意需要配置SNAPSHOT仓库

#### maven引入依赖

```xml
<dependency>
    <groupId>com.github.yjgbg</groupId>
    <artifactId>fun-valid</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

#### 查看代码示例

```java
com.github.yjgbg.fun.valid.Sample
```

[回到顶部](#以函数式编程思维设计一个通用的数据校验器)