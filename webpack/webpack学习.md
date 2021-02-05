### webpack学习

webpack是静态打包工具，内部构建一个**依赖图**，对应映射到项目所需的每个模块，生成一个或多个bundle

#### 基本概念

* entry：依赖图的开始
* output：在哪里输出生成的bundle
* loader：webpack只能理解js和json文件，loader将其他类型文件转换成有效模块使得webpack能够理解使用
* plugin：扩展webpack的能力
* mode：开发模式`development`、`production`

#### loader



