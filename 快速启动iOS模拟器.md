---
title: 快速启动iOS模拟器
date: 2017-12-30 11:55
categories: ["技术", "模拟器"]
tags: ["移动端", "iOS"]
---

&emsp;&emsp;以前学习iOS时，启动项目总是在xcode中启动的，因为xcode毕竟是iOS标配，方式倒没啥毛病。后面用react-native来开发APP，启动的时候也会自动选择一个模拟器挺方便。后面谷歌出了一个Flutter也是用于开发跨平台的APP，决定试试，看看开发效率和性能跟其他解决方案有什么差距，当然这篇文章的重点不是这个。启动一个Flutter项目，他会侦测当前哪个模拟器处于激活状态，然后安装APP到那个模拟器。当然你打开xcode启动模拟器也没啥没病，就是觉得有点别扭。但是作为程序员，肯定会嫌太麻烦，觉得应该有命令行，你搜索还真有的。咦！问题解决了。哈哈，貌似还没有。

启动模拟器命令是这样的：
```shell
xcrun simctl boot 设备ID
```
设备ID ? 这又是什么鬼？该怎么拿到呢？继续查，命令是这样的。

```shell
xcrun simctl list devices
```
列出了可用和不可用的设备：
<img src="http://oofjmuxr0.bkt.clouddn.com/deviceList.png" width="600" />

稍微看了一下，也比较清晰，设备名，设备ID, 设备状态

<img src="http://oofjmuxr0.bkt.clouddn.com/deviceDetail.png" width="600" />

那事情就简单啦，不就是复制一下设备ID，然后通过上面提到的命令启动模拟器就可以啦。事情解决了。好像什么不对劲，等等，让我缓缓，被你带偏了。事实上到了这里就是我们要达到的功能，但是对不起我们的题目啊，如果只止于此，那就是忽悠观众。

&emsp;&emsp;让我们继续的原因有几个：

- 我们不可能让每个人都记住这两个命令
- 每次我们需要启动一个模拟器都要查一下它的ID

这我们如何能忍，能不能提供一种大家比较容易接受的方式呢？当然了，那就写个命令行来简化流程，这不正是程序员喜欢干的事情嘛。

## 获取设备列表

```javascript
const getDeviceList = (callback) => {
  exec('xcrun simctl list devices', (err, stdout, stderr) => {
    let devices = [];
    if (err) {
      return callback(err, []);
    }
    try {
      let str = `${stdout}`
  
      let lines = str.split('\n');
      let deviceTypes = [];
      //解析设备类型（iOS，watch ...）
      const typeReg = /^\-\- ([\w\.\s*:\-]+) \-\-$/ig;
      lines.forEach((line, index) => {
        let match = typeReg.exec(line);
        if (match) {
          deviceTypes.push({index: index, name: match[1]})
        }
      })
      deviceTypes.map((type, index) => {
        if (type.name.indexOf('iOS ') > -1) {
          if (index < deviceTypes.length - 1) {
            devices = lines.slice(type.index + 1, deviceTypes[index + 1].index);
          }
          if (index === deviceTypes.length - 1) {
            devices = lines.slice(type.index + 1);
          }
        }
      })
    
      //解析设备名， 设备id, 设备状态
      const nameReg = /\s{4}([\w\s]+)\s\(/;
      const deviceIdReg = /\(([\w\d\-]+)\)/;
      const statusReg = /\((Booted|Shutdown)\)/;
      
      devices = devices.map(device => {
        
        let name = nameReg.exec(device)[1];
        let deviceId = deviceIdReg.exec(device)[1];
        let status = statusReg.exec(device)[1];
        return {name, deviceId, status};
      })
      callback(null, devices);
    } catch(err) {
      callback(err, devices);
    }    
  })
}
```

## 操作设备

```javascript
const opDevice = (type) => {
  
  getDeviceList((err, devices) => {
    if (err) {
      return console.log(chalk.red(err));
    }
    let deviceNames = devices.map(device => device.name);
    inquirer.prompt([
      {
        type: 'list',
        name: 'devicename',
        message: '请选择一种设备',
        choices: deviceNames
      }
    ]).then(answers => {
      let device = devices.filter(item => {
        return item.name === answers.devicename
      })

      const { deviceId, status } = device.length > 0 ? device[0] : {}

      if (!deviceId) {
        return console.log(chalk.red('数据问题，请联系'));
      }

      if (status === 'Booted' && type === 'launch') {
        return console.log(chalk.red('该模拟器已经启动'));
      }

      if (status === 'Shutdown' && type === 'shutdown') {
        return console.log(chalk.red('该模拟器已经停止'));
      }

      const bootCmdStr = `
        xcrun simctl boot ${device[0].deviceId};
        open -n /Applications/Xcode.app/Contents/Developer/Applications/Simulator.app --args -currentDeviceUDID ${device[0].deviceId}
      `
      const shutdownCmdStr = `
        xcrun simctl shutdown ${device[0].deviceId}
      `
      const cmd = type == 'launch' ? bootCmdStr : shutdownCmdStr;
      exec(`${cmd}`, (err, stdout, stderr) => {
        if (!err) {
          console.log(chalk.cyan(`${type} success!!!!`));
          return;
        } 
        console.log(chalk.red(`${type} fail!!!!${err}`));
      })
    })
  })
}
```
这两个函数就是刚才的那两个命令，然后把他们做成命令行

## 命令

```javascript
#!/usr/bin/env node

yargs.usage('$0 <cmd> [args]')
  .command('launch', '启动模拟器', (args) => {
  }, function(argv) {
    console.log(chalk.cyan('launch---------'));
    opDevice('launch')
  })
  .command('shutdown', '停止模拟器', (args) => {
  }, function(argv) {
    console.log(chalk.cyan('shutdown ---------'));
    opDevice('shutdown')
  })
  .help()
  .argv

```

到这里就基本可以实现了，当然这其实只是做了最简单的启动跟停止，其他`xcrun simctl`功能很强大，可以自行研究，然后把自己的命令行做大做强。如果满足启动跟停止模拟器可以直接安装。安装步骤如下：

首先需要一个node环境，具体这个怎么装就不说了。

## 安装
```shell
npm i lanchsimulator -g
```

## 启动模拟器 
```shell
ios-simulate launch
```

## 停止模拟器
```shell
ios-simulate shutdown
```

这个也只是一个简化版本，还有很多地方需要优化，但是我们主要是探讨一些解决方法，至于更具体的实现因人而异。
