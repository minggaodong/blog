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

### 使用步骤
- docker 拉取 tensorflow/serving 镜像，镜像分为 CPU 和 GPU 两种版本，选取一个适合自己的。
- 运行1个 docker 容器，并设置模型的名称，模型文件在宿主机的磁盘映射，端口映射等，此时在线 Serving 服务已经构建成功了。
- 客户端通过 gRPC/RestApi 调用 Serving 的 Predict/predict 方法，调用时需要指定模型名称。

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
通过-e参数设置环境变量MODEL_NAME

容器内部分别用8500和8501作为gRPC和RestApi的端口，通过-p参数将容器内部端口开放出来。
```
TESTDATA="$(pwd)/serving/tensorflow_serving/servables/tensorflow/testdata"
docker run -dt -p 8500:8500 -p 8501:8501 -v "$TESTDATA/saved_model_half_plus_two_cpu:/models/half_plus_two" -e MODEL_NAME=half_plus_two tensorflow/serving
```

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

### 模型版本
一个模型下可以包含多个版本，每个版本是一个子目录，默认只加载版本号最大的那个文件。

如果想同时支持两个版本，则需要编写一个json配置文件，在文件中指定加载多版本
```
model_config_list: {
     config: {
        name: "model1",
        base_path: "/models/model1",
        model_version_policy {
            specific {
                versions: 2
                versions: 3
            }
        }
      }
}
```

### 多模型部署
多模型或者单模型多版本部署时，无法在命令行中指定MODEL_NAME了，需要编写一个如下的json配置文件，这里取名为model.config。

默认是部署目录下最新版本的模型，如果要部署全部或者特定版本，通过model_version_policy设置。
```
model_config_list: {
    config: {
        name: "model1",
        base_path: "/models/model1",
        
        model_version_policy: {
           all: {}
    },
    config: {
        name: "model2",
        base_path: "/models/model2",
        model_platform: "tensorflow",
        model_version_policy: {
           latest: {
               num_versions: 1
           }
        }
    },
    config: {
        name: "model3",
        base_path: "/models/model3",
        model_platform: "tensorflow",
        model_version_policy: {
           specific: {
               versions: 1
           }
        }
    }
}
```


