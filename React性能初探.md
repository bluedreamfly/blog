---
title: React性能初探
date: 2016-09-06 10:47:14
categories: ["技术", "前端"]
tags: ["react", "immutable"]
---

&emsp;&emsp;今天是第一次写博客，虽然用react也有一段时间了，但总感觉只要是关于性能的东西，属于比较有难度的东西。对于初学者，甚至是使用了很长一段时间开发者来说，基本上很多人也都不关心这个东西。因为可能有几种情况才会遇到性能问题，现在的PC开发只要是页面没有十分复杂，基本也不会有性能问题，同时，react一个比较大的亮点就是virtual-dom，速度快，所以开发者对其关注度相对较少。而对于移动端开发，因为移动端的展示区域有限，基本不会涉及到多复杂的界面，所以实际上在普通的开发场景下基本上不会有什么性能问题，除非是作死。

&emsp;&emsp;React一个很重要的特点就是数据驱动，我们不再或者是很少再需要去操作DOM，这实际上是能让我们写出更好的代码和减少bug数量。而当我们在完成需求，基本上就是定义状态，传递属性等等。加上一些react生命周期的使用，就能够满足我们大部分需求了。但对于追求更优代码的我们，我想如果能在性能上有所提升，我想我们还是很乐意做的。而实际上，也并不需要花费多少时间。

&emsp;&emsp;我想在学习react过程当中，react中的生命周期API大家应该都很熟悉吧，但是是否能很好的理顺在何时执行哪个函数，我想又要少了一批人了，react很重要的就是它的生命周期，理解整个生命周期的运行对于后面性能方面的理解是很有帮助的。
<div align="center">
  <img src="http://o8695iplk.bkt.clouddn.com/react-lifecycle.png" width="600" />
</div>


&emsp;&emsp;上图即是react生命周期图，我想这张图可以让你对react生命周期有一个更直观的认识，而我们的性能优化点是在shouldComponentUpdate。每次状态改变我们都需要重新渲染当前组件及子组件，可每次如果我们的状态和属性都不曾发生改变，那么重新渲染建立虚拟DOM就是一种不必要的开销，而react官方提供了一个API，让我们可以控制组件是否需要渲染。

```javascript
class ShowName extends Component {
  render() {
    return (
      <div>{this.props.name}</div>
    )
  }
}
class ShowSchool extends Component {
  render() {
    return (
      <div>{this.props.school}</div>
    )
  }
}
class Parent extends Component {
  constructor(props) {
    super(props);
    this.state = {
      name: 'hzh',
      school: 'mju',
      address: {
        prov: 'fujian'
      }
    }
    this.changeName = this._changeName.bind(this);
  }
  render() {
    return (
      <div>
         <ShowName name={this.state.name}/>
         <ShowSchool 
           school={this.state.school} 
           address={this.state.address}
         />
         <button onClick={this.changeNmae}>改变名字</button>
         <button onClick={this.changeProv}>改变省份</button>
      </div>
    )
  }
  _changeName() {
    this.setState({
      name: `hzh${(Math.random() * 100).toFixed(2)}`
    })
  }
  _changeProv() {
    let address = this.state.address;
    address.prov = 'hangzhou'
    this.setState({
      address: address
    })
  }
}
```
上面是一个粗略的例子，但是如果你环境安装好了，那是可以直接运行的。当我们改变名字的时候，因为改变了状态，所以就会重新渲染Parent和ShowName及ShowSchool,而实际上ShowSchool的属性并没有发生变化，实际上是不需要重新渲染的，对于结构简单的还好，要是存在计算量较大且结构复杂，那就能感受到界面的明显卡顿，性能差异在移动端是尤为明显的。所以既然没有属性和状态的更新，那么我们为什么要让它重新渲染呢？我们完全可以通过官方的shouldComponentUpdate阻止其更新。经过改动的ShowSchool如下:

