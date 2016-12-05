---
title: react-native中Image标签的使用优化
date: 2016-12-03 10:37:13
tags:
- react-native
---

## 开胃菜
接触react-native到现在，已经1年多了，项目写了不少，最近发现它的一个问题。我们都只对Image标签在引用图片时，分本地和网络两种不同的写法，分别是
```javascript
//静态图片
<Image source={require('./my-icon.png')} />
//网络图片：
<Image source={{uri: 'https://facebook.github.io/react/img/logo_og.png'}}
       style={{width: 400, height: 400}} />。
```
但处于混合app开发模式下，可以直接使用uri方式来访问图片，即：
> <Image source={{uri: 'app_icon'}} style={{width: 40, height: 40}} />
那么事情很明显了，本地图片其实也可以以uri的形式显示，接下来就来看看具体如何实现。

## 看Image资源加载的实现
找到项目根目录下node_modules/Libraries/Image/，此处是js端对Image调用做的一些处理。包括路径转换，图片属性格式转换，最后对native中封装的Image进行调用。

** 1.**第一站：Image.ios.js和Image.android.js 
Image属性暂时忽略，直接跳到render，定位到this.props.source的使用位置，如下：
```javascript
render: function() {
      //（1）
    const source = resolveAssetSource(this.props.source) || { uri: undefined, width: undefined, height: undefined };

    let sources;
    let style;
    // (2)
    if (Array.isArray(source)) {
      style = flattenStyle([styles.base, this.props.style]) || {};
      sources = source;
    } else {
      // (3)
      const {width, height, uri} = source;
      style = flattenStyle([{width, height}, styles.base, this.props.style]) || {};
      sources = [source];

      if (uri === '') {
        console.warn('source.uri should not be an empty string');
      }
    }
    
    // (4)
    const resizeMode = this.props.resizeMode || (style || {}).resizeMode || 'cover'; // Workaround for flow bug t7737108
    const tintColor = (style || {}).tintColor; // Workaround for flow bug t7737108
    
    // (5)
    if (this.props.src) {
      console.warn('The <Image> component requires a `source` property rather than `src`.');
    }
    
    return (
      // (6)
      <RCTImageView
        {...this.props}
        style={style}
        resizeMode={resizeMode}
        tintColor={tintColor}
        source={sources}
      />
    );
  },
});
```
- 对this.props.source传入的资源进行了处理，得到的返回值格式如下图。对于resolveAssetSource()函数具体做了什么一会分析
__ 从打包的资源文件中引用: __
![从打包的资源文件中引用](http://bestinfoods.oss-cn-hangzhou.aliyuncs.com/Mobile%2FreactImage%2Fsource%E6%A0%BC%E5%BC%8F.png "从打包的资源文件中引用" )
__ localhost调试模式: __
![localhost调试模式](http://bestinfoods.oss-cn-hangzhou.aliyuncs.com/Mobile%2FreactImage%2Fsource%E6%A0%BC%E5%BC%8F2.png)
- 当source不是Array时，将Image标签的style和Image.ios.js中定义的styles.base合并。
```javascript
const styles = StyleSheet.create({
  base: {
    overflow: 'hidden',
  },
});
```
如果代码中是这样调用的 ```<Image source={{uri: 'https://facebook.github.io/react/img/logo_og.png'}}
       style={{width: 400, height: 400}} />```，执行flattenStyle之后style为{width:400, height: 400, overflow: 'hidden'}。
> ** overflow作用: ** 设置图片尺寸超过容器可以设置显示或者隐藏('visible','hidden')  

##### 因为看到Array的判断，特意试了一下猜想，发现果然可以以```<Image style={{width:375, height: 250}} source={[{uri: 'http://file27.mafengwo.net/M00/BE/F4/wKgB6lSKLvSAeR6sAAqbfnNiv8k56.groupinfo.w665_500.jpeg'}]}/>```这种格式直接调用网络图片，** 注意 **只能是uri形式引进来的图片，require()进来的静态资源图片因为resolveAssetSource函数有要求是会报错的，而以http://localhost:8081/assets/Images/setting_64px_1194517_easyicon.net.png这种自定义形式（后续会讲到）引进来的静态图片则没有问题 #####

- 从source中获得宽、高和uri，并将得到的宽高与styles.base、传入的标签style属性合并成新的style。并在uri为空时给出警告。

- resizeMode的取值，优先级顺序为: 标签中定义 > style中定义 > 默认"cover".

- 如果图片标签用了src属性，例如```<Image src="http://xxxx.png" />```，给出警告。

- 将获得的所有属性值通过RCTImageView传给native，native根据属性值进行Image的绘制。
到此Image标签的初步操作讲的差不多，接下来将分析resolveAssetSource()函数做的操作。

** 2. ** 第二站 resolveAssetSource()函数的秘密
```javascript
/**
 * `source` is either a number (opaque type returned by require('./foo.png'))
 * or an `ImageSource` like { uri: '<http location || file path>' }
 */
function resolveAssetSource(source: any): ?ResolvedAssetSource {
  if (typeof source === 'object') {
    return source;
  }

  var asset = AssetRegistry.getAssetByID(source);
  if (!asset) {
    return null;
  }

  const resolver = new AssetSourceResolver(getDevServerURL(), getBundleSourcePath(), asset);
  if (_customSourceTransformer) {
    return _customSourceTransformer(resolver);
  }
  return resolver.defaultAsset();
}
```
注释中就首先声明了source的格式，这里做的主要工作是获得图片具体路径，将传入的相对路径转换成绝对路径。

## 修改Image.ios.js实现 require --> uri
为了在Image中可以统一的只使用uri，即将require本地图片用uri来完成，我写了如下的函数，将路径转换：
```javascript
  getImageURL(source) {
    if (!source || !source.uri) return source;
    
    var imageURL = source.uri;
    var scriptURL = SourceCode.scriptURL;
    scriptURL = scriptURL.substring(0, scriptURL.lastIndexOf('/') + 1)+"assets/";
 
    if (imageURL.match(/^http:\/\/.+\..+/i)) {
      return source;
    }else {
      var strArray = imageURL.split('/');
      for (let i = 0; i < strArray.length; i++) {
        if(strArray[i] === '.' || strArray[i] === "..") {continue;}
        i === strArray.length-1 ? scriptURL += strArray[i] : scriptURL += strArray[i] + '/';  
      }
      source.uri = scriptURL;
      return source;
    }
  }
```









