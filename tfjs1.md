# TensorFlow.js机器学习教程(1) - 穿梭于浏览器与node间

![](https://gw.alicdn.com/imgextra/i4/O1CN01SvbpK71x8KjVYFTqY_!!6000000006398-0-tps-1254-552.jpg)

前端同学学习机器学习，面对汗牛充栋的基于Python的各种教程，虽然不见得不懂Python，但是因为不是后端，想把学到的知识用到工作中还是有一段距离。
Python确实在机器学习和深度学习领域有着不可替代的生态优势，不过，放到浏览器端和手机端，Python的生态优势好像就发挥不出来了。不管是Android手机还是iOS手机，默认都没有Python运行环境，也写不了Python应用。浏览器里和小程序里，就更没Python什么事儿了。

在浏览器里，可以直接使用TensorFlow.js库，尽管可能会有性能的问题，但是至少是从0到1的突破。

我们看个例子：
```html
<!DOCTYPE html>
<html>
    <head>
        <meta encoding="UTF-8"/>
        <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@3.6.0/dist/tf.min.js"></script>
    </head>
    <body>
        <div id="tf-display"></div>
        <script>
            let a = tf.tensor1d([1.0]);
            let d1 = document.getElementById("tf-display");
            d1.innerText = a;
        </script>
    </body>
</html>
```

可以看到，在浏览器里显示了一个值为1.0的张量的值。我们的第一个TensorFlow.js(以下简称tf.js)应用就算是跑通了。通过引用tf.js的库，我们就可以调用tf下面的函数。

下面我们修改一下，看看tf.js是靠什么技术在运行的。我们通过tf.getBackend()函数来查看支持tf.js

```html
<!DOCTYPE html>
<html>
    <head>
        <meta encoding="UTF-8"/>
        <script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@3.6.0/dist/tf.min.js"></script>
    </head>
    <body>
        <div id="tf-display"></div>
        <div id="tf-backend"></div>
        <script>
            let a = tf.tensor1d([1.0,2.0,3.0]);
            let d1 = document.getElementById("tf-display");
            d1.innerText = a;

            let backend = tf.getBackend();
            let div_backend = document.getElementById("tf-backend");
            div_backend.innerText = backend;
        </script>
    </body>
</html>
```

在我的浏览器里，tf.js是使用webgl来进行计算的。

## 运行在node里的tfjs

作为一个js库，tf.js当然也可以运行在node环境里。我们可以通过
```
npm install @tensorflow/tfjs
```
来安装tf.js库。

然后把上面网页里面的代码移值过来：
```js
const tf = require('@tensorflow/tfjs');

let a = tf.tensor1d([1.0,2.0,3.0]);
console.log(a);

console.log(tf.getBackend());
```

在我的电脑里执行，这个getBackend()返回的是'cpu'. 
tf.js还会给tfjs-node做个广告：
```
============================
Hi there 👋. Looks like you are running TensorFlow.js in Node.js. To speed things up dramatically, install our node backend, which binds to TensorFlow C++, by running npm i @tensorflow/tfjs-node, or npm i @tensorflow/tfjs-node-gpu if you have CUDA. Then call require('@tensorflow/tfjs-node'); (-gpu suffix for CUDA) at the start of your program. Visit https://github.com/tensorflow/tfjs-node for more details.
============================
```

听人劝吃饱饭，那我们就换成tfjs-node吧：

```js
const tf = require('@tensorflow/tfjs-node');

let a = tf.tensor1d([1.0,2.0,3.0]);
console.log(a);

console.log(tf.getBackend());
```

记得要
```
npm install @tensorflow/tfjs-node
```

现在，后端从cpu换成了tensorflow。

还有更凶残的，我们还可以换成tfjs-node-gpu来使用GPU：
```js
const tf = require('@tensorflow/tfjs-node-gpu');

let a = tf.tensor1d([1.0,2.0,3.0]);
console.log(a);

console.log(tf.getBackend());
```
在没有GPU的机器上，会使用CPU版的tensorflow作为后端，不会报错。

## 使用tfjs预测波士顿房价

按照国际惯例，我们的tfjs机器学习之旅也从预测房价开始。

我们先搞个node里运行的版本，这样我们可以使用tf.data.csv去读取csv数据。而在浏览器里，我们就得求助于强大的papaparser之类的库了。

首先我们先指定数据的URL，直接为网上的地址：
```js
const csvUrl =
    'https://storage.googleapis.com/tfjs-examples/multivariate-linear-regression/data/boston-housing-train.csv';
```

我们要预测的是房价中位数medv字段，先定义好：
```js
    const csvDataset = tf.data.csv(
        csvUrl, {
            columnConfigs: {
                medv: {
                    isLabel: true
                }
            }
        });
```

然后转换一下格式：
```js
    // Number of features is the number of column names minus one for the label
    // column.
    const numOfFeatures = (await csvDataset.columnNames()).length - 1;

    // Prepare the Dataset for training.
    const flattenedDataset =
        csvDataset
            .map(({xs, ys}) =>
            {
                // Convert xs(features) and ys(labels) from object form (keyed by
                // column name) to array form.
                return {xs:Object.values(xs), ys:Object.values(ys)};
            })
            .batch(10);
```

线性回归就是没有隐藏层的一层。输入是numOfFeatures个元素，输出是1个元素：
```js
    const model = tf.sequential();
    model.add(tf.layers.dense({
        inputShape: [numOfFeatures],
        units: 1
    }));
```

最后设置优化算法、学习率等参数，编译运行就好了：
```js
    model.compile({
        optimizer: tf.train.sgd(0.000001),
        loss: 'meanSquaredError'
    });

    return model.fitDataset(flattenedDataset, {
        epochs: 10,
        callbacks: {
            onEpochEnd: async (epoch, logs) => {
                console.log(epoch + ':' + logs.loss);
            }
        }
    });
```

其中说一下callbacks吧，支持一些生命周期的回调。除了onEpochEnd，还有onEpochBegin, onTrainBegin, onTrainEnd等：

```js
    return model.fitDataset(flattenedDataset, {
        epochs: 10,
        callbacks: {
            onEpochEnd: async (epoch, logs) => {
                console.log(epoch + ':' + logs.loss);
            },
            onTrainBegin: async(logs) => {
                console.log("Train Begin!");
            },
            onTrainEnd: async(logs) => {
                console.log('Train End!');
            }
        }
    });
```

用Node运行，就可以看到训练的过程了：
```
Train Begin!
Epoch 1 / 10
eta=0.0 ================================================================================> 
1124ms 41639us/step - loss=1445.89 
0:1445.89404296875
Epoch 2 / 10
eta=0.0 ================================================================================> 
1002ms 37093us/step - loss=254.11 
1:254.1094970703125
Epoch 3 / 10
eta=0.0 ================================================================================> 
989ms 36618us/step - loss=250.23 
2:250.233642578125
Epoch 4 / 10
eta=0.0 ================================================================================> 
965ms 35746us/step - loss=246.57 
3:246.57472229003906
Epoch 5 / 10
eta=0.0 ================================================================================> 
974ms 36056us/step - loss=243.12 
4:243.11817932128906
Epoch 6 / 10
eta=0.0 ================================================================================> 
955ms 35358us/step - loss=239.85 
5:239.85049438476562
Epoch 7 / 10
eta=0.0 ================================================================================> 
963ms 35659us/step - loss=236.76 
6:236.75921630859375
Epoch 8 / 10
eta=0.0 ================================================================================> 
942ms 34904us/step - loss=233.83 
7:233.83265686035156
Epoch 9 / 10
eta=0.0 ================================================================================> 
963ms 35650us/step - loss=231.06 
8:231.05992126464844
Epoch 10 / 10
eta=0.0 ================================================================================> 
920ms 34064us/step - loss=228.43 
9:228.43093872070312
Train End!
```

完整的代码如下：
```js
const tf = require('@tensorflow/tfjs-node');

const csvUrl =
    'https://storage.googleapis.com/tfjs-examples/multivariate-linear-regression/data/boston-housing-train.csv';

async function run() {
    // We want to predict the column "medv", which represents a median value of
    // a home (in $1000s), so we mark it as a label.
    const csvDataset = tf.data.csv(
        csvUrl, {
            columnConfigs: {
                medv: {
                    isLabel: true
                }
            }
        });

    // Number of features is the number of column names minus one for the label
    // column.
    const numOfFeatures = (await csvDataset.columnNames()).length - 1;

    // Prepare the Dataset for training.
    const flattenedDataset =
        csvDataset
            .map(({xs, ys}) =>
            {
                // Convert xs(features) and ys(labels) from object form (keyed by
                // column name) to array form.
                return {xs:Object.values(xs), ys:Object.values(ys)};
            })
            .batch(10);

    // Define the model.
    const model = tf.sequential();
    model.add(tf.layers.dense({
        inputShape: [numOfFeatures],
        units: 1
    }));
    model.compile({
        optimizer: tf.train.sgd(0.000001),
        loss: 'meanSquaredError'
    });

    // Fit the model using the prepared Dataset
    return model.fitDataset(flattenedDataset, {
        epochs: 10,
        callbacks: {
            onEpochEnd: async (epoch, logs) => {
                console.log(epoch + ':' + logs.loss);
            },
            onTrainBegin: async(logs) => {
                console.log("Train Begin!");
            },
            onTrainEnd: async(logs) => {
                console.log('Train End!');
            }
        }
    });
}

run();
```


## 在浏览器里预测波士顿房价

![boston](https://gw.alicdn.com/imgextra/i3/O1CN018HUiij1VBG17MSrnF_!!6000000002614-2-tps-1240-727.png)

运行在浏览器里，上面的核心算法都不用变，只是对数据处理和UI的部分要复杂一些。

csv读取我们换用强大的papaparser:
```js
export const loadCsv = async (filename) => {
  return new Promise(resolve => {
    const url = `${BASE_URL}${filename}`;

    console.log(`  * Downloading data from: ${url}`);
    Papa.parse(url, {
      download: true,
      header: true,
      complete: (results) => {
        resolve(parseCsv(results['data']));
      }
    })
  });
};
```

模型还是那个模型：
```js
export function linearRegressionModel() {
  const model = tf.sequential();
  model.add(tf.layers.dense({inputShape: [bostonData.numFeatures], units: 1}));

  model.summary();
  return model;
};
```

compile的过程也是一样的：
```js
  model.compile(
      {optimizer: tf.train.sgd(LEARNING_RATE), loss: 'meanSquaredError'});
```

训练的过程因为要更新UI变得更复杂了一些：
```js
  await model.fit(tensors.trainFeatures, tensors.trainTarget, {
    batchSize: BATCH_SIZE,
    epochs: NUM_EPOCHS,
    validationSplit: 0.2,
    callbacks: {
      onEpochEnd: async (epoch, logs) => {
        await ui.updateModelStatus(
            `Epoch ${epoch + 1} of ${NUM_EPOCHS} completed.`, modelName);
        trainLogs.push(logs);
        tfvis.show.history(container, trainLogs, ['loss', 'val_loss'])

...
      }
    }
  });

  ui.updateStatus('Running on test data...');
  const result = model.evaluate(
      tensors.testFeatures, tensors.testTarget, {batchSize: BATCH_SIZE});
  const testLoss = result.dataSync()[0];
```

浏览器里的完整代码请参看：https://github.com/tensorflow/tfjs-examples/blob/master/boston-housing/，这里希望更多给大家一个宏观的印象，就不深入细节了。
