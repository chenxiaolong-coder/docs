<div align="center">
    <h1>jina-now-v0.0.52 本地化部署</h1>
</div>

```
jina-now 是基于jina 框架开发的一款小型应用项目，其用到的技术栈为: nginx elastic python jina 等等
```


***1 安装nginx***

```
确保系统中已经安装nginx, 后面的命令行会自行调用 nginx -c xxx.conf 启动nginx 进程
```
示例: ubuntu 系统下安装nginx
```
su root apt-get install nginx
```

***2 安装 elastic-search 服务***

示例: docker-compose 方式安装【其它方式自行查阅】

```
docker-compose -f docker-compose.yml up -d
```
docker-compose.xml 文件内容如下：
```
version: "3.3"
services:
  elastic:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.5.1
    environment:
      - xpack.security.enabled=false
      - discovery.type=single-node
    ports:
      - "9200:9200"
    networks:
      - elastic
    deploy:
      resources:
        limits:
          memory: 2000M  # Use at most 50 MB of RAM

networks:
  elastic:
    name: elastic

```


***4 下载 now 源码***
```
git clone https://github.com/jina-ai/now.git
```

***5 安装 now 的python依赖包***
```
pip install -r requirement.txt
```
为避免python 环境依赖包冲突，建议通过anaconda 或 miniconda 虚拟环境进行安装。【python 环境的搭建请查询自行查阅相关资料】

requirements.txt 内容如下:
```
pyfiglet==0.8.post1
jina[perf]==3.14.2.dev18
jina-hubble-sdk>=0.15.5
pyfiglet==0.8.post1
cowsay==4.0
yaspin==2.1.0
py-cpuinfo==8.0.0
prompt_toolkit==3.0.28
docarray[common]==0.19.1
requests==2.27.1
pydub==0.25.1
nltk==3.7
jcloud==0.2.4
python-dotenv
boto3==1.26.43
elasticsearch==8.4.1
pydantic==1.9.2
httpx==0.23.2
av==9.1.1
pandas==1.3.5
streamlit==1.16.0
filetype==1.2.0
```

***5 通过flow.xml 安装 jina-now 应用***

```
cd now/now/executor
jina flow --uses flow.xml
```
目录 now/now/executor 下创建flow.xml，内容如下：
```
jtype: Flow
with:
  name: nowapi
  env:
    JINA_LOG_LEVEL: DEBUG
    M2M:
gateway:
  uses:
    jtype: NOWGateway
    py_modules:
      - gateway/now_gateway.py
  protocol:
    - http
  port:
    - 8081
  monitoring: true
  cors: true
  uses_with:
    user_input_dict:
      flow_name: cxl
      dataset_type: path
      dataset_name: null
      dataset_path: /home/chenxiaolong/Documents/images
      aws_access_key_id: null
      aws_secret_access_key: null
      aws_region_name: null
      index_fields:
        - .jpg
      index_field_candidates_to_modalities:
        .jpg: image
      filter_fields: []
      filter_field_candidates_to_modalities: {}
      field_names_to_dataclass_fields:
        .jpg: image_0
      model_choices:
        .jpg_model:
          - encoderclip
      es_index_name: null
      es_host_name: null
      es_additional_args: null
      cluster: null
      secured: false
      jwt:
        token: d3d6d841df41f50c4e0b20d04c74ef4b:fbea49d85d06771c3ec3c26c2f6f17e383d3fbc5
      admin_name: ober
      admin_emails:
        - 1049124443@qq.com
      user_emails: null
      additional_user: null
      api_key: null
  env:
    JINA_LOG_LEVEL: DEBUG
executors:
  - name: autocomplete_executor
    uses:
      jtype: NOWAutoCompleteExecutor2
      py_modules:
        - autocomplete/executor.py
    needs: gateway
    env:
      JINA_LOG_LEVEL: INFO
    uses_with:
      user_input_dict: &id001
        flow_name: cxl
        dataset_type: path
        dataset_name: null
        dataset_path: /home/chenxiaolong/Documents/images
        aws_access_key_id: null
        aws_secret_access_key: null
        aws_region_name: null
        index_fields:
          - .jpg
        index_field_candidates_to_modalities:
          .jpg: image
        filter_fields: []
        filter_field_candidates_to_modalities: {}
        field_names_to_dataclass_fields:
          .jpg: image_0
        model_choices:
          .jpg_model:
            - encoderclip
        es_index_name: null
        es_host_name: null
        es_additional_args: null
        cluster: null
        secured: false
        jwt:
          token: d3d6d841df41f50c4e0b20d04c74ef4b:fbea49d85d06771c3ec3c26c2f6f17e383d3fbc5
        admin_name: ober
        admin_emails:
          - 1049124443@qq.com
        user_emails: null
        additional_user: null
        api_key: null
      api_keys: &id002 []
      user_emails: &id003 []
      admin_emails: &id004 []
  - name: preprocessor
    needs: autocomplete_executor
    uses:
      jtype: NOWPreprocessor
      py_modules:
        - preprocessor/executor.py
    env:
      JINA_LOG_LEVEL: INFO
    uses_with:
      user_input_dict: *id001
      api_keys: *id002
      user_emails: *id003
      admin_emails: *id004
  - name: encoderclip
    uses: jinahub+docker://CLIPOnnxEncoder/0.8.1-gpu
    host: encoderclip-pretty-javelin-3aceb7f2cd.wolf.jina.ai
    port: 443
    tls: true
    external: true
    uses_with:
      access_paths: '@cc'
      name: ViT-B-32::openai
    env:
      JINA_LOG_LEVEL: DEBUG
    needs: preprocessor
  - name: indexer
    needs:
      - encoderclip
    uses:
      jtype: NOWElasticIndexer
      py_modules:
        - indexer/elastic/elastic_indexer.py
    uses_with:
      document_mappings:
        - - encoderclip
          - 512
          - - image_0
      user_input_dict: *id001
      api_keys: *id002
      user_emails: *id003
      admin_emails: *id004
    no_reduce: true
    env:
      JINA_LOG_LEVEL: INFO

```

***5 通过flow.xml 安装 jina-now 应用***

浏览器访问 http://localhost:8081/playground/ 即可访问应用
![Alt text](imgs/image.png)