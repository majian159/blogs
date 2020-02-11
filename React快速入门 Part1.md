# 前言
当下是大前端时代，前端已经从简单的切图华丽转变到了工程级项目，越来越多的能力沉淀在前端项目中，其中也包含了复杂的UI业务逻辑，用一个jQuery走天下的时代已经一去不复返了。

基于业务的持续发展我们选用了React技术栈来构建公司项目的前端能力。  
而今天我们通过线上分享的方式让大家能快速上手，录屏的视频文件请查看文章尾部。

# 技术栈
起初我们在Vue和React直接进行选择，最终我们被Ant Design项目吸引，从而选用了React。  
虽然Ant Design拥有Vue实现，但是由社区贡献的，我们稳健的选择了官方支持React。  
至于Vue和React，如果开始辩论估计能大战365天，我们这边就不过多的展开了。

## TypeScript
为JS提供类型支持，企业级项目必备，不多做介绍，越来越多的JS项目转向到TS，包含且不止 Vue、antd、Angular。

## React Native
我们在APP端使用了RN相关技术，来构建RN APP和实现APP中小程序的能力.

## Ant Design项目
Ant Design 是蚂蚁金服体验技术部推出的企业级产品设计体系，并输出了基于这个体系的许多组件库和解决方案。
### antd
基于 Ant Design 设计语言，提供了开箱即用的高质量 React 和 Angular 组件实现，用于开发和服务于企业级中后台产品。

### Ant Design Pro
开箱即用的中台前端/设计解决方案

### Ant Design Mobile
一个基于 Preact / React / React Native 的 UI 组件库

### AntV
#### G2 / G2Plot 可视化引擎
G2 是一套基于图形语法理论的可视化底层引擎，以数据驱动，具有高度的易用性和扩展性。用户无需关注各种繁琐的实现细节，一条语句即可构建出各种各样的可交互的统计图表。

G2Plot 是开箱即用、易于配置、具有良好视觉和交互体验的通用统计图表库。

#### G6 / Graphin 图可视化引擎
G6 是一个简单、易用、完备的图可视化引擎，它在高定制能力的基础上，提供了一系列设计优雅、便于使用的图可视化解决方案。能帮助开发者搭建属于自己的图可视化、图分析、或图编辑器应用。

Graphin 取名意为 Graph Insight（图的分析洞察），是一个基于 G6 封装的 React 组件库，专注在关系可视分析领域，简单高效，开箱即用。

#### F2 移动端可视化方案
F2 是一个专注于移动，开箱即用的可视化解决方案，完美支持 H5 环境同时兼容多种环境（Node, 小程序，Weex），完备的图形语法理论，满足你的各种可视化需求，专业的移动设计指引为你带来最佳的移动端图表体验。

#### L7 空间数据可视分析
蚂蚁金服 AntV 数据可视化团队推出的基于 WebGL 的开源大规模地理空间数据可视分析开发框架。

### Ant Motion
使用 Ant Motion 能够快速在 React 框架中使用动画。
提供了单项，组合动画，以及整套解决方案。

### More...
以上组件足以应付大多数业务场景了，的人官方和社区还有更多丰富的组件，有兴趣的同学可以去官网查看。

# 开发环境
## NodeJS
大前端必备，可以去官网下载各个平台的安装包：https://nodejs.org/en/download/
## Yarn V1
包管理器我们选择了 Yarn V1，这边大家不要去使用最新的V2版本berry
### 设置淘宝NPM镜像
yarn config set registry https://registry.npm.taobao.org
### 常用命令
|  命令   | 描述  |
|  ----  | ----  |
| add / add -D  | 安装依赖和安装开发依赖 |
| remove  | 删除依赖 |
| start  | 启动项目 |
| build  | 构建项目 |
| upgradeInteractive  | 交互式更新依赖包 |

## VSCode
我们选用了VSCode作为IDE
### 推荐插件安装
|  插件名称   | 描述  |
|  ----  | ----  |
| Settings Sync  | 同步VSCode的插件和配置到Github Gist |
| Auto Rename Tag  | 自动重命名标签 |
| Bracket Pair Colorizer  | 为括号对提供不同的颜色标记 |
| ESLint  | ES静态语法分析 |
| TSLint  | TS静态语法分析 |
| Prettier  | 文件格式化插件 |
| React Native Tools  | RN工具 |
| TypeScript Import Sorter  | 对import进行整理和排序 |

### 推荐工作区配置
文件保存时自动格式化代码和自动整理import (删除未使用的import和import排序)  
菜单导航: “Code - 首选项 - 设置 - 工作区 - 使用json打开”
```json
{
	"editor.formatOnSave": true,
	"importSorter.generalConfiguration.sortOnBeforeSave": true,
	"importSorter.sortConfiguration.removeUnusedImports": true,
	"importSorter.sortConfiguration.removeUnusedDefaultImports": true,
	"importSorter.sortConfiguration.customOrderingRules.rules": [{
		"regex": "^(react|react-native)$",
		"orderLevel": 8
	}, {
		"type": "importMember",
		"regex": "^$",
		"orderLevel": 5,
		"disableSort": true
	}, {
		"regex": "^[^.@]",
		"orderLevel": 10
	}, {
		"regex": "^[@]",
		"orderLevel": 15
	}, {
		"regex": "^[.]",
		"orderLevel": 30
	}]
}
```
至此我们的开发环境就搭建完成了。

# 知识点脑图
## 包管理器
![PM](https://i.loli.net/2020/02/11/KAFi4eQdsSEO2Gj.png)
## ES6
![ES6](https://i.loli.net/2020/02/11/miaNfcjUF5WK8Et.png)
## TypeScript
![TypeScript](https://i.loli.net/2020/02/11/itc5CJMFSPonsw2.png)
## 常见的文件扩展名
![常见的文件扩展名](https://i.loli.net/2020/02/11/Z52osEAJycUmR7r.png)
## React
![React](https://i.loli.net/2020/02/11/rAYaMtoegNbO1XH.png)

# 我的第一个项目
## 创建
```sh
# 创建一个基于umi的项目
yarn create umi

# 包依赖还原
yarn

# 启动项目
yarn start
```
### 删除不需要的脚手架文件
脚手架中包含了一些默认文件，大家如果不需要可以自行选择删除。

## 我的第一个Page - TodoList
这一节我们会完成一个简单的任务列表，功能点有
- 添加任务
- 任务列表的展示
- 删除一个任务

完成Demo的内容后你将掌握以下内容：
- TypeScript的基本使用
- antd组件的使用
- React Hooks
  - useState
  - useEffect
  - useMemo
- 通过yarn安装、删除、和升级包
- linqts
- ......

# 《React Part 1》录屏文件
## 渠道一：优酷
https://v.youku.com/v_show/id_XNDU0MTQ4Njk3Ng==.html  
密码：jkzlReact

# 最后
今天我们从安装环境开始到第一个Demo，其中尽量降低了大家的学习门槛，许多React的高阶用法还没有说明，在下一节中我们会继续介绍Component、DvaJS、UmiJS、redux、样式表、自定义Hook、服务端交互等内容以助力大家可以使用TS+React开发企业级应用，敬请期待。