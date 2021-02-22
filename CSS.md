### CSS

#### 块级元素和内联元素

| 类型 | 包含                | 特性                                                         |
| ---- | ------------------- | ------------------------------------------------------------ |
| 块级 | div、p、ul、li      | 独占一行，可设置宽高                                         |
| 内联 | span、a、img、input | 可与文字在一行显示，不可设置宽高，垂直方向上的padding、margin失效 |

#### display属性

每个元素可以理解为有两个盒子：

* 外在盒子：负责元素流布局
* 内在（容器）盒子：负责元素宽高、内容展示等

因此display:inline-block就可以理解为外在盒子为inline，容器盒子为block，即表现为内联元素，又可以设置宽高等属性

#### 盒子模型

包含标准盒子和IE盒子

区别在于其宽高的范围，标准的只包含content的范围，IE包含content+padding+border的范围

在css3中box-sizing属性可定义：border-box,padding-box,content-box

#### vertical-align

vertical-align值作用于display: inline,inline-block,inline-table,table-cell

* 一个inline-block元素，如果里面没有内联元素或overflow:visible，则该元素的基线为margin底部，否则为其内联元素最后一行的基线

![image-20210222103348534](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210222103348534.png)

* 块级元素中的图片总是有一条间隙，该间隙产生的原因就是“**幽灵空白节点**”、line-height、vertical-align

  ![image-20210222124030262](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210222124030262.png)

#### float浮动

特性：

* **块状化**并格式化上下文：display表现为block
* 破坏文档流，**父级高度塌陷**

#### 清除浮动的方法

1. clear:both；**clear只能作用于块级元素，本质是让自己与浮动元素不在一行显示**

   给浮动元素的父级添加一个:after伪元素添加clear:both，

   ```js
   .clear:after{
       content:'';
       display:block;
       clear:both;
   }
   ```

2. 为父元素添加**overflow：hidden**

3. 

#### BFC块级格式化上下文

特性：

* 一个独立的渲染空间，内部子元素不会影响外部元素，外部元素也不能影响内部子元素
* 不会发生margin重叠

触发条件：

* 根元素
* float不为none
* position:fixed、absolute
* display:inline-block、table-cell、table-caption、flex
* overflow不为visible

作用：

* 布局
* 避免margin重叠
* 清除浮动overflow：hidden

#### position

1. absolute：绝对定位

   * 脱离文档流，不占据空间

   * 位置相对于最近的已定位（不为static）的祖先元素
   * 对于无依赖绝对定位（无top等位置属性），如果其祖先元素全为非定位元素，其位置还是当前位置而不是浏览器左上方

2. relative：相对定位

   * 位置相对于它本身
   * 无论是否移动，元素仍然是占据原来的空间

3. fixed：固定定位

   * 脱离文档流，不占据空间
   * 位置相对于浏览器窗口固定

#### 层叠上下文

如果一个元素含有层叠上下文，(也就是说它是层叠上下文元素)，我们可以理解为这个元素在`Z轴`上就“高人一等”，最终表现就是它离屏幕观察者更近。

触发条件：

* 根元素
* 普通元素设置`position`属性为**非**`static`值并设置`z-index`属性为具体数值，产生层叠上下文

#### 层叠水平即顺序

层叠水平决定同一个层叠上下文中元素在z轴的显示顺序，所有元素都有层叠水平

层叠顺序：

![image-20210222153512529](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210222153512529.png)

#### z-index

z-index**只能在定位元素上奏效**，该属性设置一个定位元素沿z轴的位置

#### 选择器优先级

![img](https://img2020.cnblogs.com/blog/1864877/202004/1864877-20200408234042787-674324294.png)

#### 隐藏页面元素的方法及区别

1. display：none；不在页面中占据空间
2. visibility：hidden；元素隐藏，在页面中占据空间，即不改变页面布局，但不触发事件
3. opacity：0；透明度为0，不改变页面布局，可以触发事件

#### flex布局

![img](http://www.ruanyifeng.com/blogimg/asset/2015/bg2015071004.png)

容器属性：

* flex-direction：主轴方向
* flex-wrap：轴线排不下时如何换行
* justify-content：item在主轴上的对齐方式
* align-items：item在交叉轴的对齐方式

item属性：

* order：排列顺序
* flex-grow：放大比例
* flex-shrink：缩小比例，0为不缩放
* align-self：

#### 三栏布局

* 双飞翼布局

  ```js
  <div class="header">头</div>
  <div class="middle">
      <div class="content">主</div>
  </div>
  <div class="left">左</div>
  <div class="right">右</div>
  <div class="footer">底</div>
  
  <style type="text/css">
      .middle{
          float: left;
          width: 100%;
          height: 100%;
      }
      .content{
          margin: 0 100px;
      }
      .left{
          float: left;
          width: 100px;
          background:#0c9;
          margin-left: -100%;
      }
      .right{
          float: left;
          width: 100px;
          background:#0c9;
          margin-left: -100px;
      }
      .footer{
          clear: both;
      }
  </style>
  ```

* 圣杯布局(position:relative)

  ```js
  <div class="header">头</div>
  <div class="main">
      <div class="middle">主</div>
      <div class="left">左</div>
      <div class="right">右</div>
  </div>
  <div class="footer">底</div>
  
  <style type="text/css">
      .main{
          padding: 0 100px;
      }
      .middle{
          float: left;
          width: 100%;
          height: 100%;
      }
      .left{
          float: left;
          width: 100px;
          background:#0c9;
          margin-left: -100%;
          position: relative;
          left: -100px;
      }
      .right{
          float: left;
          width: 100px;
          background:#0c9;
          margin-left: -100px;
          position: relative;
          right: -100px;
      }
      .footer{
          clear: both;
      }
  </style>
  ```

* flex布局

  ```js
  <div class="header">头</div>
  <div class="main">
      <div class="left">左</div>
      <div class="middle">主</div>
      <div class="right">右</div>
  </div>
  <div class="footer">底</div>
  
  <style type="text/css">
      .main{
          display: flex;
      }
      .middle{
          float: left;
          width: 100%;
          height: 100%;
      }
      .left{
          float: left;
          width: 100px;
          background:#0c9;
          flex-shrink: 0;
      }
      .right{
          float: left;
          width: 100px;
          background:#0c9;
          flex-shrink: 0;
      }
  </style>
  ```

  

#### 画一条0.5px的线



#### 单行文字溢出省略效果

```js
.ell{
    text-overflow:ellipsis;
    white-space:nowrap;
    overflow:hidden;
}
```

#### 垂直水平居中

1. 内联元素

   水平居中

   * text-align:center
   * flex布局：display:flex;justify-content:center;

   垂直居中

   * 单行：height===line-height
   * 多行：display:table-cell;vertical-align:middle;

