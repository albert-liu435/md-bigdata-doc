## es windows安装

下载

1、下载elasticsearch-7.13.2-windows-x86_64.zip的zip包，并解压到指定目录，下载地址：

https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.13.2-windows-x86_64.zip

![015d1924bdc45c36a01a456be004c3e](.\pic\015d1924bdc45c36a01a456be004c3e.png)

2、安装中文分词插件，在elasticsearch-7.13.2\bin目录下执行以下命令：elasticsearch-plugin install

https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.13.2/elasticsearch-analysis-ik-7.13.2.zip

具体参考：https://github.com/medcl/elasticsearch-analysis-ik

![651434efa58a0caf205cad0ab032537](.\pic\651434efa58a0caf205cad0ab032537.png)

3、运行bin目录下的elasticsearch.bat启动Elasticsearch

访问http://localhost:

4、下载Kibana,作为访问Elasticsearch的客户端,下载地址为：

https://www.elastic.co/cn/downloads/kibana

运行bin下面的kibana.bat，启动Kibana的用户界面

5、访问http://localhost:5601 即可打开Kibana的用户界面