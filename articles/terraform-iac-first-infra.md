---
title: "TerraformでWebアプリ公開の土台を組む(VCNからVMまで)"
emoji: "🛠️"
type: "tech"
topics: ["terraform", "iac", "oracle", "oci", "infrastructure"]
published: true
---

## 概要

- Oracle Cloud Infrastructure(OCI)上に、Terraformで「ネットワーク+VM」の土台を組めるようになる
- 実際に動かしたRAGチャットアプリをホストしているTerraform構成で解説
- 結局何を、どういう順番で作ればアプリが外から見られるようになるのかが理解る

## はじめに

この記事では、OCIを例に使います(無料枠があるので手元で試しやすいです)。ただ、ここで出てくる考え方自体はAWSやAzureなど他のクラウドでも共通です。

クラウドのWeb UIは、選択できるボタン、要素が多すぎます。VCNを作るページ、サブネットを作るページ、セキュリティルールを作るページがそれぞれ別にあって、どれを先に作ればいいのかはUIを見ただけでは分かりません。
一度作れても、半年後に同じ構成をもう一度作ろうとすると、どのページで何を押したかはだいたい忘れています。

Terraformは、この「何を、どの順番で作るか」をコードに書いて残します。書いたコードは、そのまま構成の記録になり、何度でも同じインフラを再現する手順にもなります。この記事では、実際に動いているVM1台分の構成を例に、ネットワークからVMまでを順番に組み立てていきます!

## この記事で作るもの

最終的に出来上がるのは、インターネットから接続できるVMが1台だけ、という単純な構成です。ただし、VMを1台外部公開するだけでも、裏では複数のコンポーネントを連携させる必要があります。

![Oracle Cloud上のインフラ構成図](https://raw.githubusercontent.com/bigshine777/zenn-content/master/images/terraform-oci-architecture.png)

各コンポーネントの役割は次のとおりです。

- **VCN(Virtual Cloud Network)**: ネットワークの土台になる仮想的な箱。AWSでいうVPCに相当
- **インターネットゲートウェイ**: VCNとインターネットをつなぐ出入り口
- **ルートテーブル**: 「この宛先への通信は、どこに送るか」を決めるルール群
- **セキュリティリスト**: ファイアウォールに相当し、通したい通信方法だけを許可
- **サブネット**: VCNの中で、実際にVMを置く場所。ルートテーブルとセキュリティリストを紐付けることで、どの通信が通るかを決める
- **VM(Virtual Machine)**: アプリが実際に動くサーバー本体

Web UIで同じ構成を作る場合も、裏側ではこの6つを作ることになります。違うのは、Web UIではこれを複数回の作成、紐付けで行い、Terraformではコードとして作成するという点だけです。

## なぜTerraformで組むのか

Terraformのようなツールは、IaC(Infrastructure as Code)と呼ばれます。IaCのポイントは、構成を宣言的に書くことです。

手続き的に書く場合は、作る順番と紐付けを1ステップずつ自分で指定する必要があります。Terraformでは、「VCNとインターネットゲートウェイとルートテーブルが欲しい、こう紐付いていてほしい」と欲しい状態をまとめて書くだけで済みます。どのリソースがどのリソースを参照しているかはコードの中の参照(`oci_core_vcn.this.id`のような書き方)から自動的に読み取られ、作る順番はTerraform側が解決してくれます。

Terraformは、実際に作ったリソースの情報を`terraform.tfstate`というファイルに記録します。`terraform apply`を実行すると、Terraformはこのstateファイルを読んで「今何が存在していることになっているか」を把握し、コードに書かれた「あるべき状態」と比較して、差分だけをcloudに反映します。`terraform plan`で「何が追加・変更・削除されるか」を実行前に確認できるのは、このstateとコードを比較しているからです。stateファイルにはリソースのIDなどの情報が含まれ、構成によっては機密性のある値が載ることもあるため、`.gitignore`でGitの管理対象から外しておくほうが良いです

コード自体が構成の記録にもなるので、半年後に見返しても何を作ったかをコードから追えます。

## 実際にコードで組む

ここから、実際のコードを1ファイルずつ見ていきます。使うファイルは次の5つです。

- `versions.tf`: 使うTerraform自体とproviderのバージョンを固定する
- `variables.tf`: 外から渡す値(入力パラメータ)を定義する
- `network.tf`: VCNからサブネットまでのネットワーク一式
- `compute.tf`: VM本体
- `outputs.tf`: 作成後に取り出せる値

Terraformは、同じディレクトリ内の`.tf`ファイルをすべてまとめて1つの設定として読み込みます。ファイル名そのものに特別な意味はなく、分割は人間が読みやすくするためだけのものです。

### versions.tf: 使うTerraformとproviderを固定する

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    oci = {
      source  = "oracle/oci"
      version = "~> 6.0"
    }
  }
}

