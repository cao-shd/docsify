部署项目需要通过Publish Over SSH插件，让目标服务器执行命令。为了方便一次性实现拉取镜像和启动的命令，推荐采用脚本文件的方式。

添加脚本文件到目标服务器，再通过Publish Over SSH插件让目标服务器执行脚本即可。

-  编写脚本文件，添加到目标服务器 
```shell
vi /usr/local/bin/deploy.sh

harbor_url=$1
harbor_project_name=$2
project_name=$3
tag=$4
port=$5
harbor_user=admin
harbor_pwd=Admin@123456

imageName=$harbor_url/$harbor_project_name/$project_name:$tag

containerId=`docker ps -a | grep ${project_name} | awk '{print $1}'`
if [ "$containerId" != "" ] ; then
    docker stop $containerId
    docker rm $containerId
    echo "Delete Container Success"
fi

imageId=`docker images | grep ${project_name} | awk '{print $3}'`

if [ "$imageId" != "" ] ; then
    docker rmi -f $imageId
    echo "Delete Image Success"
fi

docker login -u $harbor_user -p $harbor_pwd $harbor_url

docker pull $imageName

docker run -d -p $port:$port --name $project_name $imageName

echo "Start Container Success"
echo $project_name
```

- 设置权限为可执行  
```shell
chmod a+x /usr/local/bin/deploy.sh
```
