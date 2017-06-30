---
title: 搭建一个react-native高德地图
date: 2017-05-27 18:24
categories: ["技术", "前端", "react-native"]
tags: ["native", "react-native"]
---
&emsp;&emsp;最近在用`react-native`写一个混合原生的应用，暂时是这么打算点，先把一些简单部分的先通过`react-native`来替换掉，后面等替换部分稳定之后，再进行比较核心功能的替换。这样也可以有一个比较平稳的过程，毕竟`react-native`还不是很稳定，所以打算以一种比较平稳的方式过渡。废话不多说了，接下来就让我们开始来动手来做一个高德地图，所涉及到的只是部分native Api。但也不用担心，因为大部分暴露api的方式都类似，重复讲也只是浪费大家的时间。

&emsp;&emsp;我想有地图开发经验的都知道，我们需要在地图平台上申请`key`来调用地图的`SDK`，高德地图也不例外。所以首先我们需要在阿里开发者平台上申请应用的`key`。至于怎么申请我就不多说，官方文档写得很清楚。接下来我们就通过下面命令来新建一个`react-native`项目:

```shell
  react-native init AmapTest
```
安装完之后可以先跑起来，万一程序在这里本身就出问题了，接下来的更多操作让你更难定位出问题在哪。

```shell
  react-native run-ios
```
当一切都没问题了，那么下面会涉及到native开发，所以没native半点基础的我建议先去大概了解下native开发的基础知识。下面我需要安装高德地图的SDK，这有几种方式，可以手动，但为了简单直接我们用`CocoaPods`IOS开发依赖工具，其实就像`npm`、`yarn`一样，我们需要写要安装的依赖库和相应的版本，就像`package.json`依赖配置文件一样。在刚才新建项目的ios目录下新建一个Podfile配置文件，内容如下：
```shell
platform :ios, '8.0'

target 'AmapTest' do
  pod 'AMap2DMap' #2D地图SDK,
  pod 'AMapSearch' #地图搜索
  pod 'AMapLocation' #定位SDK
end

```
至于需要引入哪些库看你的项目需求。那么接下来的命令也跟`npm install`类似：
```shell
pod install
```
到这里依赖库就都安装好了，那么我们可以打开项目来进行编码了。慢着，这里有个很关键的项目是通过那个文件打开，如果没有引入其他iOS项目，那么双击`ios/AmapTest.xcodeproj`就可以了。因为我们引入三方项目，这是会在ios目录下生成`AmapTest.xcworkspace`，这是一个工作空间，可以容纳多个项目，这时我们双击这个文件就可以，这样就包含我们本身的项目和高德地图的项目。

其实我感觉环境搭建还是相当麻烦，特别是对于新手来说，被各种配置搞得晕头转向的，但这都是一个过程，慢慢也就知道这些配置的意思了。

那就让我们开始写代码吧。首先我们需要注册一下前面注册好得到的高德地图的key
```Object-C
#import <AMapFoundationKit/AMapFoundationKit.h>
[AMapServices sharedServices].apiKey = @"你申请的key";

```
接下来需要创建一个地图方法的模块，一般我们需要创建这样的两个文件，我这边的暴露模块名字为Amap，所以需要创建两个文件，一个是Amap.m和AmapManger.m，但是为了看起来比较直观我们就创建一个AmapManger.m就可以，但这并不是最好的写法。最好是将视图相应的管理放在Amap.m这件，然后AmapManger.m主要是管理需要暴露给js调用的方法。这样就算不是用作react-native，这个模块也可以重用。但这边我为了直观就都写到AmapManger.m，真实的项目不建议这么写。那么接下来我们来看看AmapManger.m的代码。
```
#import <MAMapKit/MAMapKit.h>
#import <AMapFoundationKit/AMapFoundationKit.h>

#import <React/RCTViewManager.h>
#import <React/RCTBridge.h>
#import <React/RCTUIManager.h>

@interface RNTMapManager : RCTViewManager
@end

@implementation RNTMapManager
//暴露模块给react-native
RCT_EXPORT_MODULE(Amap)

//返回一个视图
- (UIView *)view
{
  MAMapView *_mapView = [[MAMapView alloc] init];
  _mapView.delegate = self; //设置代理
  return _mapView;
}


//暴露一个操作地图的方法
RCT_EXPORT_METHOD(addPoint:(nonnull NSNumber *)reactTag
                  args:(NSDictionary *)args)
{
  [self.bridge.uiManager addUIBlock:
   ^(__unused RCTUIManager *uiManager, NSDictionary<NSNumber *, MAMapView *> *viewRegistry) {
     //数据转换
     //第一个点
     double lat = [[RCTConvert NSNumber:args[@"lat"]] doubleValue];
     double lon = [[RCTConvert NSNumber:args[@"lon"]] doubleValue];
     
     NSLog(@"long %f lat %f", lon, lat);
     //获取注册的视图对象，在js端是通过ref来引用的
     //第二个点
     MAMapView *view = viewRegistry[reactTag];
     if (!view || ![view isKindOfClass:[MAMapView class]]) {
       RCTLogError(@"Cannot find MAMapView with tag #%@", reactTag);
       return;
     }
     //绘制大头针标注
     MAPointAnnotation *pointAnnotation = [[MAPointAnnotation alloc] init];
     pointAnnotation.coordinate = CLLocationCoordinate2DMake(lat, lon);
     pointAnnotation.title = @"方恒国际";
     pointAnnotation.subtitle = @"阜通东大街6号";
     
     [view addAnnotation:pointAnnotation];
   }];
}

//绘制大头针标注对应的配置，通过代理的方式调用
-(MAAnnotationView *)mapView:(MAMapView *)mapView viewForAnnotation:(id <MAAnnotation>)annotation
{
  if ([annotation isKindOfClass:[MAPointAnnotation class]])
  {
    static NSString *pointReuseIndentifier = @"pointReuseIndentifier";
    MAPinAnnotationView*annotationView = (MAPinAnnotationView*)[mapView dequeueReusableAnnotationViewWithIdentifier:pointReuseIndentifier];
    if (annotationView == nil)
    {
      annotationView = [[MAPinAnnotationView alloc] initWithAnnotation:annotation reuseIdentifier:pointReuseIndentifier];
    }
    annotationView.canShowCallout= YES;       //设置气泡可以弹出，默认为NO
    annotationView.animatesDrop = YES;        //设置标注动画显示，默认为NO
    annotationView.draggable = YES;        //设置标注可以拖动，默认为NO
    annotationView.pinColor = MAPinAnnotationColorPurple;
    return annotationView;
  }
  return nil;
}

@end
```

