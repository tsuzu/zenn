---
title: "Kubernetes 1.33: SIG-Instrumentationの変更内容" # 記事のタイトル
emoji: "☸️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["kubernetes"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

# はじめに
本記事では Kubernetes v1.33 の [Changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.33.md) から、SIG-Instrumentation 関連及びメトリクスの変更点について取り上げ、まとめています。

* [Kubernetes 1.33: 変更点まとめリンク集](https://qiita.com/uesyn/items/7b882d147847ab8b02f8)
* [Kubernetes 1.31 SIG Instrumentation の変更内容](https://qiita.com/watawuwu/items/ae739b0a2085bd16c0f9)
* [Kubernetes 1.32 SIG Instrumentation の変更内容](https://qiita.com/watawuwu/items/5280af2088bbc1a86aca)

:::message
メトリクス名の横に絵文字で変更の種類を記載しています。

* 🆕: 新規で追加されたメトリクス
* 🆙: ラベルなどに更新があったメトリクス
* ⚠️: 非推奨になったメトリクス
* 🆑: 削除になったメトリクス
:::

# Changes by kind
## Urgent Upgrade Notes
* SIG/Instrumentationに関連する項目はありませんでした
## Deprecation
* SIG/Instrumentationに関連する項目はありませんでした

## API Changes
* `/statusz` endpoint (件数が多いためまとめて紹介します)
	* PRs
		* kube-proxyに `/statusz` endpointが追加されました ([#128989](https://github.com/kubernetes/kubernetes/pull/128989), [@Henrywu573](https://github.com/Henrywu573))
		* kube-schedulerに `/statusz` endpointが追加されました ([#128818](https://github.com/kubernetes/kubernetes/pull/128818), [@yongruilin](https://github.com/yongruilin))
		* kubeletに `/statusz` endpointが追加されました ([#128811](https://github.com/kubernetes/kubernetes/pull/128811), [@zhifei92](https://github.com/zhifei92))
		* kube-controller-managerに `/statusz` endpointが追加されました ([#128991](https://github.com/kubernetes/kubernetes/pull/128991), [@Henrywu573](https://github.com/Henrywu573))
		* kube-schedulerに `/statusz` endpointが追加されています ([#128987](https://github.com/kubernetes/kubernetes/pull/128987), [@Henrywu573](https://github.com/Henrywu573))
	* [KEP-4827: Component Statusz](https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/4827-component-statusz) にて提案されている機能で、kube-scheduler、kubeletといった各Kubernetesのcomponentに `/statusz` エンドポイントを追加し、ビルドバージョン、Go のバージョンといった情報を返すようになりました。v1.33ではkube-apiserver以外でのサポートが追加されています。
		* 詳細は [Kubernetes 1.32 SIG Instrumentation の変更内容](https://qiita.com/watawuwu/items/5280af2088bbc1a86aca#kep-4827-component-statusz)で紹介されています。
		* Kubernetes v1.33 でも`ComponentStatusz` Feature Gateは引き続きalphaでデフォルトは無効化されています。

* `/flagz` endpoint (件数が多いためまとめて紹介します)
	* PRs
		* kubeletに `/flagz` endpointが追加されました ([#128857](https://github.com/kubernetes/kubernetes/pull/128857), [@zhifei92](https://github.com/zhifei92))
		* kube-proxyに `/flagz` endpointが追加されました ([#128985](https://github.com/kubernetes/kubernetes/pull/128985), [@yongruilin](https://github.com/yongruilin))
		* kube-controller-managerに `/flagz` endpointが追加されました ([#128824](https://github.com/kubernetes/kubernetes/pull/128824), [@yongruilin](https://github.com/yongruilin))
		* kube-apiserverが `/flagz` endpointで正しくパース済みのflagを返すようになりました ([#130328](https://github.com/kubernetes/kubernetes/pull/130328), [@richabanker](https://github.com/richabanker)) / ([#129996](https://github.com/kubernetes/kubernetes/pull/129996), [@yongruilin](https://github.com/yongruilin)) 
	*  [KEP-4828: Component Flagz](https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/4828-component-flagz) にて提案されている機能で、KEP-4827の様にkube-scheduler、kubeletといった各Kubernetesのcomponentに `/flagz` エンドポイントを追加し、コマンドラインのフラグといった情報を返すようになりました。v1.33ではkube-apiserver以外でのサポートが追加されています。
		* 詳細は [Kubernetes 1.32 SIG Instrumentation の変更内容](https://qiita.com/watawuwu/items/5280af2088bbc1a86aca#kep-4828-component-flagz)で紹介されています。
		* Kubernetes v1.33 でも`ComponentFlagz` Feature Gateは引き続きalphaでデフォルトは無効化されています。
	* 📝 v1.32からフォーマットが変更されています。feature-gatesの値が表示されない問題があったようですが、修正されています。

```diff
kube-apiserver flags
Warning: This endpoint is not meant to be machine parseable, has no formatting compatibility guarantees and is for debugging purposes only.

- admission-control []
+ admission-control=[]
- admission-control-config-file 
+ admission-control-config-file=
- advertise-address 172.18.0.2
+ advertise-address=172.18.0.2
...
- feature-gates 
+ feature-gates=:ComponentFlagz=true,:ComponentStatusz=true,:KubeletInUserNamespace=true
...
```

* Device Resource Allocation(DRA)において、デバイスごとのtaints/tolerationを付与する機能が導入されました。これに際して2つのメトリクスが導入されています。 ([#130447](https://github.com/kubernetes/kubernetes/pull/130447), [@pohly](https://github.com/pohly))
	* 🆕 `device_taint_eviction_controller_pod_deletions_total`
		* taintによってevictされたPodの総数を返します。
	* 🆕 `device_taint_eviction_controller_pod_deletion_duration_seconds_*`
		* taintが付与されてからPodがeviction controllerによって削除されるまでの時間を表すbucketです。
* ReplicationController の `replicas` と `minReadySeconds` フィールドの最小値バリデーションが宣言的バリデーションに移行しました。従来の方式とvalidation結果が異なったり問題があった際にメトリクスが報告されます。 ([#130725](https://github.com/kubernetes/kubernetes/pull/130725), [@jpbetz](https://github.com/jpbetz))
	* 1.33からDeclarativeValidation Feature Gateがbetaとして導入され、デフォルトで有効化されています。validation自体は実施されますが、同様に1.33から導入されたDeclarativeValidationTakeover Feature Gateを有効化しない限り、宣言的バリデーションの結果は適用されません。こちらはデフォルトで無効化されています。
	* 🆕 `declarative_validation_mismatch_total`
		* mismatchが発生した場合に報告されるメトリクスです。
	* 🆕 `declarative_validation_mismatch_panic_total`
		* validationでpanicが発生した場合に報告されるメトリクスです。


## Features
* API serverのwatch cacheとetcdのデータに差異があるかを5分おきにチェックし、メトリクスとして報告されます([#130475](https://github.com/kubernetes/kubernetes/pull/130475), [@serathius](https://github.com/serathius), KEP‑4988)
	* 🆕 `apiserver_storage_consistency_checks_total`
		* labelとしてsuccess/failure/errorがあります。
	* [`hash/fnv`](https://pkg.go.dev/hash/fnv) を利用してハッシュが計算されています。
	* 現在ではListFromCacheSnapshot Feature Gateはalphaです。
		* beta以降では自動的にキャッシュをバイパスするようになるそうです。
* kube-apiserverのtracing機能で、認証済みなリクエストに対してヘッダで渡された既存のtrace idを引き継げるようになりました ([#127053](https://github.com/kubernetes/kubernetes/pull/127053), [@dashpole](https://github.com/dashpole))
	* abuse防止のため、`system:master` または `system:monitoring` groupに所属しているユーザのみ利用できます。

## Documentation
* SIG/Instrumentationに関連する項目はありませんでした

## Bug or Regression
* SIG/Instrumentationに関連する項目はありませんでした

## Other (Cleanup or Flake)
* `pod_scheduling_duration_seconds` メトリクスが削除されました。代わりに `pod_scheduling_sli_duration_seconds` を使う必要があります。 ([#128906](https://github.com/kubernetes/kubernetes/pull/128906), [@sanposhiho](https://github.com/sanposhiho))
    * 🆑 `pod_scheduling_duration_seconds`
    	* v1.28からdeprecateされており、v1.33にて削除されました。
	* `pod_scheduling_duration_seconds` は、KEP-3521にて[Scheduling Gate](https://kubernetes.io/ja/docs/concepts/scheduling-eviction/pod-scheduling-readiness/)に関連して追加されたPreEnqueueという拡張点における時間を含んでおりメトリクスとして不適当でした。

## Metrics Changes
:::message
SIG Instrumentationではないが、メトリクスに変更があるものについて記載しています。
:::

* kubeletのリソースアラインメントエラーの主な原因を公開するメトリクスが追加されました。 ([#129950](https://github.com/kubernetes/kubernetes/pull/129950), [@ffromani](https://github.com/ffromani))
	* 🆕 `kubelet_container_aligned_compute_resources_failure_count`
* NUMA nodeごとのCPUの利用数を公開するメトリクスが追加されました ([#130491](https://github.com/kubernetes/kubernetes/pull/130491), [@swatisehgal](https://github.com/swatisehgal))
	* 🆕 `kubelet_cpu_manager_allocation_per_numa`
* image volumeのmountに関するメトリクスが追加されました ([#130135](https://github.com/kubernetes/kubernetes/pull/130135), [@saschagrunert](https://github.com/saschagrunert))
	* 🆕 `kubelet_image_volume_requested_total`
	* 🆕 `kubelet_image_volume_mounted_succeed_total`
	* 🆕 `kubelet_image_volume_mounted_errors_total`
* kube-proxyのメトリクスにIP familyのlabelが追加されました ([#129173](https://github.com/kubernetes/kubernetes/pull/129173), [@aroradaman](https://github.com/aroradaman))
	* 🆙 `kubeproxy_network_programming_duration_seconds`
	* 🆙 `kubeproxy_proxy_healthz_total`
	* 🆙 `kubeproxy_sync_partial_proxy_rules_duration_seconds`
	* 🆙 `kubeproxy_sync_proxy_rules_duration_seconds`
	* 🆙 `kubeproxy_sync_proxy_rules_endpoint_changes_pending`
	* 🆙 `kubeproxy_sync_proxy_rules_iptables_partial_restore_failures_total`
	* 🆙 `kubeproxy_sync_proxy_rules_iptables_restore_failures_total`
	* 🆙 `kubeproxy_sync_proxy_rules_iptables_total`
	* 🆙 `kubeproxy_sync_proxy_rules_last_queued_timestamp_seconds`
	* 🆙 `kubeproxy_sync_proxy_rules_last_timestamp_seconds`
	* 🆙 `kubeproxy_sync_proxy_rules_nftables_cleanup_failures_total`
	* 🆙 `kubeproxy_sync_proxy_rules_nftables_sync_failures_total`
	* 🆙 `kubeproxy_sync_proxy_rules_no_local_endpoints_total`
* kube-proxyが削除したconntrack flowの累積値を取得するメトリクスが追加されました ([#130204](https://github.com/kubernetes/kubernetes/pull/130204), [@aroradaman](https://github.com/aroradaman))
	* 🆕 `kubeproxy_conntrack_reconciler_deleted_entries_total`
    * こちらもIP familyのlabelが導入されています
* kube-proxyのconttrackのreconcileにかかった時間を取得できるメトリクスが追加されました ([#130200](https://github.com/kubernetes/kubernetes/pull/130200), [@aroradaman](https://github.com/aroradaman))
	* 🆕 `kubeproxy_conntrack_reconciler_sync_duration_seconds`
    * こちらもIP familyのlabelが導入されています
* `scheduler_cache_size` メトリクスが導入され、代わりに `scheduler_scheduler_cache_size` メトリクスはdeprecateされました ([#128810](https://github.com/kubernetes/kubernetes/pull/128810), [@googs1025](https://github.com/googs1025))
	* 🆕 `scheduler_cache_size`
	* ⚠️ `scheduler_scheduler_cache_size`
        * v1.34で削除される予定です。
