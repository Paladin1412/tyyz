#制作 OCR docker镜像

## 镜像打包步骤

### 下载申请
    http://lab.oa.com/#/privateVersion

### 基础容器
    1， 以 centos7.6.1810:ocr.base 作为基础镜像启动容器
        基础镜像位置，企业云盘： /团队文件/TI_MATRIX/youtu/youtu_ocr_service/ocr.docker.image.base.tgz
    2， 映射安装文件所在目录到容器中
    3， 映射 license 目录
    4， 使用主机网络

    5， 示例：
    docker run -it --net=bridge --privileged --name youtu_ocr -v /root/work:/workspace -v /root/work/license:/data/youtu/license centos7.6.1810:ocr.base bash

### 进容器安装服务
	docker exec -it youtu_ocr bash
	... 常规安装步骤
	... 

### 增加 /root/entrypoint.sh
    entrypoint.sh 复制到容器内 /root/entrypoint.sh
    并修改以下两行为服务对应版本：

SERVICE_NAME="youtu_private_bizlicenseocr"
SERVICE_VER="2.3.2"


### 处理 libcuda
OCR服务显式链接了 libcuda.so.1 ，就算CPU版不用cuda，也需要带 libcuda.so.1 ，使用服务提供的runtime就行。

而对于GPU版，则需要使用主机相同版本的 libcuda。方法：
    起容器时，映射主机上 libcuda.so.1 以及 libnvidia-fatbinaryloader.so.***.** 到容器中。
    修改LD_LIBRARY_PATH ，指定主机映射的lib目录为先，runtime随其后。

### 脚本
    可以改成 docker Makefile
``` bash

docker rm ocr_test youtu_ocr
    
IMAGE=centos7.6.1810:ocr.base
docker run -it --net=bridge --privileged --name youtu_ocr -v /root/workspace:/workspace -v /data/youtu:/data/youtu ${IMAGE} bash

## install service
## ...

## 修改 /workspace/youtu.ocr/entrypoint.sh
## 以下变量改为对应服务正确值
SERVICE_NAME=youtu_private_handwriting_ocr
SERVICE_VER=1.3.0.2

## 
mkdir /usr/local/services
cp -rf /workspace/youtu.ocr/runtime /usr/local/services
cp -rf /workspace/youtu.ocr/entrypoint.sh /root

bash /root/entrypoint.sh

## 如果错误提示 dlopen handle null, err libcudnn.so.7
## 或者       dlopen handle null, err libcusparse.so.9.0
## 或者 其它缺少 cuda 库文件的情况
## 需要安装 cuda9 依赖库： youtu_ocr_common_runtime_cuda9_private-1.0.3-install.tar.gz
## 并且解压其中的 libcaffe2_gpu.so.zip
## 注意 cuda9 依赖库较大，实际项目需要GPU或者提示缺少库文件的情况，才需要在镜像中安装 cuda9 依赖库
## 然后 修改 /root/entrypoint.sh，增加动态库搜索路径
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/usr/local/services/youtu_ocr_common_runtime_cuda9_private-1.0/shared:/usr/local/services/runtime

## test api
TEST_PORT=59099
TEST_API=generalocr
bash /workspace/youtu.ocr/ocr.test/ocr.test.sh localhost ${TEST_PORT} ${TEST_API} /workspace/youtu.ocr/ocr.test/


# 多数用户需要一个心跳接口监测服务健康状态；
# 查看服务文档，如果服务没有提供心跳接口，可以自己手工添加。
# 增加心跳接口
cd /root/services/${SERVICE_NAME}
vi conf/nginx.conf
# 在服务包的conf/nginx.conf文件 /http/server 段里面加一段：
        location ^~ /youtu/ocrapi/heartbeat {
            if ($request_method !~ ^(GET)$ ) {
                return 400;
            }
            return 200;
        }

# 重启服务
/root/services/${SERVICE_NAME}/restart.sh all

# 依赖库
## 如果错误提示 err libgomp.so.1: cannot open shared object file: No such file or directory
## 需要安装 libgomp
yum install libgomp-4.8.5-36.el7_6.2.x86_64.rpm
## 如果错误提示 err libcaffe2_gpu.so: cannot open shared object file: No such file or directory
## 需要安装 cuda9 运行库，并解压其中的 libcaffe2_gpu.so.zip
## 并且修改 admin/start.sh, admin/restart.sh , 增加动态库搜索路径：
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/root/services/youtu_ocr_common_runtime_cuda9_private/shared


# IP换成当前服务器的IP
IP=localhost
# 测试
curl -v ${IP}:${TEST_PORT}/youtu/ocrapi/heartbeat
# 输出 HTTP/1.1 200 OK 即服务心跳OK

## save and export images
docker commit --message="installed ${SERVICE_NAME}:${SERVICE_VER}" youtu_ocr ccr.ccs.tencentyun.com/timatrix-test/${SERVICE_NAME}:${SERVICE_VER}

docker save ccr.ccs.tencentyun.com/timatrix-test/${SERVICE_NAME}:${SERVICE_VER} > ${SERVICE_NAME}.${SERVICE_VER}.tar
tar -czvf ${SERVICE_NAME}.${SERVICE_VER}.tgz ${SERVICE_NAME}.${SERVICE_VER}.tar
    
## test image
docker run -it --privileged --name ocr_test -v /data/youtu:/data/youtu -v /data/log:/data/log ccr.ccs.tencentyun.com/timatrix-test/${SERVICE_NAME}:${SERVICE_VER} /root/entrypoint.sh
```

### 更新镜像包
制作完成之后，上传到 COS，方法
    https://docs.qq.com/doc/DYnBrYXl4RnpoZm5l?opendocxfrom=admin 
上传路径
    /OCR-release/OCR.docker.images
