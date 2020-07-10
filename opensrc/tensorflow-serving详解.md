# TensorFlow-Serving在线预估服务构建

项目地址：[tensorflow-serving](https://github.com/tensorflow/serving)

## 简介
TensorFlow-Serving 是 Google 开源的机器学习在线 Serving 框架，支持将离线模型方便的部署到线上，并开放接口给外部调用。

#### 功能特点
- 支持模型版本控制和回滚
- 支持高并发，实现高吞吐量
- 支持 gRPC 和 RestApi 两种对外接口
- 开箱即用，并且可定制化
- 支持多模型服务
- 支持批处理
- 支持热更新

## 安装部署
TensorFlow-Serving 支持 docker，二进制，源码编译三种安装方式，强烈推荐采用 docker 安装（除非有特殊需求无法基于容器运行）。

### 安装测试
TensorFlow-Serving 源码中提供了一些测试模型，现在用来从零开始构建一个在线serving。

#### 下载docker镜像
```
docker pull tensorflow/serving                   // 默认安装TensorFlow Serving二进制文件的最小镜像
docker pull tensorflow/serving:latest            // 安装TensorFlow Serving二进制文件的最小镜像
docker pull tensorflow/serving:latest-gpu        // 安装TensorFlow Serving二进制文件的最小镜像(支持GPU)
docker pull tensorflow/serving:latest-devel      // 包括要开发的所有源/依赖项/工具链，以及在CPU上运行的已编译二进制文件
docker pull tensorflow/serving:latest-devel-gpu  // 包括要开发的所有源代码依赖项/工具链（cuda9 / cudnn7），以及可在NVIDIA GPU上运行的已编译二进制文件。
```

#### 下载测试模型
```
git clone git@github.com:tensorflow/serving.git
```

#### 运行 serving 服务
```
TESTDATA="$(pwd)/serving/tensorflow_serving/servables/tensorflow/testdata"
docker run -dt -p 8500:8500 -p 8501:8501 -v "$TESTDATA/saved_model_half_plus_two_cpu:/models/half_plus_two" -e MODEL_NAME=half_plus_two tensorflow/serving
```
参数解释
- -p：端口映射，在容器内部，gRPC的端口是8500，REST的端口是8501
- -v：目录映射，新版中已经移除–mount type=bind,source=%source_path,target=$target_path的挂载目录方式
- -e：设置环境变量，docker内部 MODEL_NAME 默认值为 model，MODEL_BASE_PATH 默认值为 /models

#### 查看模型信息
```
curl http://localhost:8501/v1/models/half_plus_two/metadata
```

#### 测试调用
这里使用RestApi接口测试
```
curl -d '{"instances": [1.0, 2.0, 5.0]}' -X POST http://localhost:8501/v1/models/half_plus_two:predict

返回结果：
{
    "predictions": [2.5, 3.0, 4.5]
}
```

## 使用配置文件
启动serving时，可以使用类似json格式的配置文件，注意需要多模型加载时必须使用配置文件指定。

### 带配置文件启动serving
```
docker run -dt -p 8501:8501 -v "$(pwd)/models/:/models/" tensorflow/serving --model_config_file=/models/models.config --model_config_file_poll_wait_seconds=60
```

### 加载多模型
```
model_config_list {
  config {
    name: 'my_first_model'
    base_path: '/tmp/my_first_model/'
  }
  config {
    name: 'my_second_model'
    base_path: '/tmp/my_second_model/'
  }
}
```

### 加载多版本
模型目录下可以有多个版本，通过 model_version_policy 指定加载版本，如果不指定默认加载版本号最大的(最新)。
```
model_version_policy {
  specific {
    versions: 42
    versions: 43
  }
}
```

gRPC和Rest客户端调用模型时可以指定版本号，如果不指定默认使用版本号最大的。
```
curl -d '{"instances": [1.0, 2.0, 5.0]}' -X POST http://localhost:8501/v1/models/half_plus_two:predict
curl -d '{"instances": [1.0, 2.0, 5.0]}' -X POST http://localhost:8501/v1/models/half_plus_two/versions/00000123:predict
```

### 版本号别名
可以给版本号设置别名，当版本更新时，只需要修改别名配置，而客户端使用别名访问，对版本更新是不感知的。

版本别名只能分配给已经加载的模型，这样设计是防止别名请求指到无效的模型上，当模型加载完成，配置被reload时别名才生效。

```
version_labels {
  key: 'stable'
  value: 42
}
version_labels {
  key: 'canary'
  value: 43
}
```
gRPC和Rest客户端调用模型时可以指定版本别名
```
curl -d '{"instances": [1.0, 2.0, 5.0]}' -X POST http://localhost:8501/v1/models/half_plus_two/labels/stable:predict
```

## 模型部署
### 模型格式
离线训练时保存的模型是 checkpoint 格式，而 TensorFlow Serving 则支持的是servable_model格式。

tensorflow-estimator支持将模型导出为servable_model格式，包含一个pb文件和一个variables目录。

```
-ckpt_model
        -checkpoint
        -***.ckpt.data-00000-of-00001
        -***.ckpt.index
        -***.ckpt.meta
        
-servable_model
        -version
                -saved_model.pb
                -variables
```

### 热更新
serving 热更新时，会检查配置文件是否有变动，并只更新有变动的内容，比如配置文件模型A改成了模型B，serving会加载B并卸载A。

#### 热更新触发方式
- 配置文件定时检查：通过设置--model_config_file_poll_wait_seconds检查更新配置。
- RPC调用：主动调用gRPC的HandleReloadConfigRequest接口，进行检查更新。