provider "oci" {
  # ~/.oci/config の DEFAULT プロファイルをそのまま使う
  region = var.region
}
```

`required_providers`では、クラウドを操作するためのproviderプラグインを指定します。ここでは[Oracle Cloud Infrastructure(OCI)](https://www.oracle.com/jp/cloud/)用のproviderを使い、バージョンを`~> 6.0`(6.x系の最新)に固定しています。

`provider "oci"`のブロックには、認証情報を直接書きません。OCIの認証情報は`~/.oci/config`に置き、そのデフォルトプロファイルをそのまま使う設計にしています。認証情報をコードに書かないのは、このコード自体をそのままGitリポジトリに残しても問題にならないようにするためです(Gitに認証情報を置くのはセキュリティ的にまずい...)

### variables.tf: 外から渡す値を整理する

```hcl
variable "tenancy_ocid" {
  type = string
}

variable "region" {
  type    = string
  default = "ap-tokyo-1"
}

variable "instance_shape" {
  type    = string
  default = "VM.Standard.E2.1.Micro"
}

variable "ssh_public_key_path" {
  type    = string
  default = "~/.ssh/sample.pub"
}

variable "project_name" {
  type    = string
  default = "project-name"
}
```

`variable`ブロックは、複数回使用する変数などをまとめる場所です。ここでは2種類の変数が混在しています。

多くの変数には`default`を設定しています。たとえば`instance_shape`(VMのスペック)を変数にしておくことで、後からVMのスペックを変えたくなっても、この1箇所を書き換えるだけで済みます。

### network.tf: ネットワークの土台を組む

先ほどの図のとおり、5つのリソースを順番に定義していきます。

```hcl
resource "oci_core_vcn" "this" {
  compartment_id = var.tenancy_ocid
  cidr_block     = "10.0.0.0/16"
  display_name   = "${var.project_name}-vcn"
  dns_label      = "project-name"
}
```

`oci_core_vcn`は、VPCのリソースを表しています。`cidr_block`でこのVCN全体が使えるIPアドレスの範囲(ここでは`10.0.0.0/16`)を決めます。
10.0.0.0のアドレス帯はプライベートIPアドレスを表しており、16は後半の16ビット(0.0〜255.255)を自由に使えることを意味します。

もう1つ、全リソースに共通して出てくる`compartment_id`にも触れておきます。OCIでは作成したリソースを**コンパートメント**という論理的な単位で区切って管理し、コンパートメントごとにアクセス権限(IAMポリシー)を設定できます。今回のコードは`var.tenancy_ocid`、つまりテナント直下のルートコンパートメントをそのまま使う簡易構成にしています。

```hcl
resource "oci_core_internet_gateway" "this" {
  compartment_id = var.tenancy_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.project_name}-igw"
}
```

`oci_core_internet_gateway`は、`vcn_id`でVCNに紐付け、インターネットとの出入口を作ります。

```hcl
resource "oci_core_route_table" "this" {
  compartment_id = var.tenancy_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.project_name}-route-table"

  route_rules {
    destination       = "0.0.0.0/0"
    network_entity_id = oci_core_internet_gateway.this.id
  }
}
```

`oci_core_route_table`は、「`0.0.0.0/0`(すべての宛先)への通信は、このインターネットゲートウェイに送る」というルールです。

```hcl
resource "oci_core_security_list" "this" {
  compartment_id = var.tenancy_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.project_name}-security-list"

  egress_security_rules {
    destination = "0.0.0.0/0"
    protocol    = "all"
  }

  ingress_security_rules {
    source   = "0.0.0.0/0"
    protocol = "6" # TCP
    tcp_options {
      min = 22
      max = 22
    }
  }

  ingress_security_rules {
    source   = "0.0.0.0/0"
    protocol = "6" # TCP
    tcp_options {
      min = 443
      max = 443
    }
  }
}
```

`oci_core_security_list`がファイアウォールです。インバウンド(外部から入ってくる通信)は、22番(SSH)・443番(HTTPS)だけを許可しています。アウトバウンド(サーバーから外に出ていく通信)はすべて許可しています。許可するポートを最小限に絞ることで、外部から接続できる経路そのものを減らしています。

```hcl
resource "oci_core_subnet" "this" {
  compartment_id             = var.tenancy_ocid
  vcn_id                     = oci_core_vcn.this.id
  cidr_block                 = "10.0.0.0/24"
  display_name               = "${var.project_name}-subnet"
  dns_label                  = "public"
  route_table_id             = oci_core_route_table.this.id
  security_list_ids          = [oci_core_security_list.this.id]
  prohibit_public_ip_on_vnic = false
}
```

最後の`oci_core_subnet`が、実際にVMを置く場所です。`route_table_id`・`security_list_ids`・`vcn_id`の3つを束ねることで、「このVCN内の、このルートテーブルとセキュリティリストが適用されるサブネット」として定義されます。この「VCN→インターネットゲートウェイ→ルートテーブル→セキュリティリスト→サブネット」という組み方は、OCIに限らず、クラウドのネットワーク構築でよく見る形式です。

### compute.tf: VM本体を作る

```hcl
data "oci_identity_availability_domains" "this" {
  compartment_id = var.tenancy_ocid
}
```

`resource`は「Terraformが新しく作って管理するもの」、`data`は「すでに存在するものを検索して値を取ってくるだけ」という違いがあります。これは、VMを配置できる可用性ドメインの名前を検索しているだけです。可用性ドメインとは、同じリージョン内で電源やネットワークが独立したデータセンターの単位のことで、障害が起きたときの影響範囲を分けるために存在します。

```hcl
data "oci_core_images" "ubuntu" {
  compartment_id           = var.tenancy_ocid
  operating_system         = "Canonical Ubuntu"
  operating_system_version = "22.04"
  shape                    = var.instance_shape
  sort_by                  = "TIMECREATED"
  sort_order               = "DESC"
}
```

こちらは、使えるUbuntu 22.04イメージの中から最新のものを検索しています。

以下が本体の`resource`です。`lifecycle`と`metadata`は少し長いので、あとで分けて見ます。

```hcl
resource "oci_core_instance" "this" {
  compartment_id      = var.tenancy_ocid
  availability_domain = data.oci_identity_availability_domains.this.availability_domains[0].name
  display_name        = "${var.project_name}-server"
  shape                = var.instance_shape

  create_vnic_details {
    subnet_id        = oci_core_subnet.this.id
    assign_public_ip = true
  }

  source_details {
    source_type = "image"
    source_id   = data.oci_core_images.ubuntu.images[0].id
  }

  lifecycle {
    ignore_changes = [source_details, metadata]
  }
}
```

`create_vnic_details`の中の`subnet_id = oci_core_subnet.this.id`が、`network.tf`と`compute.tf`をつなぐ接点です。Terraformはこの参照関係から「サブネットを先に作ってから、VMを作る」という実行順序を自動的に解決します。

```hcl
  metadata = {
    ssh_authorized_keys = file(var.ssh_public_key_path)
    user_data = base64encode(<<-EOF
      #!/bin/bash
      apt-get update
      apt-get install -y python3-venv python3-pip git
      iptables -I INPUT 5 -p tcp --dport 8501 -j ACCEPT -m state --state NEW
      netfilter-persistent save
      EOF
    )
  }
