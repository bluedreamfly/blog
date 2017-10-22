---
title: javascript decorator(装饰器) 一探
date: 2017-10-26 10:46:55
categories: ['技术', '模式']
---

今天我们要讲的并不是什么非常高深的设计模式，我们想从`ECMASCRIPT`草案的`装饰器`中延伸一些相关概念，以及在项目的一些实践。
其实说白了，javascrpt提出的装饰器也只不过是语法糖而已。但是我们都知道，其实`ES6`中很多
特性都只是在`ES3`和`ES5`的基础上进行封装。总的来说，形式不是很重要，重要的是能解决我们当前的问题。`ES6`可以提高开发效率及更好规范代码，
这才是我们选择的重点。而`装饰器`同样是因为同样的原因产生的。先来看段代码，当然下面并没覆盖装饰模式的所有功能，只是举个例子。

```java
@Data
public class User implements Serializable {

  private String id;

  //名字不能为空
  @NotNull(message = "姓名不能为空")
  private String name;

  //年龄最小10最大20 不在此范围报错
  @Min(10) 
  @Max(20)
  private Integer age;

  //原来要写字段的getter和setter因为有了@Data注解，不用写，是不是很方便
  public String getId() {
    return id;
  }
  public void setId(String id) {
    this.id = id;
  }
}

```

是不是觉得突然间觉得写代码也能很舒服，当然这是`java`的语法，是不是突然想拥抱`java`了，哈哈，开个玩笑。但是js中的`装饰器`完全也可以做到这点，前端开发者是不是也很激动。但是别高兴太早，这种语法暂时还只能借助编译器来完成，并不原生支持。但至少可以在我们的项目中使用了，这是没问题的。是不是手痒了， 那现在介绍一下装饰器在js中的使用。但是需要借助编译器，我用的是`babel`和相应的插件`transform-decorators-legacy`, 具体怎么用，这里我就不赘述了。

```javascript
//decorator
//这个就是对应的java @Data注解的功能
const Data = (target, key, descriptor) => {

  let temp = new Function('className', `return new className()`);
  let obj = temp(target)
  let keys = Object.keys(obj);

  keys.forEach(key => {
    let name = key.slice(1);
    Object.defineProperty(target.prototype, name, {
      configurable: true,
      set(val) {
        this[key] = val;
      },
      get() {
        return this[key];
      }
    })
  })
}

const common = data => {
  return (target, key, descriptor) => {
    let name = key.slice(1);

    let originSet;
    if (name in target) {
      originSet = Object.getOwnPropertyDescriptors(target)[name].set;
    }
    
    Object.defineProperty(target, name, {
      configurable: true,
      set(val) {
        originSet && originSet.call(this, val)
        let condition = data.condition;
        if(condition(val)) {
          throw new Error(data.message)
        }
        this[key] = val
      },
      get() {
        return this[key]
      }
    })
  }
}

const NotNull = (message) => {
  let data = {
    condition: (value) => {
      return !value
    }, 
    message: message || '不能为空'
  }
  return common(data);
}
const Min = (minValue, message) => {
  let data = {
    condition: (value) => {
      return value < minValue
    },
    message: message || `value is less ${minValue}`
  }
  return common(data);
}
const Max = (maxValue, message) => {
  let data = {
    condition: (value) => {
      return value > maxValue
    },
    message: message || `value is great than ${maxValue}`
  }
  return common(data);
}
@Data
class User {
  
  //约定私有 暴露通过id
  _id = '',

  @NotNull('姓名不能为空')
  _name = '',

  @Min(10)
  @Max(20)
  _age = ''

}
```

这样基本上就可以实现java的注解功能了，是不是突然发现不用再写很多if或者写pattern来验证数据了。除此之外可读性是不是也提高了不少。而且像前端这样偏数据展示的，用这种方式其实是更简洁的。

当然上述@Data是作用在类上面的，所以target指向的是User。而像@NotNull,@Min, @Max这样的装饰器则是作用在实例变量或者类属性上面的，target指向的是User.prototype。


在我们实际的项目中可以怎么用呢？因为API数据是原子性比较强的，不能在这一层就满足各种不同页面需要的数据格式，我们可以有一个基础的API数据model，然后通过继承基础model，进行字段拓展，然后新增一些字段用装饰器进行装饰。

```javascript

class BaseUseModel {
  _id = '',

  @NotNull('姓名不能为空')
  _name = '',

  @Min(10)
  @Max(20)
  _age = ''

  
  _addTime = ''
}

class UserInfoPageModel extends BaseUseModel {

  @dateFormat('yyyy-MM-dd HH:mm:ss')
  _addTime = ''
}


```

装饰器的基本使用我也就大概讲了一点，其实它可以做很多事情，其实最核心的一点就是功能增强。














