### 为什么要阅读react源码
诚然，网上有很多关于react源码的解析，但大多数都是按模块进行解读，对于我这种初学者来说，实在是太教科书了，所以决定自己从一个个简单的应用开始，了解react的机制。
### 现在开始吧
从[ReactVersion.js](../../ReactVersion.js)中可以知道，我们正在阅读的react版本为16.0.0-alpha.12
### SimpleYouTubeSearch
在[这里](https://github.com/clayJa/ReactCode/tree/master/SimpleYouTubeSearch)你可以找到对应的代码。
##### index.html
你只需关注
```html
<body>
  <div class="container"></div>
</body>
<script src="/bundle.js"></script>
```
在这我们设置了一个react渲染的根节点"<div class="container"></div>",其他的都交给通过react生成的"bundle.js"
我们从/index.js开始看
```javascript
class App extends Component {
  ...
    return (
      <div>
           <SearchBar onSearchTermChange={videoSearch}/>
           <VideoDetail video={this.state.selectedVideo}  />
           <VideoList videos={this.state.videos} onVideoSelect={selectedVideo  => this.setState({selectedVideo})}/>
      </div>
    );
  }
}
// Take this component to page
ReactDOM.render(<App/>, document.querySelector('.container'));
```
在这里我们定义了一个继承于Component的类组件<App />，这个类返回一段html代码，然后，我们用ReactDOM类的render函数将组件<App />和通过document.querySelector()函数获取/index.html的Document对象作为参数，目的是为了将组件渲染到/index.html。
*值得注意的是：我们应该将组件的定义放在components文件夹下，所以严格意义来说，这里的做法是不规范的。*
#### 接下来我们来看看react替我们完成了怎样的操作
首先找到*ReactDOM* ,在[/renderers/dom/ReactDOM.js](../../renderers/dom/ReactDOM.js#L32),我们看到[render](../../renderers/dom/ReactDOM.js#L34)属于[ReactMount](../../renderers/dom/stack/client/ReactMount.js#L307)的[render](../../renderers/dom/stack/client//ReactMount.js#L596)方法
ReactMount:是通过创建组件内的DOM元素，并把元素插入到提供的*container*元素中（container元素之前的内容将被替换），来实现初始化React组件的过程。
render：从代码可以看出render实际对传入的* @param {ReactElement} nextElement*通过[_renderSubtreeIntoContainer](../../renderers/dom/stack/client//ReactMount.js#L450)一层层遍历，然后添加到*container*元素上
```javascript
/**
* Renders a React component into the DOM in the supplied `container`.
* See https://facebook.github.io/react/docs/react-dom.html#render
*
* If the React component was previously rendered into `container`, this will
* perform an update on it and only mutate the DOM as necessary to reflect the
* latest React component.
*
* @param {ReactElement} nextElement Component element to render.
* @param {DOMElement} container DOM element to render into.
* @param {?function} callback function triggered on completion
* @return {ReactComponent} Component instance rendered in `container`.
*/
render: function(nextElement, container, callback) {
  return ReactMount._renderSubtreeIntoContainer(
   null,
   nextElement,
   container,
   callback,
  );
}
```
#### *_renderSubtreeIntoContainer*
我们先标记下代码的[开始](../../renderers/dom/stack/client//ReactMount.js#L450)，[结束](../../renderers/dom/stack/client//ReactMount.js#L581)
 ```javascript
 _renderSubtreeIntoContainer: function(
  parentComponent,
  nextElement,
  container,
  callback,
){
     /*...*/
     //The most important code
     var nextWrappedElement = React.createElement(TopLevelWrapper, {
      child: nextElement,
    });
     //getContextForSubtree:获取子树的上下文
    var nextContext = getContextForSubtree(parentComponent);
    //获得最外层包裹的容器
    var prevComponent = getTopLevelWrapperInContainer(container);
     //这个逻辑可以确保在没有任何支持实例的情况下对无状态树进行操作。
    if (prevComponent) {
      var prevWrappedElement = prevComponent._currentElement;
      var prevElement = prevWrappedElement.props.child;
      //如果应该更新组件，则调用ReactMount._updateRootComponent进行更新
      if (shouldUpdateReactComponent(prevElement, nextElement)) {
        var publicInst = prevComponent._renderedComponent.getPublicInstance();
        var updatedCallback =
          callback &&
          function() {
            validateCallback(callback);
            callback.call(publicInst);
          };
        ReactMount._updateRootComponent(
          prevComponent,
          nextWrappedElement,
          nextContext,
          container,
          updatedCallback,
        );
        //返回当前实例
        return publicInst;
      } else {
          //销毁container中的组件
        ReactMount.unmountComponentAtNode(container);
      }
    }
     // 是否应该渲染洗新组件
    var reactRootElement = getReactRootElementInContainer(container);
    var containerHasReactMarkup =
      reactRootElement && !!internalGetID(reactRootElement);
    var containerHasNonRootReactChild = hasNonRootReactChild(container);

    /*...*/

    var shouldReuseMarkup =
      containerHasReactMarkup &&
      !prevComponent &&
      !containerHasNonRootReactChild;
     //在_renderNewRootComponent中，初始渲染是同步的，但是发生的任何更新在componentWillMount或componentDidMount中的渲染将根据目前的策略被批处理。
    var component = ReactMount._renderNewRootComponent(
      nextWrappedElement,
      container,
      shouldReuseMarkup,
      nextContext,
      callback,
    )._renderedComponent.getPublicInstance();
    return component;
}
```
#### *React.createElement*
在_renderSubtreeIntoContainer中我们发现最重要的当属React.createElement，接下来我们来看看React.creactElement是如何运作的。
我们找到[React.js](../../isomorphic/React.js)中React.[createElement](../../isomorphic/React.js#L56)的[源代码](../../isomorphic/classic/element/ReactElement.js#L183)
```javascript
ReactElement.createElement = function(type, config, children) {
     //...从config提取一些参数如key，ref，self，source，props
     //...我们可以传入两个以上参数，ReactElement.createElement将它添加到props.children
     return ReactElement(
       type,
       key,
       ref,
       self,
       source,
       ReactCurrentOwner.current,
       props,
     );
     //最后return 出来一个 ReactElement
}

var ReactElement = function(type, key, ref, self, source, owner, props) {
  var element = {
    // This tag allow us to uniquely identify this as a React Element
    $$typeof: REACT_ELEMENT_TYPE,、
    // Built-in properties that belong on the element
    type: type,
    key: key,
    ref: ref,
    props: props,
    // Record the component responsible for creating this element.
    _owner: owner,
  };
  return element;
};
//到这里我们就已经清楚组件渲染过程了

//对照_renderSubtreeIntoContainer中相关参数
//TopLevelWrapper：store all top-level pending updates on composites
/*var nextWrappedElement = React.createElement(TopLevelWrapper, {
child: nextElement,
});*/

```
#### 稍微小结一下render
ReactDOM.render(<App/>, document.querySelector('.container'));通过不断调用  _renderSubtreeIntoContainer方法,把<App />中一层层剥开通过ReactElemen处理返回的element，最终由_renderNewRootComponent渲染到DOM。
