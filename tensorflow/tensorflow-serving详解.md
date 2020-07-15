# TensorFlow-Serving在线预估服务构建

项目地址：[tensorflow-serving](https://github.com/tensorflow/serving)

## 简介
TensorFlow-Serving 是 Google 开源的机器学习在线 Serving 框架，支持将离线模型方便的部署到线上，并开放接口给外部调用。

### 功能特点
- 支持模型版本控制和回滚
- 支持多模型服务
- 支持热更新
- 支持 gRPC 和 RestApi 两种对外接口
- 支持批处理，高吞吐量
- 支持 docker + k8s 动态扩缩容

## 安装运行
TensorFlow-Serving 支持 docker，二进制，源码编译三种安装方式，强烈推荐采用 docker 安装（除非有特殊需求无法基于容器运行）。

### 安装测试
TensorFlow-Serving 源码中提供了一些测试模型，以此为例从零开始构建一个在线serving。

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
参数说明
- -p：端口映射，在容器内部，gRPC的端口是8500，REST的端口是8501
- -v：目录映射，新版中已经移除–mount type=bind,source=%source_path,target=$target_path的挂载目录方式
- -e：设置docker环境变量：模型名称：MODEL_NAME；模型路径：MODEL_BASE_PATH， 默认值为 /models

#### 请求测试
这里使用RestApi接口请求测试
```
curl -d '{"instances": [1.0, 2.0, 5.0]}' -X POST http://localhost:8501/v1/models/half_plus_two:predict
```
返回结果
```
{
    "predictions": [2.5, 3.0, 4.5]
}
```

### 模型配置文件
serving需要支持多模型服务时，需要使用模型配置文件。

#### 启动 serving (带模型配置)
```
docker run -dt -p 8501:8501 -v "$TESTDATA/:/models/" tensorflow/serving --model_config_file=/models/models.config --model_config_file_poll_wait_seconds=60
```
参数说明
- --model_config_file：设置模型配置文件的路径，配置文件名可以自定义
- --model_config_file_poll_wait_seconds：系统定时检查配置文件是否修改，该配置为检查的间隔时间

#### 配置reload方式
- 配置文件定时检查：通过--model_config_file_poll_wait_seconds设置检查的间隔频率。
- RPC调用：主动调用gRPC的HandleReloadConfigRequest接口，执行reload。

#### 多模型配置示例
```
model_config_list {
  config {
    name: 'half_plus_two'
    base_path: '/models/saved_model_half_plus_two_cpu'
    model_platform: 'tensorflow'
  }

  config {
    name: 'half_plus_three'
    base_path: '/models/saved_model_half_plus_three'
    model_platform: 'tensorflow'
  }
}
```

#### 多版本控制
模型目录下可以有多个版本(版本号为子目录名)，serving 加载时默认只加载版本号最大的那个模型；版本号必须是整数，一般用时间戳作为版本号。

如果要同时加载多个版本，需要在模型配置文件中通过 model_version_policy 指定, 支持all(全部加载)和specific(指定versions)两种配置。

##### 配置示例
```
model_config_list {
  config {
    name: 'saved_model_half_plus_two_2_versions'
    base_path: '/models/saved_model_half_plus_two_2_versions'
    model_platform: 'tensorflow'
    model_version_policy {
      specific {
        versions: 123
        versions: 124
      }
    }
  }
}
```

##### 客户端指定版本
gRPC和Rest客户端调用模型时可以指定版本号，如果不指定默认使用版本号最大的。
```
curl -d '{"instances": [1.0, 2.0, 5.0]}' -X POST http://localhost:8501/v1/models/half_plus_two/versions/123:predict
```

##### 版本别名
版本可以起一个别名，客户端访问模型时用别名替代版本号，当版本更新时，只需修改模型配置文件，而客户端对版本变化是不感知的。

版本别名只能在对应模型加载成功后才有效，这样设计是防止版本切换时，版本请求指向无效的模型。

Rest客户端不支持版本别名，gRPC客户端支持。

##### 别名配置示例
```
model_config_list {
  config {
    name: 'saved_model_half_plus_two_2_versions'
    base_path: '/models/saved_model_half_plus_two_2_versions'
    model_platform: 'tensorflow'
    model_version_policy {
      all {}
    }
    version_labels {
        key: 'stable'
        value: 22
    }
    version_labels {
        key: 'gray'
        value: 23
    }
  }
}

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

### 模型热更新
tf-serving 会监控模型根目录下的文件变化，并根据模型配置分析决定是否启动热更新，如果变化涉及模型更新，则会立刻执行热更新。

模型的热更新和配置文件的检查更新是各自独立执行，模型热更新时，仍然沿用内存中已加载的模型配置。

#### 模型升级
- 模型配置为不指定版本时，默认加载最新的版本，当在根目录下创建一个新版本时，立即触发热更新，tf-serving会卸载旧模型，加载新模型。
- 模型配置为加载所有版本并使用别名，当在根目录创建新版本时，触发热更新并立刻被加载，但此时客户端不能使用，需要修改配置文件中别名的指向，并等待配置文件更新生效。

#### 模型回滚
- 模型配置为不指定版本时，默认加载最新的版本，此时删除最新版本，模型会自动回滚到上一个版本。

## 客户端调用
客户端支持RestApi和gRPC两种接口，通过RestApi接口可以很方便的查看模型的一些信息和状态

### Rest接口
#### 查看模型状态
```
curl http://localhost:8501/v1/models/half_plus_two
```

#### 查看模型信息
```
curl http://localhost:8501/v1/models/half_plus_two/metadata
```
### gRPC接口
#### gRPC客户端示例
[gRPC_Client](https://github.com/minggaodong/serving/blob/master/tensorflow_serving/example/resnet_client.cc)



