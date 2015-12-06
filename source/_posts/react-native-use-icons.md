title: react-native-use-icons
date: 2015-12-07 06:21:51
tags:
---

在React-Native里面使用icons, 我们使用的是一个流行的库叫[react-native-icons](https://github.com/corymsmith/react-native-icons)

安装方法:
1. `npm install react-native-icons@latest --save`
2. In XCode, in the project navigator right click Libraries ➜ Add Files to [your project's name]
3. Go to node_modules ➜ react-native-icons➜ ios and add ReactNativeIcons.xcodeproj
4. Add libReactNativeIcons.a (from 'Products' under ReactNativeIcons.xcodeproj) to your project's Build Phases ➜ 
    `Link Binary With Libraries` phase
5. Add the font files you want to use into the `Copy Bundle Resources` build phase of your project 
    (click the '+' and click 'Add Other...' then choose the font files from 
    `node_modules/react-native-icons/ios/ReactNativeIcons/Libraries/FontAwesomeKit`)
6. Run your project (`Cmd+R`)

使用方法:
name, size, style 里面一个都不能少

```
<Icon name='ion|ios-timer-outline' size={150} style={{width: 150, height: 150}} />
```

参考链接:
[https://github.com/corymsmith/react-native-icons](https://github.com/corymsmith/react-native-icons)


