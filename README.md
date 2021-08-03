# local-glue
## ユニットテスト
### 環境
* java: 8.0.282.hs-adpt
* python: 3.6.0


データ準備
テーブルを用意するDynamoDB
```
aws dynamodb create-table \
    --table-name aws-glue-local-test-table \
    --attribute-definitions \
        AttributeName=Id,AttributeType=S \
    --key-schema AttributeName=Id,KeyType=HASH \
    --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1 \
    --endpoint-url http://localhost:4566
```

テストデータ用意
```
aws dynamodb put-item \
    --table-name aws-glue-local-test-table  \
    --item \
        '{"Id": {"S": "test"}, "Column1": {"S": "test1"}, "Column2": {"S": "test2"}, "Column3": {"S": "test3"}}' \
    --endpoint-url http://localhost:4566
```

出力用のS3バケット用意
```
aws s3api create-bucket \
    --bucket aws-glue-local-test-bucket \
    --endpoint-url http://localhost:4566
```



aws-glue-libsの環境を構築するために下記手順を実行していきます。
https://github.com/awslabs/aws-glue-libs
   
1-1. Apache Maven をダウンロードします。
```
mkdir opt
cd opt

curl -O https://aws-glue-etl-artifacts.s3.amazonaws.com/glue-common/apache-maven-3.6.0-bin.tar.gz
tar xzvf apache-maven-3.6.0-bin.tar.gz
```

1-2. Apache MavenのPATHを利用しているシェルに通します。
(例) .zshrcの場合(.bashrcも同様)
```
export M3_HOME=$HOME/opt/apache-maven-3.6.0
M3=$M3_HOME/bin
export PATH=$M3:$PATH
```


2-1. sparkをダウンロードします。
```
# 1-1で作成したoptディレクトリに移動
cd opt
curl -O https://aws-glue-etl-artifacts.s3.amazonaws.com/glue-2.0/spark-2.4.3-bin-hadoop2.8.tgz
tar -xvzf spark-2.4.3-bin-hadoop2.8.tgz
# sparkにディレクトリ名を変更(別にしなくてもいい）
mv spark-2.4.3-bin-spark-2.4.3-bin-hadoop2.8 spark
```

2-2. sparkのPATHを利用しているシェルに通します。
(例) .zshrcの場合(.bashrcも同様)
```
# spark
export SPARK_HOME=/opt/spark
export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin
```

3-1 aws-glue-libsをクローンします。(重要: glue-1.0ブランチを使うこと！)
```
# 1-1で作成したoptディレクトリに移動
cd opt
git clone https://github.com/awslabs/aws-glue-libs.git
git checkout glue-1.0
```

3-2. aws-glue-libsの下記コマンドが実行できるか確認します。
```
# Glue submit:
~/opt/aws-glue-libs/bin/gluesparksubmit
```

4-1. pythonのバージョンを3.6.0に変更します。
(例: pyenvの場合)
```
pyenv install 3.6.0
pyenv global 3.6.0
```

5-1. spark.confに以下の設定を追加します。
```
cd ~/opt/aws-glue-libs
vim bin/glue-setup.sh
```

追加する行
```
echo "spark.driver.extraClassPath $GLUE_JARS_DIR/*" >> $SPARK_CONF_DIR/spark-defaults.conf
echo "spark.executor.extraClassPath $GLUE_JARS_DIR/*" >> $SPARK_CONF_DIR/spark-defaults.conf

# ここから追記
echo "spark.hadoop.dynamodb.endpoint http://localhost:4566" >> $SPARK_CONF_DIR/spark-defaults.conf
echo "spark.hadoop.fs.s3a.endpoint http://localhost:4566" >> $SPARK_CONF_DIR/spark-defaults.conf
echo "spark.hadoop.fs.s3a.path.style.access true" >> $SPARK_CONF_DIR/spark-defaults.conf
echo "spark.hadoop.fs.s3a.signing-algorithm S3SignerType" >> $SPARK_CONF_DIR/spark-defaults.conf
```

6-1. ダミーのcredentialを設定
```
export AWS_ACCESS_KEY_ID='dummy'
export AWS_SECRET_ACCESS_KEY='dummy'
export AWS_REGION='ap-northeast-1'
```

6-2. シェルの再起動
```
exec $SHELL -l
```

7-1. pythonファイルの実行
(例)
```
./bin/gluesparksubmit <実行するファイル> --JOB_NAME='dummy'
./bin/gluesparksubmit src/sample.py --JOB_NAME='dummy'
```

7-2. 確認
```
aws s3api list-objects --bucket aws-glue-local-test-bucket --endpoint-url http://localhost:4566
aws s3 cp s3://aws-glue-local-test-bucket ~/Downloads/aws-glue-local-test-bucket --recursive --endpoint-url http://localhost:4566

※ aws-cliのパスが通ってない場合
/usr/local/aws-cli
./aws s3api list-objects --bucket aws-glue-local-test-bucket --endpoint-url http://localhost:4566
./aws s3 cp s3://aws-glue-local-test-bucket ~/Downloads/aws-glue-local-test-bucket --recursive --endpoint-url http://localhost:4566
```

ーーーーーーーーーーーーーー

# Glueメモ

1最大容量 = 1worker = (G.1X: 1executor,  G.2X は2DPUにマッピングされているので2executor?)
設定値みてもパッとみあってそう・・・？

https://docs.aws.amazon.com/ja_jp/glue/latest/dg/aws-glue-api-jobs-job.html
WorkerType - UTF-8 文字列 (有効な値: Standard="" | G.1X="" | G.2X="")。

ジョブの実行時に割り当てられる事前定義済みのワーカーの種類。使用できる値は、Standard、G.1X、または G.2X です。
Standard ワーカータイプでは、各ワーカーは 4 vCPU、16 GB のメモリ、50 GB のディスク、ワーカーあたり 2 個のエグゼキュターを提供します。
G.1X ワーカータイプでは、各ワーカーは 1 DPU (4 vCPU、16 GB のメモリ、64 GB のディスク) にマッピングされており、ワーカーごとに 1 個のエグゼキュターを提供します。メモリを大量に消費するジョブには、このワーカータイプをお勧めします。
G.2X ワーカータイプでは、各ワーカーは 2 DPU (8 vCPU、32 GB のメモリ、128 GB のディスク) にマッピングされており、ワーカーごとに 1 個のエグゼキュターを提供します。メモリを大量に消費するジョブには、このワーカータイプをお勧めします
8月4日 7:59
西　悠之[水以外在宅]
西　悠之[水以外在宅]
1executorで実行されるタスク数は決められているので
groupSize は、各ファイルから読み取り、個々の Spark タスクで処理するデータの量を設定できるオプションのフィールドなので
変にタスクの並列数が増えたとしても、1タスクで処理するデータ量を抑えてやればOOMは抑えていけそう？
→ 運用中にOOMが発生してもジョブパラメータなので最悪試行錯誤できる
https://aws.amazon.com/jp/blogs/news/optimize-memory-management-in-aws-glue/
