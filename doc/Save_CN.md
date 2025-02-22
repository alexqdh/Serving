# 怎样保存用于Paddle Serving的模型？

(简体中文|[English](./Save_EN.md))

## 从已保存的模型文件中导出
如果已使用Paddle 的`save_inference_model`接口保存出预测要使用的模型，你可以使用Paddle Serving提供的名为`paddle_serving_client.convert`的内置模块进行转换。
```python
python -m paddle_serving_client.convert --dirname ./your_inference_model_dir
```

也可以通过Paddle Serving的`inference_model_to_serving`接口转换成可用于Paddle Serving的模型文件。
```python
import paddle_serving_client.io as serving_io
serving_io.inference_model_to_serving(dirname, serving_server="serving_server", serving_client="serving_client",  model_filename=None, params_filename=None)
```

模块参数与`inference_model_to_serving`接口参数相同。
| 参数 | 类型 | 默认值 | 描述 |
|--------------|------|-----------|--------------------------------|
| `dirname` | str | - | 需要转换的模型文件存储路径，Program结构文件和参数文件均保存在此目录。|
| `serving_server` | str | `"serving_server"` | 转换后的模型文件和配置文件的存储路径。默认值为serving_server |
| `serving_client` | str | `"serving_client"` | 转换后的客户端配置文件存储路径。默认值为serving_client |
| `model_filename` | str | None | 存储需要转换的模型Inference Program结构的文件名称。如果设置为None，则使用 `__model__` 作为默认的文件名 |
| `params_filename` | str | None | 存储需要转换的模型所有参数的文件名称。当且仅当所有模型参数被保>存在一个单独的二进制文件中，它才需要被指定。如果模型参数是存储在各自分离的文件中，设置它的值为None |


### 从动态图模型中导出

PaddlePaddle 2.0提供了全新的动态图模式，因此我们这里以imagenet ResNet50动态图为示例教学如何从已保存模型导出，并用于真实的在线预测场景。

```
wget https://paddle-serving.bj.bcebos.com/others/dygraph_res50.tar #模型
wget https://paddle-serving.bj.bcebos.com/imagenet-example/daisy.jpg #示例输入（向日葵）
tar xf dygraph_res50.tar
python -m paddle_serving_client.convert --dirname . --model_filename dygraph_model.pdmodel --params_filename dygraph_model.pdiparams --serving_server serving_server --serving_client serving_client
```
我们可以看到`serving_server`和`serving_client`文件夹分别保存着模型的服务端和客户端配置

启动服务端（GPU）
```
python -m paddle_serving_server.serve --model serving_server --port 9393 --gpu_id 0
```

客户端写法，保存为`test_client.py`
```
from paddle_serving_client import Client
from paddle_serving_app.reader import Sequential, File2Image, Resize, CenterCrop
from paddle_serving_app.reader import RGB2BGR, Transpose, Div, Normalize

client = Client()
client.load_client_config(
    "serving_client/serving_client_conf.prototxt")
client.connect(["127.0.0.1:9393"])

seq = Sequential([
    File2Image(), Resize(256), CenterCrop(224), RGB2BGR(), Transpose((2, 0, 1)),
    Div(255), Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225], True)
])

image_file = "daisy.jpg"
img = seq(image_file)
fetch_map = client.predict(feed={"inputs": img}, fetch=["save_infer_model/scale_0.tmp_0"])
print(fetch_map["save_infer_model/scale_0.tmp_0"].reshape(-1))
```
执行
```
python test_client.py
```
即可看到成功的执行了预测，以上就是动态图ResNet50模型在Serving上预测的内容，其他动态图模型使用方式与之类似。

## 从训练或预测脚本中保存(静态图)
目前，Paddle Serving提供了一个save_model接口供用户访问，该接口与Paddle的`save_inference_model`类似。

``` python
import paddle_serving_client.io as serving_io
serving_io.save_model("imdb_model", "imdb_client_conf",
                      {"words": data}, {"prediction": prediction},
                      paddle.static.default_main_program())
```
imdb_model是具有服务配置的服务器端模型。 imdb_client_conf是客户端rpc配置。

Serving有一个提供给用户存放Feed和Fetch变量信息的字典。 在示例中，`{"words"：data}` 是用于指定已保存推理模型输入的提要字典。`{"prediction"：projection}`是指定保存的推理模型输出的字典。可以为feed和fetch变量定义一个别名。 如何使用别名的例子 示例如下：

 ``` python
 from paddle_serving_client import Client
import sys

client = Client()
client.load_client_config(sys.argv[1])
client.connect(["127.0.0.1:9393"])

for line in sys.stdin:
    group = line.strip().split()
    words = [int(x) for x in group[1:int(group[0]) + 1]]
    label = [int(group[-1])]
    feed = {"words": words, "label": label}
    fetch = ["acc", "cost", "prediction"]
    fetch_map = client.predict(feed=feed, fetch=fetch)
    print("{} {}".format(fetch_map["prediction"][1], label[0]))
 ```