```javascript
class ShowSchool extends Component {
  shouldComponentUpdate(nextProps, nextState) {
    return this.props.school !== nextProps.school &&
    this.props.address !== nextProps.address
  }
  render() {
    return (
      <div>{this.props.school}</div>
      <div>{this.props.address.prov}</div>
    )
  }
}
```
而当我们点击改变省份，address.prov从fujian变成了hangzhou,但是因为address引用是同一个变量，所以shouldComonentUpdate始终是false,所以ShowSchool不会更新，那么这是不合理的，因为地址已经改变了,那么现在我们来看下_changeProv的另一种实现。

```javascript
_changeProv() {
    this.setState({
      address:{
        prov: 'hangzhou'
      }
    })
  }
```
这下应为this.props.address和nextProps.address引用的是不同的变量，所以更新了，但是这时又出现了新的问题，就是后面的点击事实上address.prov都是hangzhou，并没改变，但是还是重新渲染了，所以事实上还是没有解决问题，但是如果我们要真正解决问题的话，就需要遍历整个对象做深层对比，但是一旦对象比较复杂，这个过程是相对耗时的，但是如果只做浅层比较的话，就像刚才address.prov例子一样，至于state如果是基础类型的话可以这样来判断，但是state存在引用类型的话，就不能这样判断。但是大多数的场合下，状态的复杂度并不会很大，所以，基本可以深比较两个层级的数据，其他的一些性能损失还是能够接受的。而官方也提供了es6 react-addons-shallow-compare这个工具，只做浅比较。

为了解决上述问题，Facebook出了一个不可变数据的类库immutable，使用这个类库的优势是深层比较几乎零成本，但是不好的一点是需要学习其数据API操作方法，这事实上加重了学习成本，而且这个库本身也是有一定体积的，特别是在移动端网络请求金贵的情况下，如何去平衡加载速度与性能的关系，但其本质上还是好的，可以更好维护状态，提高性能。
<div align="center">
  <img src="http://o8695iplk.bkt.clouddn.com/immutable.png" width="600" />
</div>

实际上你可以把上面这种图对应的节点树当成是state，因为平常我们组件树是否需要渲染完全是通过state来控制的，当有某个组件对应的属性或者状态改变了，并不一定需要整棵树都重新渲染的，而immutable的特点也是如此，但immutable中的某个属性发生变化，首先是产生一个新的对象，但是新对象与旧对象里不变的东西还是共用的，这是为什么immutable可以做快速深层比较的原因。下面我们通过immutable来实现上面的同样的功能。

```javascript
class Parent extends Component {
  constructor(props) {
    super(props);
    this.state = {
      name: 'hzh',
      school: 'mju',
      address: Immutable.Map({prov: 'fujian'})
    }
    this.changeName = this._changeName.bind(this);
  }
  render() {
    return (
      <div>
         <ShowName name={this.state.name}/>
         <ShowSchool 
           school={this.state.school} 
           address={this.state.address}
         />
         <button onClick={this.changeNmae}>改变名字</button>
         <button onClick={this.changeProv}>改变省份</button>
      </div>
    )
  }
  _changeName() {
    this.setState({
      name: `hzh${(Math.random() * 100).toFixed(2)}`
    })
  }
  _changeProv() {
    let adress = this.state.address;
    address = address.set('prov', 'hangzhou');
    this.setState({
      address: address
    })
  }
}
```
```javascript
class ShowSchool extends Component {
  shouldComponentUpdate(nextProps, nextState) {
    return this.props.school !== nextProps.school &&
    !Immutable.is(this.props.address, nextProps.address)
  }
  render() {
    return (
      <div>{this.props.school}</div>
      <div>{this.props.address.get('prov')}</div>
    )
  }
}
```
貌似改动的也并不大，就能实现状态更新渲染无损失，这样在性能要求较高的场合是很有必要的。

而关于react值得探索的东西还有很多，比如setState这个API的一些运行机制，如何能更好的使用setState。总的来说，react性能优化这条路还比较长,我也只是稍微触碰，并没有多深入理解，如果不对的地方可以指正。暂时没有评论，等评论搭好之后，有什么问题，可以再下方发言，共同讨论，共同进步。