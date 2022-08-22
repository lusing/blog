# TensorFlow.js机器学习教程(2) - js味儿的张量操作

既然我们使用TensorFlow.js来写机器学习代码而不是用Python版的TensorFlow和PyTorch，我们还是希望让代码充满js本身的味道。

## 复习：js数组是动态的

![](https://gw.alicdn.com/imgextra/i2/O1CN0152pERw1VE0OcVc3AJ_!!6000000002620-0-tps-750-422.jpg)

为了不被后面充满了静态语言特色的TensorFlow.js API带偏了，我们首先复习下js的数组操作。

首先我们不要忘了，js是一门动态语言，js的数组是动态数组，没有定长数组越界这一说法的。

比如说我们要给一个空数组的第2个元素赋值，这是没有任何问题的：
```javascript
let a1 = [];
a1[2] = 3;
console.log(a1);
```
输出结果为：
```
[ <2 empty items>, 3 ]
```

我们可以毫无压力地用这样的数组去生成张量：
```js
let a1_t = tf.tensor1d(a1);
a1_t.print();
```

tf.js会给我们甩出两个NaN出来：
```
Tensor
    [NaN, NaN, 3]
```

不但是空数组随便添加元素，我们用new Array生成一个长度的数组后，仍然可以说话不算话，随意给赋值。比如我们new 5个元素的Array，给第9个赋值：
```js
let a2 = new Array(5);
a2[9] = 10;
console.log(a2);


let a2_t = tf.tensor1d(a2);
a2_t.print();
```

tf.js照例给我们补9个NaN出来：
```
[ <9 empty items>, 10 ]
Tensor
    [NaN, NaN, NaN, NaN, NaN, NaN, NaN, NaN, NaN, 10]
```

如果懒得数一共多少个元素，就想在数组的末尾添加新元素，可以使用push方法，参数个数不限，push几个元素都可以：
```js
let a3 = new Array();
a3.push(1,2,3);
a3.push(4,5);

let a3_t = tf.tensor1d(a3);
a3_t.print();
```

输出为：
```
Tensor
    [1, 2, 3, 4, 5]
```

如果想从头添加新元素，可以使用unshift方法：
```js
let a3 = new Array();
a3.push(1,2,3);
a3.push(4,5);
a3.unshift(6);

let a3_t = tf.tensor1d(a3);
a3_t.print();
```

输出为：
```
Tensor
    [6, 1, 2, 3, 4, 5]
```

同时我们复习一下，与push相对的，删除最后一个元素的是pop方法；而与unshift相对的是shift方法。

比如我们对上面的a3进行pop：
```javascript
let a4 = a3;
let a00 = a3.pop();
console.log(a00);
console.log(a4);
```

所得结果为：
```
5
[ 6, 1, 2, 3, 4 ]
```

最后，我们还有强大的splice方法，可以在任意位置添加与删除。

splice方法的第一个参数是起始位置，第二个参数是要删除的个数。
我们来看个例子，我们先生成10个元素的数组，然后把前5个空元素都删掉：

```js
let a5 = []
a5.length = 10;
a5[5] = 100;
console.log(a5);
a5.splice(0,5);
console.log(a5);
```

输出结果为：
```
[ <5 empty items>, 100, <4 empty items> ]
[ 100, <4 empty items> ]
```

如果不删除，想要添加元素的话，我们可以给第二个参数置0，然后后面是要添加的元素。比如我们给上面的a5在100后面增加三个新元素1.5, 2.5, 3.5：

```js
a5.splice(1,0,1.5,2.5,3.5);
console.log(a5);
```

输出如下：

```
[ 100, 1.5, 2.5, 3.5, <4 empty items> ]
```

记住是要给元素值，而不是给个数组啊，否则的话就变成二维数组了：

```js
a5.splice(1,0,[1.5,2.5,3.5]);
console.log(a5);
```

结果为：
```
[ 100, [ 1.5, 2.5, 3.5 ], 1.5, 2.5, 3.5, <4 empty items> ]
```

好，复习至此，我们来看tf.js中的张量

## tf.js中的张量

![](https://gw.alicdn.com/imgextra/i3/O1CN014VUvp61V3THjtCw8E_!!6000000002597-0-tps-1080-704.jpg)

### 一维张量

tfjs支持从1d到6d一共6维张量构造函数，当然7维以上没有专用函数了还是可以reshape出来。

最简单的张量是一维的，我们可以用tf.tensor1d：
```js
let t1d = tf.tensor1d([1, 2, 3]);
t1d.print();
```

输出为：
```
Tensor
    [1, 2, 3]
```

当然，还可以指定数据类型：
```js
const t1d_f = tf.tensor1d([1.0,2.0,3.0],'float32')
t1d_f.print();
```

输出结果为：
```
Tensor
    [1, 2, 3]
```

数据类型可用值为：
- 'float32'
- 'int32'
- 'bool'
- 'complex64'
- 'string'

可以通过linspace函数来生成一维序列，其原型为：
```js
tf.linspace (start, stop, num)
```
其中
- start为起始值
- end为结束值
- num为生成的序列的元素个数

例： 
```js
tf.linspace(1, 10, 10).print();
```

输出结果为：
```
Tensor
    [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

如果想用指定步长的方式来生成，可以使用range函数：
```
tf.range(start, stop, step?, dtype?)
```

我们来看个例子：
```js
tf.range(0, 9, 2).print();
```

输出结果为：
```
Tensor
    [0, 2, 4, 6, 8]
```

### 二维张量

![](https://gw.alicdn.com/imgextra/i4/O1CN01n40DwI278vP4Am2BZ_!!6000000007753-0-tps-800-283.jpg)

二维张量可以用二维数组来定义：
```js
let t2d = tf.tensor2d([[0,0],[0,1]]);
t2d.print();
```

不过tf.js的二维张量必须是矩阵，而js的二维数组是可以不等长的，这点尤其要注意。

因为二维张量主要用于存放矩阵，有生成矩阵的方法可供调用。

比如我们可以使用tf.eye来生成单位矩阵：
```js
const t_eye = tf.eye(4);
t_eye.print();
```
 
输出结果为：
```
Tensor
    [[1, 0, 0, 0],
     [0, 1, 0, 0],
     [0, 0, 1, 0],
     [0, 0, 0, 1]]
```

我们也可以将一维向量转化为以其为对角向量的二维向量：
```js
const x1 = tf.tensor1d([1, 2, 3, 4, 5, 6, 7, 8]);
tf.diag(x1).print();
```

输出结果为：
```
Tensor
    [[1, 0, 0, 0, 0, 0, 0, 0],
     [0, 2, 0, 0, 0, 0, 0, 0],
     [0, 0, 3, 0, 0, 0, 0, 0],
     [0, 0, 0, 4, 0, 0, 0, 0],
     [0, 0, 0, 0, 5, 0, 0, 0],
     [0, 0, 0, 0, 0, 6, 0, 0],
     [0, 0, 0, 0, 0, 0, 7, 0],
     [0, 0, 0, 0, 0, 0, 0, 8]]
```

从二维张量开始，我们可以指定张量的形状了。

比如我们用一维数组给定值，然后指定[2,2]的形状：
```js
let t2d2 = tf.tensor2d([1,2,3,4],[2,2],'float32');
t2d2.print();
```

输出结果如下：
```
Tensor
    [[1, 2],
     [3, 4]]
```

### 高维向量

![](https://gw.alicdn.com/imgextra/i2/O1CN01jJ8tw122YVRcv0Vxx_!!6000000007132-2-tps-318-397.png)

从三维开始，用高维数组来表示张量值的可读性就越来越差了。比如：
```js
tf.tensor3d([[[1], [2]], [[3], [4]]]).print();
```

输出结果为：
```
Tensor
    [[[1],
      [2]],

     [[3],
      [4]]]
```

我们可以还是先指定一维数组，然后再指定形状：
```js
tf.tensor3d([1,2,3,4,5,6,7,8],[2,2,2],'int32').print();
```

输出如下：
```
Tensor
    [[[1, 2],
      [3, 4]],

     [[5, 6],
      [7, 8]]]
```

我们向4，5，6维挺进：
```js
tf.tensor4d([[[[1], [2]], [[3], [4]]]]).print();
tf.tensor5d([[[[[1],[2]],[[3],[4]]],[[[5],[6]],[[7],[8]]]]]).print();
tf.tensor6d([[[[[[1],[2]],[[3],[4]]],[[[5],[6]],[[7],[8]]]]]]).print();
```

输出如下：
```
Tensor
    [[[[1],
       [2]],

      [[3],
       [4]]]]
Tensor
    [[[[[1],
        [2]],

       [[3],
        [4]]],


      [[[5],
        [6]],

       [[7],
        [8]]]]]
Tensor
    [[[[[[1],
         [2]],

        [[3],
         [4]]],


       [[[5],
         [6]],

        [[7],
         [8]]]]]]
```

此时，指定形状的优势就更加明显了。

我们可以用tf.zeros函数生成全是0的任意维的张量：
```js
tf.zeros([2,2,2,2,2,2]).print();
```

也可以通过tf.ones将所有值置为1:
```js
tf.ones([3,3,3]).print();
```

还可以通过tf.fill函数生成为指定值的张量：
```js
tf.fill([4,4,4],255).print();
```

比起序列值和固定值，生成符合正态分布的随机值可能是更常用的场景。其原型为：
```js
tf.truncatedNormal(shape, mean?, stdDev?, dtype?, seed?)
```
其中：
- shape是张量形状
- mean是平均值
- stdDev是标准差
- dtype是数据类型，整形和浮点形在此差别可能很大
- seed是随机数种子

我们看个例子：
```js
tf.truncatedNormal([3,3,3],1,1,"float32",123).print();
tf.truncatedNormal([2,2,2],1,1,"int32",99).print();
```

输出如下：
```
Tensor
    [[[0.9669023 , 0.2715541 , 0.6810297 ],
      [-0.8329115, -0.7022814, 1.4331075 ],
      [1.8136243 , 1.8001028 , -0.3285823]],

     [[1.381816  , 1.1050107 , 0.7487067 ],
      [1.9785664 , 0.9248876 , -0.9470147],
      [0.0489896 , 0.3297685 , 0.8626058 ]],

     [[0.3341007 , 1.1067212 , 0.4879217 ],
      [2.1620302 , 1.3034405 , 0.2832415 ],
      [1.3012471 , 1.0853187 , 1.9235317 ]]]
Tensor
    [[[0, 1],
      [1, 0]],

     [[0, 0],
      [1, 2]]]
```

##  将张量转换成js数组

![](https://gw.alicdn.com/imgextra/i4/O1CN01unyeHu1wjbLR9nI4H_!!6000000006344-2-tps-300-168.png)

前面我们学习了很多种张量的生成方法。但是，不知道你意识到了没有，很多时候还是转回到js数组更容易进行一些高阶的操作。

将张量转换成为数组有两种方式，一种是按照原形状转换成数组。异步的可以使用Tensor.array()方法，同步的可以使用Tensor.arraySync()方法。

我们来将上节生成的随机数的向量转回成js的数组：
```js
let t7 = tf.truncatedNormal([2,2,2],1,1,"int32",99);
let a7 = t7.arraySync();
console.log(a7);
```

输出结果为：
```
[ [ [ 0, 1 ], [ 1, 0 ] ], [ [ 0, 0 ], [ 1, 2 ] ] ]
```

记得这是一个高维数组啊，每个元素都是数组。
比如：
```js
a7.forEach(
    (x) => { console.log(x);}
);
```

输出将是两个数组元素：
```
[ [ 0, 1 ], [ 1, 0 ] ]
[ [ 0, 0 ], [ 1, 2 ] ]
```

如果不想要形状，可以用data()或者dataSync()方法将张量转换成TypedArray.

```js
let t5 = tf.truncatedNormal([2,2,2],1,1,"int32",99);
let a5 = t5.dataSync();
console.log(a5);
```

输出结果如下：
```
Int32Array(8) [
  0, 1, 1, 0,
  0, 0, 1, 2
]
```

如果对TypedArray进行forEach操作：
```js
a5.forEach(
    (x) => { console.log(x);}
);
```
获取的结果就是线性的了：
```
0
1
1
0
0
0
1
2
```

拍平成一维的之后，我们就可以用every和some等来进行元素的判断了。
比如我们看a5是不是所有元素都是0，是不是有元素为0：
```js
console.log(a5.every((x) => { return(x===0)}));
console.log(a5.some((x) => { return(x===0)}));
```

因为不全为0，所以every的值为假，而some为真。
