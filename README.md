# HashiCorp Sentinel Workshop

[Sentinel](https://developer.hashicorp.com/sentinel) は HashiCorp が中心に開発をする OSS のポリシー管理フレームワークおよび記述言語です。Sentinel を利用することで、どのような状況で特定の動作を許可するかを定義することができ、Terraform、Vault、Consul、Nomad といった様々な HashiCorp Enterprise 製品と組み合わせることで Policy as Code を実現します。

Sentinel では import と呼ばれる構文を用いることで、外部ファイルやライブラリ、プラグインなどを取り込むことが可能で、再利用性が高くなるように設計されており、拡張性にも長けています。

Sentinel は、ポリシーの記述言語としての側面だけでなく、開発フローやテストといったポリシー適用に必要な機能も提供するランタイムとしての側面も持っており、
HashiCorp Enterprise 製品との親和性が高い機能を持ち合わせています。

## Pre-requisite

* 環境
	* macOS or Linux(Ubuntu 推奨)

* ソフトウェア
	* Sentinel
	* HCP Terraform

* アカウント
	* GitHub
	* AWS / Azure / GCP

## アジェンダ
* [初めての Sentinel](contents/hello-sentinel.md)
* [Sentinel 設定ファイル](contents/configurations.md)
* [Sentinel Language](contents/language-features.md)
* [Import 1: Standard imports](contents/imports-standard.md)
* [Import 2: Static imports](contents/imports-static.md)
* [Import 3: Module imports](contents/imports-modules.md)
* [Tests 1: 設定ファイル](contents/test-configuration.md)
* [Tests 2: mocking data](contents/test-mock-data.md)
* [Tests 3: mocking modules](contents/test-mock-modules.md)
* [Tests 4: mocking terraform contexts](contents/test-mock-tf.md)
* [Terraform 連携](contents/terraform-integrations.md)
* [Vault 連携](contents/vault-integrations.md)
* [Consul 連携](contents/consul-integrations.md)
* [Nomad 連携](contents/nomad-integrations.md)
* [ポリシーの開発](contents/policy-development.md)
