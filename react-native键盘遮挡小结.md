---
title: react-native键盘遮挡小结
date: 2017-07-28 17:33
categories: ["技术", "前端", "react-native"]
tags: ["native", "react-native"]
---

又是好久没更文了，自己都觉得有点慌了。刚好今天解决react-native键盘遮挡问题，所以在这里总结一下。

是不是经常会遇到一些文本输入的时候被键盘遮挡很不爽，那我们来讲讲如何来实现不遮挡功能。首先我们来看下面这张图，

![](http://oofjmuxr0.bkt.clouddn.com/blog_rn_keyboard.png)

遮挡基本上就是这种情况，但是键盘的大小和位置是固定的，我们没办法去更改这部分的东西。因此，我们能想到的就是把我们自己的内容往上挪动。就在键盘出现的时候让文本框网上挪动，键盘消失回到原来的位置。貌似很简单，其实也确实不难，那具体要怎么做呢？其实这里面很关键的一个点是我们需要挪动多少距离来保证键盘不遮挡，如果这个解决了，那就完成了大部分了。下面我们再看张图

![](http://oofjmuxr0.bkt.clouddn.com/blog_rn_keyboard2.png)

现在应该知道如何下手了，那接下来我们就开始写逻辑。

```javascript
import React, { Component } from 'react'
import {
  View,
  Text,
  Image,
  TextInput,
  StyleSheet,
  Keyboard,
  Animated,
  TouchableOpacity,
  findNodeHandle,
  Dimensions
} from 'react-native'
var ReactNativeComponentTree = require('ReactNativeComponentTree');

export default class Login extends Component {

  constructor(props) {
    super(props);
    this.keyboardHeight = new Animated.Value(0);
  }
  
  componentWillMount () {
    this.keyboardWillShowSub = Keyboard.addListener('keyboardWillShow', this.keyboardWillShow);
    this.keyboardWillHideSub = Keyboard.addListener('keyboardWillHide', this.keyboardWillHide);
  }

  componentWillUnmount() {
    Keyboard.removeListener('keyboardWillShow', this.keyboardWillShow);
    Keyboard.removeListener('keyboardWillHide', this.keyboardWillHide);
  }

  keyboardWillShow = (event) => {
    
    this.input.measure((fx, fy, width, height, px, py) => {
      //fx fy 是相对于父元素的坐标
      //width height 元素自身的宽高
      //px py 元素相对于页面的坐标

      //py + height 文本框底部在页面中的距离 如果大于键盘的在页面中距离 那就要移动
      if (height + py > event.endCoordinates.screenY) {
        Animated.timing(this.keyboardHeight, {
          duration: event.duration,
          toValue: -(height + py - event.endCoordinates.screenY + 30),
        }).start()
      }
    })
  };

  keyboardWillHide = (event) => {
      Animated.timing(this.keyboardHeight, {
        duration: event.duration,
        toValue: 0,
      }).start();
  };
  

  render() {
   
   return <Animated.View style={[styles.container, {transform: [{translateY: this.keyboardHeight}]}]}>
      <View style={styles.header}>
        <Image source={require('../../images/logo.png')} />
      </View>
      <View style={styles.formContainer} ref={(ref) => this.container = ref}>
        <View style={[styles.inputWrap, {marginBottom: 20}]}>
          <View style={styles.imageWrap}>
            <Image source={require('../../images/phone_number.png')} />
          </View>
          <TextInput  onFocus={this.onFocus} onLayout={this.onLayout} placeholder="输入手机号" placeholderTextColor="#999"  style={styles.input}/>
        </View>
        <View style={styles.inputWrap}>
          <View style={styles.imageWrap}>
            <Image source={require('../../images/verification_code.png')} />
          </View>
          <TextInput onFocus={this.onFocus} placeholder="输入验证码" placeholderTextColor="#999" style={styles.input}/>
          <TouchableOpacity style={styles.codeBtn}>
            <Text style={styles.codeBtnText}>获取验证码</Text>
          </TouchableOpacity>
        </View>
      </View>
      <TouchableOpacity style={styles.loginBtn} onPress={this.login}><Text style={styles.loginBtnText}>登录</Text></TouchableOpacity>
      <View style={styles.userTipWrap}>
        <Text style={{color: '#999'}}>登录后即表示同意</Text>
        <TouchableOpacity onPress={this.lookAgreement}>
          <Text style={{color: '#00A8EA'}}>《登录后即表示同意》</Text>
        </TouchableOpacity>
      </View>
    </Animated.View>
  }
  onFocus = (e) => {
    //获取当前是激活的是哪个文本框
    this.input = ReactNativeComponentTree.getInstanceFromNode(e.nativeEvent.target);
  }
}

```

看完代码是不是觉得很简单，但是毕竟自己动手去做了，才能发现问题。