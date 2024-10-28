# Slurm with Lustre (DDN EXAScaler) の構築手順

本手順では、Cluster Toolkit を利用して A3 Mega VM をノードに利用する Slurm クラスタ、および Lustre ファイル システム (DDN EXAScaler) のデプロイ方法について説明します。

## 実施内容
本手順での具体的な作業は以下のとおりです。

1. Cluster Toolkit の準備
2. デプロイ用 Blueprint ファイルの作成
3. クラスタのデプロイ
4. テスト ワークロードの実行
5. 実際のジョブの実行

必要に応じて、本手順実施前に A3 Mega VM の CUD 購入、および予約の作成や CUD の共有設定を有効化してください。  
本手順では CUD や予約関連の設定手順の説明は割愛します。  

## 1. Cluster Toolkit の準備
[Cluster Toolkit](https://github.com/GoogleCloudPlatform/cluster-toolkit) は Google Cloud 上で AI/ML や HPC ワークロードに利用するクラスタのデプロイを簡単にするための OSS です。  
Cluster Toolkit では Slurm クラスタや Google Kubernetes Engine (GKE) などのクラスタがデプロイできますが、今回は Slurm クラスタを作成します。  
また、今回は Lustre ファイル システムとして Google Cloud の Marketplace 上で利用可能な [DDN EXAScaler](https://www.ddn.com/ddn-cloud-services/) を利用します。

ただし、最新の Cluster Toolkit はクライアント OS として Debian を利用している一方で DDN EXAScaler はクライアント OS として RHEL、および Ubuntu をサポートします。  
そのため、本手順では Cluster Toolkit の v.1.39.0 を利用し、Slurm クラスタの OS を DDN EXAScaper がサポートする Ubuntu へ変更します。

### 1-1. 環境準備
Cluster Toolkit は Terraform や Packer などがインストールされていることを前提とします。  
そのため、[こちら](https://cloud.google.com/cluster-toolkit/docs/setup/install-dependencies)のドキュメントに記載されているモジュールをインストールします。  
その上で、以下のコマンドで Cluster Toolkit の v1.39.0 をローカル環境にクローンします。

```bash
git clone https://github.com/GoogleCloudPlatform/cluster-toolkit -b v1.39.0
```

### 1-2. Cluster Toolkit のビルド
以下のコマンドで Cluster Toolkit をビルドします。

```bash
cd cluster-toolkit
make
```

### 1-3. Google Cloud アカウントでのログイン
Cluster Toolkit では Terraform で Google Cloud リソースをデプロイします。  
リソースのデプロイには権限が必要になるため、以下のコマンドでリソースのデプロイに利用するアカウントにログインします。

```bash
gcloud auth application-default login
```

## 2. デプロイ用 Blueprint ファイルの作成
Cluster Toolkit 配下の examples/machine-learning/a3-megagpu-8g ディレクトリに A3 Mega VM をノードに利用する Slurm クラスタをデプロイするための Blueprint が含まれます。  
デフォルトではそれぞれ役割の異なる以下の 3 つの blueprint ファイルと、変数を定義する 2 つのファイル (deployment-base.yaml と deployment-image-cluster.yaml) から構成されています。

- slurm-a3mega-base.yaml
    - GPU 以外の通信で利用される VPC、およびネットワーク ストレージの作成
- slurm-a3mega-image.yaml
    - Slurm クラスタで利用するマスター イメージの作成 (ドライバのインストールなどを実施した後にカスタム イメージを作成)
- slurm-a3mega-cluster.yaml
    - Slurm クラスタの作成

デフォルトではネットワーク ストレージとして Filestore を利用します。  
本手順ではネットワーク ストレージとして Lustre を利用するために Blueprint もそれにあわせた変更が必要です。  
利用可能な Blueprint を blueprint ディレクトリ配下に格納しておりますが、以下でそれぞれについて解説します。  
deployment-base.yaml と deployment-image-cluster.yaml についてはファイル内に変更が必要な箇所を記載しています。

### 2-1. slurm-a3mega-base-lustre.yaml
VPC とネットワーク ストレージ (DDN EXASCaler) をデプロイするための Blueprint です。  
DDN EXAScaler のデフォルト値は[こちら](https://github.com/DDNStorage/exascaler-cloud-terraform/tree/scripts/2.1.8/gcp)となりますが、イメージ バージョンや MDS、MGS、OSS のインスタンス サイズやそれぞれのディスクなどは適宜ご変更ください。(instance_image はサンプルのとおり **exascaler-cloud-6-3-rocky-linux-8-optimized-gcp** を利用ください)

実行後に DDN EXAScaler に関する情報が出力されますが、server_ip に記載されている IP アドレスを deployment-image-cluster.yaml の server_ip_lustrefs に記載ください。

なお、今回のサンプルでは /home は Controller にアタッチされたディスクの領域を利用します。  
DDN EXAScaler のローカル マウント パスを /home に設定することで /home が DDN EXAScaler の領域を利用するよう設定が可能です。

ただし、DDN EXAScaler では複数のローカル マウント パスを設定することができません。(/home と /data を共に DDN EXAScaler にマウントするなど)  
そのため、例えば /home を DDN EXAScaler のマウント先に指定し、/home 配下の別ディレクトリ (/home/data など) をデータの保存先に指定するなどの対応が可能です。

### 2-2. slurm-a3mega-image-lustre.yaml
Packer を利用してマスター イメージを作成するための Blueprint で、基本的には変更不要です。  
追加のモジュールのインストールなどを行う場合には Shell スクリプトを実装いただくことでカスタムの処理が可能です。

### 2-3. slurm-a3mega-cluster-lustre.yaml
Slurm クラスタをデプロイするための Blueprint で、基本的に変更不要です。  
データを格納している GCS バケットを GCS FUSE でマウントすることも可能であり、その場合は 54 ~ 61 行目、および 246 行目を変更ください。(デフォルトではコメントアウトされています)

## 3. クラスタのデプロイ
Cluster Toolkit では gcluster deploy コマンドの引数に Blueprint ファイルを指定することでリソースをデプロイします。  
今回は Slurm クラスタのデプロイに加え、Prolog スクリプトを Ubuntu 用に変更します。

### 3-1. Slurm クラスタのデプロイ
以下の 3 つのコマンドを順に実行することで Slurm クラスタをデプロイします。

```bash
./gcluster deploy -w -d deployment-base.yaml slurm-a3mega-base-lustre.yaml --auto-approve
./gcluster deploy -w -d deployment-image-cluster.yaml slurm-a3mega-image-lustre.yaml --auto-approve
./gcluster deploy -w -d deployment-image-cluster.yaml slurm-a3mega-cluster-lustre.yaml --auto-approve
```

### 3-2. Prolog スクリプトの変更
A3 Mega VM では GPUDirect-TCPXO というネットワーク スタックを利用して他ノードとの GPU 間通信を高速化します。  
GPUDirect-TCPXO を利用する場合は Google Cloud の NCCL Plugin のインストール、および Receive Data Path Manager というコンテナの起動が必要となりますが、これらの処理は Slurm の Prolog / Epilog スクリプトで自動的に実行されます。

このスクリプトは /opt/apps/adm/slurm/scripts 配下に 2 ファイル (sudo-oslogin と rxdm) 保存されておりますが、rxdm の Prolog が Debian 用の実装になっているために 52 行目を変更します。

```bash
// 変更前
if (grep -q "ID=debian" /etc/os-release && lsb_release -rs | grep -q "12"); then

// 変更後
if (grep -q "ID=ubuntu" /etc/os-release && lsb_release -rs | grep -q "22.04"); then
```

> [!NOTE]
> NCCL Plugin のインストール、および Receive Data Path Manager コンテナの起動は Slurm ジョブが 2 ノード以上で動作する場合のみ実行されます。  
> 初回実行時にはコンテナのダウンロードなどが行われるため、ジョブの開始までに数分程度要する場合があります。

## 4. テスト ワークロードの実行
ここまでで Slurm クラスタのデプロイが完了したため、NCCL のベンチマーク テストを実行します。  
テスト手順は examples/machine-learning/a3-megagpu-8g/nccl-tests 配下の README に記載されており、以下の 3 つのコマンドを実行します。  
なお、手順内の build-nccl-tests.sh と run-nccl-tests.sh も同ディレクトリに保存されておりますので適宜 Slurm クラスタ内にファイルを作成ください。

```bash
enroot import docker://nvcr.io#nvidia/pytorch:24.04-py3
sbatch build-nccl-tests.sh
sbatch run-nccl-tests.sh
```

正常に実行された場合、NCCL のベンチマーク テストの結果は以下のようになります。(タイムゾーンの Warning が出る場合がありますが、GPUDirect-TCPXO の動作に影響はございません)  
busbw の値が以下の例よりも極端に低い場合は GPUDirect-TCPXO が正常に稼働していないため、設定の見直しが必要な可能性があります。

    #
    #                                                              out-of-place                       in-place          
    #       size         count      type   redop    root     time   algbw   busbw #wrong     time   algbw   busbw #wrong
    #        (B)    (elements)                               (us)  (GB/s)  (GB/s)            (us)  (GB/s)  (GB/s)       
         8388608        131072     float    none      -1    349.2   24.02   22.52    N/A    347.0   24.18   22.67    N/A
        16777216        262144     float    none      -1    400.8   41.86   39.24    N/A    393.6   42.63   39.96    N/A
        33554432        524288     float    none      -1    444.0   75.57   70.85    N/A    443.6   75.65   70.92    N/A
        67108864       1048576     float    none      -1    726.4   92.39   86.62    N/A    728.5   92.12   86.37    N/A
       134217728       2097152     float    none      -1   1120.9  119.74  112.26    N/A   1152.3  116.48  109.20    N/A
       268435456       4194304     float    none      -1   1765.3  152.06  142.56    N/A   1764.5  152.13  142.63    N/A
       536870912       8388608     float    none      -1   2876.0  186.68  175.01    N/A   2874.7  186.76  175.09    N/A
      1073741824      16777216     float    none      -1   5539.6  193.83  181.72    N/A   5583.2  192.32  180.30    N/A
      2147483648      33554432     float    none      -1    10863  197.69  185.34    N/A    10845  198.02  185.64    N/A
      4294967296      67108864     float    none      -1    21511  199.67  187.19    N/A    21493  199.83  187.34    N/A
      8589934592     134217728     float    none      -1    42783  200.78  188.23    N/A    42777  200.81  188.26    N/A
    # Out of bounds values : 0 OK
    # Avg bus bandwidth    : 126.358 
    #

## 5. 実際のジョブの実行
Cluster Toolkit でデプロイした Slurm クラスタでジョブを実行する場合、以下の設定が必要となります。

- Enroot / Pyxis で実行するコンテナのインポート (Slurm クラスタでインポート)
- GPUDirect-TCPXO 用の環境変数の設定 (コンテナへ設定)
- LD_LIBRARY_PATH の設定 (コンテナへ設定)
- NCCL Plugin がインストールされたパスのマウント (コンテナ起動時にマウント)

コンテナのインポートにつきましては、先の手順で実行した `enroot import` コマンドで実施可能です。  
GPUDirect-TCPXO 用の環境変数については /var/lib/tcpxo/lib64/nccl-env-profile.sh を利用した一括設定が可能です。  
LD_LIBRARY_PATH の設定と併せて run-nccl-tests.sh を参考に以下をジョブのスクリプト内に記載ください。

```bash
NCCL_LIB_DIR="/var/lib/tcpxo/lib64" source /var/lib/tcpxo/lib64/nccl-env-profile.sh
export NCCL_FASTRAK_CTRL_DEV=enp0s12
export NCCL_FASTRAK_IFNAME=enp6s0,enp7s0,enp13s0,enp14s0,enp134s0,enp135s0,enp141s0,enp142s0
export NCCL_SOCKET_IFNAME=enp0s12
export NCCL_FASTRAK_LLCM_DEVICE_DIRECTORY=/dev/aperture_devices
export LD_LIBRARY_PATH=/var/lib/tcpxo/lib64:/usr/lib/x86_64-linux-gnu:\$LD_LIBRARY_PATH;
```

NCCL Plugin がインストールされたパス (/var/lib/tcpxo/lib64) はジョブの実行時に以下のように引数を設定してマウントください。

```bash
srun --container-image "コンテナ イメージへのパス" --container-mounts "/var/lib/tcpxo/lib64:/var/lib/tcpxo/lib64","その他マウントが必要なパス"
```

> [!NOTE]
> 上記の通り当該インスタンスで初めてジョブを実行し、Prolog が実行されたタイミングでコンテナのダウンロードやモジュールのインストールが行われます。  
> そのため、NCCL のベンチマーク テストで利用されたインスタンス以外ではジョブの実行開始までに時間を要する場合があります。  
> 例えばあらかじめ全ノードを指定して簡単なジョブを実行しておくとモジュールのインストールなどが完了しております。