```

`metadata.user_data`は、VMの初回起動時に自動で実行されるスクリプト(cloud-init)です。ここではPythonの実行環境を入れ、`iptables`でポートを1つ開けています。注意点として、OCIが提供するUbuntuイメージは、OS側の`iptables`でSSH以外の通信をデフォルトで拒否する設定になっています。つまり、セキュリティリスト(クラウド側のファイアウォール)でポートを許可しただけでは足りず、VMの内側でも同じポートを個別に開ける必要があります。

### outputs.tf: 結果を取り出す

```hcl
output "public_ip" {
  value = oci_core_instance.this.public_ip
}
```

`terraform apply`を実行した後、`terraform output`コマンドでここに書いた値を取り出せます。ここではVMのグローバルIPアドレスだけを出力し、SSH接続先の確認に使っています。

## 実行する

ファイルが揃ったら、実際にインフラを作ります。流れは次の3ステップです。

```bash
terraform init   # providerのダウンロードなど初期化
terraform plan   # 何が作成/変化するかを確認
terraform apply  # 実際に反映する
```

`apply`の前に必須の準備が2つあります。1つは、OCI CLIの認証設定です。`~/.oci/config`に、自分のテナントの認証情報を書いておく必要があります。もう1つは、`variables.tf`で`default`を持たない変数(`tenancy_ocid`)の値を、`terraform.tfvars`というファイルに書くことです。

```hcl
# terraform.tfvars
tenancy_ocid = "ocid1.tenancy.oc1..xxxxxxxx"
```

準備ができたら、`terraform plan`の出力を必ず確認してから`apply`します。ここのplanをよく見ておかないとロールバックがめんどくさいことになることもあります...

## 気をつけておきたい点: dataソースの「最新」追従

`compute.tf`の`data "oci_core_images" "ubuntu"`は、`sort_by = "TIMECREATED", sort_order = "DESC"`という条件で、常に「その時点で最新のUbuntuイメージ」を検索します。これは初回作成のときには便利な書き方ですが、一度VMを作った後に同じ設定で`terraform plan`を実行すると、別の問題が起きます。

Oracle側で新しいUbuntu 22.04イメージが公開されると、`data`ソースが返すイメージIDが変わります。すると、Terraformは「`source_details`(使っているイメージ)が変わった」と判定します。この属性はOCI provider上でForceNew(変更するには作り直しが必須)な属性なので、差分が出た瞬間に「稼働中のVMを削除して、作り直す」計画が立てられてしまいます。

これを避けるため、`compute.tf`では`lifecycle { ignore_changes = [source_details, metadata] }`を指定し、一度作成した後はイメージの変化を追従しないようにしています。`metadata`(cloud-initの内容)も同じ理由で追従対象から外しています。

## まとめ

VCNからVMまで、Webアプリを外部公開するための最小構成をTerraformのコードで組みました。ネットワーク側の5つのリソースとVM本体、合わせて実質20数行ほどのコードで、クラウドのWeb UIで何十クリックもかけてやっていたことを再現できます。

この土台の上で実際に動いているRAGチャットアプリの話は、別記事にまとめています。[学生が無料枠だけでRAGシステムを作った話](https://qiita.com/bigshine/items/b7057565da34e0e55f8d)。興味があれば、あわせて読んでみてください!