这边我重点解释2个点：

- 就是js调用到原生对象需要进行转换，通过调用RCTConvert来进行转换
- 就是在uiManager里维护的视图列表获取对应要操作的视图

到现在原生部分的代码已经完成，接下来就是在js端的调用。我们需要有这么有一个组件来对原生组件暴露的方法进行封装，让写业务的同学感受不到native端的东西。那么我们创建一个Amap组件：

```
//Amap component
import { requireNativeComponent, View, NativeModules, findNodeHandle } from 'react-native';
import React from 'react'
import PropTypes from 'prop-types';

class Amap extends React.Component {
  render() {
    return (
      <RNAmap 
        ref={(ref) => this.ref = ref}
        {...this.props}
      />
    )
  }

  addPoint(obj) {
    ／*原生方法的调用 
      findNodeHandle(this.ref) 这是获取维护在uiManager组件列表的视图引用
    *／
    
    NativeModules.Amap.addPoint(findNodeHandle(this.ref), obj);
  }
}
Amap.propTypes = {
  ...View.propTypes
};
const RNAmap = requireNativeComponent('Amap', Amap);

module.exports = Amap;

```

到这里差不多了，接下来就是在业务中调用这个组件了

```
import React, { Component } from 'react';
import {
  AppRegistry,
  StyleSheet,
  Text,
  View
} from 'react-native';

import Amap from './Amap'

export default class AmapTest extends Component {

  componentDidMount() {
    this.map.addPoint({
      lon: 119.986721,
      lat: 30.263183
    });
  }

  render() {
    return (
      <View style={styles.container}>
        <Amap ref={_ref => this.map = _ref} style={{flex: 1}}/>
      </View>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    // justifyContent: 'center',
    // alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  welcome: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  },
  instructions: {
    textAlign: 'center',
    color: '#333333',
    marginBottom: 5,
  },
});

AppRegistry.registerComponent('AmapTest', () => AmapTest);

```

到这里代码就已经完成，在xcode里CMD+R来运行项目，效果如下：

![](http://oofjmuxr0.bkt.clouddn.com/map_capture.png)

&emsp;&emsp;看到运行成功是不是挺开心的，可能过程当中会遇到各种个样的问题，因为不管是iOS开发者还是前端开发，对彼此的端都不是很熟悉，但要始终保持一种态度，遇到问题就解决问题，不要试图逃避。比如可以查询官网文档，兴许文档并不是解释的非常清楚。那到哪个点遇到问题就去谷歌百度。如果这样还找不到解决的办法，可以尝试去看看源码，主要有足够耐心，我想最终都可以解决的。

&emsp;&emsp;还有就是我们就暴露了一个接口，同时我们也可以暴露更多的方法，需要用到什么暴露什么，渐渐的就包含了业务需要用到的所有接口，那这时候这个组件也就趋于稳定。以后有需要的业务直接引入便可。










