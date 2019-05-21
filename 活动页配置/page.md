#### 1、活动配置背景与目的
- 主要是h5活动页面的模板，交互基本是差不多的，为了解决重复开发的问题，加快活动页的生成，减少活动
页开发周期，所以做了这个系统。系统内预置多个组件，用户可通过拖拽的方式生成页面并立即发布，可操作上下架。
    
#### 2、整体方案及流程
- 项目整体采用前后端分离。前端dva+zent, node端 koa2+mysql+sequelize, 对接微服务活动模块，console-center模块。
大致流程图如下：![uml](img/uml.png)

用户通过console后台登入，点击跳转到cms，接口都先通过console-center做权限判定。
前端预置一些组件，通过拖拽的方式产生一个组件配置json，并且通过react的renderToString方法把react组件
渲染成html片段。点击暂存和上架，提交json和这个html片段。node端将json插入到数据库中，再次编辑时可以还原上次的
编辑内容；html片段用来写到一个html文件中。发布时的页面访问方式，访问yyfax的预设地址，通过nginx
代理到cms的静态活动页的位置。ajax跨域问题通过node做一次接口转发解决。默认接入了神策系统。

#### 3、项目结构
```
├── dist                     // 打包后的前端代码
├── bin                      // 编译后的server代码   
├── public                   // 静态资源
     ├─ assets                      // 资源目录
          ├─ css                        // 公共css，以及对应组件css
          ├─ images                     // 组件图片
          ├─ js                         // 组件业务js
          ├─ lib                        // 公共js
          └─ h5                         // 生成的html文件
├── doc                      // 文档 
     ├─ img                         // 文档图片
     ├─ powerdesgin                 // 表结构图
     ├─ uml                         // 时序图   
     └─ page.md                     // 文档markdown          
├── server                   // node代码  
     ├─ business                    // 业务代码，和数据库交互
     ├─ config                      // 配置
     ├─ helper                      // 辅助方法
     ├─ middleware                  // 中间件
     ├─ model                       // 表
     ├─ router                      // 接口路由
     ├─ schedule                    // 自动化处理
     ├─ service                     // 接口服务
     └─ index.ts                    // node服务入口          
├── src                      // 前端代码
     ├─ common                      // 路由和菜单的配置项
          ├─ menu.js                // 菜单配置
          └─ router.js              // 路由配置
     ├─ components                  // react公共组件库
     ├─ models                      // dva models
     ├─ pages                       // 路由对应的react业务组件
     ├─ services                    // ajax请求服务
     ├─ utils                       // 工具类文件夹
     └─ index.js                    // 前端入口文件
├── package.json
├── .gitignore
└── README.md
```

#### 4、组件写法
- 基于[zanUI](https://youzan.github.io/zent/zh/component/design)的微页面编辑组件。我们在其之上只需要
开发相应的组件即可。例如开发一个背景图组件，它分了两部分展示，一部分是预览展示，一部分是编辑配置展示。
```javascript
// imagePreview.js
import React, { PureComponent } from 'react';

const defaultStyle = {
    width: '100%'
};

export default class ImagePreview extends PureComponent {

    render() {
        const { value, styles = {} } = this.props;
        return (
            <div>
                <img src={value.imagePath} style={{...defaultStyle, ...styles}}/>
            </div>
        )
    }
}
```
```javascript
// imageEditor.js
import React from 'react';
import { DesignEditor, ControlGroup } from 'zent/lib/design/editor/DesignEditor';
import UploadFile from 'components/common/Upload';

// DesginEditor是所有的editor的基类，它提供了数据监听、修改、更新方法， 数据校验等。
// 开发基本只需要复写，render、designType、designDescription、getInitialValue以及相关的change事件。
export default class ImageEditor extends DesignEditor{
    constructor(props) {
        super(props);
        this.state = {
            imageList: {fileList: []},
            meta: {}
        };
    }

    // 相应的change事件
    handleChange = ({file, fileList, event}) => {
        if (file.status !== 'removed' && (file.type.indexOf('image') === -1 || file.size / 1024 / 1024 > 1)) {
            return;
        }
        const type = {fileList: fileList};

        if (file.response && file.status === 'done') {
            fileList[0].url = file.response.content.imgUrl;
            type.fileList = fileList;
            type['img-url'] = file.response.content.imgUrl;
            // 在成功返回的文件地址后，手动触发onCustomInputChange事件，更新相应字段            
            this.onCustomInputChange('imagePath')(file.response.content.imgUrl);
        }

        if (file.status === 'removed') {
            type['img-url'] = '';
        }
        this.setState({imageList: {...this.state.imageList, ...type}});
    };

    render() {
        return (
            <div className="rc-design-component-notice-editor">
                <ControlGroup
                    label="背景图:"
                >
                <UploadFile
                    name="imagePath"
                    options={{onChange: info => this.handleChange(info)}}
                    fileList={this.state.imageList.fileList}/>
                </ControlGroup>
            </div>
        );
    }

    static designType = 'imageBanner'; //这个是生成的组件配置的唯一标示
    static designDescription = '背景设置'; // 对应组件名
    // 初始化的数据
    static getInitialValue(settings, globalConfig) {
        return {
            imagePath: '/assets/images/imageBanner/banner.png',
            style: {},
            scrollable: false
        };
    }
}
```
```javascript
// index.js
// export出一个object
import ImagePreview from './imagePreview';
import ImageEditor from './imageEditor';
export default {
    type: ImageEditor.desginType,
    preview: ImagePreview,
    editor: ImageEditor
}
```
```javascript
// Desgin/index.js
import imageBannerConf from '../ImageBanner';

const groupedComponents = [
    Object.assign({}, configConf, {
        // 是否可以拖拽
        dragable: false,

        // 是否出现在底部的添加组件区域
        appendable: false,

        // 是否可以编辑，UMP里面有些地方config是不能编辑的
        // editable: true,

        configurable: false,

        highlightWhenSelect: false
    }),

    Design.group('基础'),
    imageBannerConf,

    Design.group('其他'),
    Object.assign({ limit: 1 }, whitespaceConf),
    Object.assign({ limit: 2 }, lineConf)
];
```
按着这个过程一个组件就写好了，更多的细节可以阅读zanUI库源码。
