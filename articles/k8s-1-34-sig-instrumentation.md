---
title: "Kubernetes 1.34: SIG-Instrumentationの変更内容" # 記事のタイトル
emoji: "☸️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["kubernetes"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

# はじめに
本記事では Kubernetes v1.34 の [Changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.34.md) から、SIG-Instrumentation 関連及びメトリクスの変更点について取り上げ、まとめています。

* [Kubernetes 1.31 SIG Instrumentation の変更内容](https://qiita.com/watawuwu/items/ae739b0a2085bd16c0f9)
* [Kubernetes 1.32 SIG Instrumentation の変更内容](https://qiita.com/watawuwu/items/5280af2088bbc1a86aca)
* [Kubernetes 1.33: SIG-Instrumentationの変更内容](https://zenn.dev/tsuzu/articles/k8s-1-33-sig-instrumentation)

:::message
メトリクス名の横に絵文字で変更の種類を記載しています。

* 🆕: 新規で追加されたメトリクス
* 🆙: ラベルなどに更新があったメトリクス
* ⚠️: 非推奨になったメトリクス
* 🆑: 削除になったメトリクス
:::

# Changes by kind
## Upgrade Notes
* APIサーバーのメトリクスにおいて、API groupを抽出して新しい `group` ラベルとして追加されました ([#131845](https://github.com/kubernetes/kubernetes/pull/131845), [@serathius](https://github.com/serathius))
  * 🆙 対象メトリクス:
  	 * `apiserver_request_body_size_bytes`
  	 * `apiserver_storage_events_received_total`
  	 * `apiserver_storage_list_evaluated_objects_total`
  	 * `apiserver_storage_list_fetched_objects_total`
  	 * `apiserver_storage_list_returned_objects_total`
  	 * `apiserver_storage_list_total`
  	 * `apiserver_watch_cache_events_dispatched_total`
  	 * `apiserver_watch_cache_events_received_total`
  	 * `apiserver_watch_cache_initializations_total`
  	 * `apiserver_watch_cache_resource_version`
  	 * `watch_cache_capacity`
  	 * `apiserver_init_events_total`
  	 * `apiserver_terminated_watchers_total`
  	 * `watch_cache_capacity_increase_total`
  	 * `watch_cache_capacity_decrease_total`
  	 * `apiserver_watch_cache_read_wait_seconds`
  	 * `apiserver_watch_cache_consistent_read_total`
  	 * `apiserver_storage_consistency_checks_total`
  	 * `etcd_bookmark_counts`
  	 * `storage_decode_errors_total`
	 * `apiserver_cache_list_fetched_objects_total`
	 * `apiserver_cache_list_returned_objects_total`
	 * `apiserver_cache_list_total`
	 * `etcd_request_duration_seconds`
	 * `etcd_requests_total`
	 * `etcd_request_errors_total`
	 * `apiserver_selfrequest_total`
  * リソースを表すlabelが `group` `resource` に統一されました

## Deprecation
* ⚠️ `apiserver_storage_objects` が非推奨となり、`apiserver_resource_objects` メトリクスで置き換えられました ([#132965](https://github.com/kubernetes/kubernetes/pull/132965), [@serathius](https://github.com/serathius))
  * 他のメトリクスと一致するラベルを使用します
  * `resource` → `group` `resource`

## API Changes
* kube-log-runner にログファイルローテーション機能が追加されました ([#127667](https://github.com/kubernetes/kubernetes/pull/127667), [@zylxjtu](https://github.com/zylxjtu))
  * https://kubernetes.io/docs/concepts/cluster-administration/system-logs/#klog
	* klogは常にstderrにログを出力するのでそれをstdoutやファイルにリダイレクトするためのコマンド
  * `-log-file-size` パラメータでログファイルサイズに基づくローテーション
  * `-log-file-age` パラメータで古いログファイルの自動削除
  * `-flush-interval` パラメータで定期的なフラッシュサポート

* `KubeletTracing` feature gate がGAに昇格しました ([#132341](https://github.com/kubernetes/kubernetes/pull/132341), [@dashpole](https://github.com/dashpole))
	* https://github.com/kubernetes/enhancements/issues/2831
		* 1.25: alpha
		* 1.27: beta
		* 1.34: stable

## Features
* 🆕 APIサーバーに `apiserver_resource_size_estimate_bytes` メトリクスが追加されました ([#132893](https://github.com/kubernetes/kubernetes/pull/132893), [@serathius](https://github.com/serathius))
	* https://github.com/kubernetes/kubernetes/issues/132233

* 🆕 compatibility versioning のアルファメトリクスが追加されました ([#131842](https://github.com/kubernetes/kubernetes/pull/131842), [@michaelasp](https://github.com/michaelasp))

* 🆕 `DetectCacheInconsistency` feature gate が追加され、APIサーバーがキャッシュとetcd間の整合性を定期的に検証できるようになりました ([#132884](https://github.com/kubernetes/kubernetes/pull/132884), [@serathius](https://github.com/serathius))
  * 不整合は `apiserver_storage_consistency_checks_total` メトリクスで報告されます
  * 影響を受けたキャッシュスナップショットのパージがトリガーされます

* 🆕 kubeletに `dra_resource_claims_in_use` メトリクスが追加されました ([#131641](https://github.com/kubernetes/kubernetes/pull/131641), [@pohly](https://github.com/pohly))
  * アクティブな `ResourceClaims` を全体およびドライバー別に報告します

- 🆕 in-place Pod resizeに関連するメトリクスが追加されました ([#132903](https://github.com/kubernetes/kubernetes/pull/132903), [@natasha41575](https://github.com/natasha41575))
	* 🆕 `kubelet_container_requested_resizes_total`
	* 🆕 `kubelet_pod_resize_duration_milliseconds`
	* 🆕 `kubelet_pod_pending_resizes`
	* 🆕 `kubelet_pod_infeasible_resizes_total`
	* 🆕 `kubelet_pod_in_progress_resizes`
	* 🆕 `kubelet_pod_deferred_accepted_resizes_total`

- `ResourceClaims` について、admin accessの有無で分類可能なメトリクスが追加されました
	- 🆕 `resourceclaim_controller_creates_total`: Podによる `ResourceClaims` の作成要求の総数及び `status`
	- 🆕 `resourceclaim_controller_resource_claims` ：現在の `ResourceClaims` の数及び確保済みかどうか

- kube-schedulerでのAPI呼び出しを非同期にする `SchedulerAsyncAPICalls` feature gate が有効化されている際に利用可能な3つのメトリクスが追加されました ([#133120](https://github.com/kubernetes/kubernetes/pull/133120), [@utam0k](https://github.com/utam0k))
	- 🆕 `scheduler_async_api_call_execution_total`: 実行された API コールをコールタイプと結果（成功/エラー）ごとに追跡します
	- 🆕 `scheduler_async_api_call_duration_seconds`: 非同期API呼び出しにかかった時間のヒストグラム
	- 🆕 `scheduler_pending_async_api_calls`: QueueにあるpendingのAPI呼び出しの数

* 🆕 swapBehaviorが `LimitedSwap` の際にコンテナに割り当てられるスワップ制限値を確認できる `container_swap_limit_bytes` メトリクスが追加されました。 ([#132348](https://github.com/kubernetes/kubernetes/pull/132348), [@iholder101](https://github.com/iholder101))

- Added a warning when alpha metrics are used with emulated versions. ([#132276](https://github.com/kubernetes/kubernetes/pull/132276), [@michaelasp](https://github.com/michaelasp)) [SIG API Machinery and Architecture] [sig/api-machinery,sig/architecture]
- `KubeletCgroupDriverFromCRI` feature gateがGAとなり、サポートが切れたCRI実装を追跡できるメトリクスが追加されました. ([#133157](https://github.com/kubernetes/kubernetes/pull/133157), [@haircommander](https://github.com/haircommander))
	- 🆕 `kubelet_cri_losing_support` 
		- `version` labelがKubernetesのバージョンです

- `kubelet_container_resize_requests_total` メトリクスについて全てのresize関連の更新が記録がされるようになりました ([#133060](https://github.com/kubernetes/kubernetes/pull/133060), [@natasha41575](https://github.com/natasha41575))
	- resizeがpendingの最中に再度resizeを上書きした際にも記録されるようになるようです。

- apiserverのconfig読み込みに関するメトリクスが追加されました ([#132299](https://github.com/kubernetes/kubernetes/pull/132299), [@aramase](https://github.com/aramase))
	- 🆕 `apiserver_authentication_config_controller_last_config_info`:  authentication configuration fileの読み込み成功時
	- 🆕 `apiserver_authorization_config_controller_last_config_info`: authorization configuration fileの読み込み成功時
	- 🆕 `apiserver_encryption_config_controller_last_config_info`: encryption configuration fileの読み込み成功時

- kubeletが credential provider configurationのhashを `kubelet_credential_provider_config_info` メトリクスで扱えるようになりました。  `hash` labelで確認できます。 ([#133016](https://github.com/kubernetes/kubernetes/pull/133016), [@aramase](https://github.com/aramase))
	- おそらくセキュリティ観点での監視が目的?

## Documentation
* SIG/Instrumentationに関連する項目はありませんでした

## Bug or Regression
* `apiserver_admission_webhook_rejection_count` の誤解を招く応答コードが修正されました ([#132165](https://github.com/kubernetes/kubernetes/pull/132165), [@gavinkflam](https://github.com/gavinkflam))
	* Mutating WebhookのREST Clientを取得する処理がエラーになった際、5XX系を返すべきところで4XX系が返っており、misleadingとなっていました。
	* それに伴い、 `apiserver_admission_webhook_rejection_count` メトリクスの集計方法も一部修正されています。

- swap関連のメトリクスが  `/metrics/resource` エンドポイントで取得できなかった問題が修正されました ([#132065](https://github.com/kubernetes/kubernetes/pull/132065), [@yuanwang04](https://github.com/yuanwang04))

## Other (Cleanup or Flake)
- 🆙 mutaging webhookからのレスポンスのdecodeに失敗した際に失敗(failure)として扱い、failurePolicyが適用され、 `webhook_fail_open_count` メトリクスにカウントされるようになりました ([#131627](https://github.com/kubernetes/kubernetes/pull/131627), [@dims](https://github.com/dims))

* apiserverの `authentication_config_controller` の自動configリロード関連のメトリクスがBETAに昇格しました ([#131798](https://github.com/kubernetes/kubernetes/pull/131798), [@aramase](https://github.com/aramase))
  * `apiserver_authentication_config_controller_automatic_reloads_total`
  * `apiserver_authentication_config_controller_automatic_reload_last_timestamp_seconds`

* apiserverの `authorization_config_controller` の自動configリロード関連のメトリクスがBETAに昇格しました ([#131768](https://github.com/kubernetes/kubernetes/pull/131768), [@aramase](https://github.com/aramase))
  * `apiserver_authorization_config_controller_automatic_reloads_total`
  * `apiserver_authorization_config_controller_automatic_reload_last_timestamp_seconds`

- kube-apiserverのencryption configに関する非推奨のメトリクスが削除されました ([#132238](https://github.com/kubernetes/kubernetes/pull/132238), [@aramase](https://github.com/aramase))
	- 🆑 `apiserver_encryption_config_controller_automatic_reload_success_total`  
	- 🆑 `apiserver_encryption_config_controller_automatic_reload_failure_total`
	- 代わりに `apiserver_encryption_config_controller_automatic_reloads_total` を利用してください

- 🆑 kube-schedulerの非推奨メトリクス `scheduler_scheduler_cache_size` が削除されました ([#131425](https://github.com/kubernetes/kubernetes/pull/131425), [@carlory](https://github.com/carlory))
	- 代わりに `scheduler_cache_size` を利用してください

# Kubernetes Metrics Changes: v1.33.0 → v1.34.0
今回から自動生成したメトリクスの差分の一覧を掲載するようにしました。
実装はこちらにありますので表示形式などフィードバックがあれば歓迎いたします。 https://github.com/tsuzu/k8s-metrics-changes

:::message
現在 `deprecatedVersion` に記載された次のマイナーバージョンではメトリクスが無効化されることを示しているようです。これはメトリクスのstabilityに関わらず同じ挙動となっています。 (ref: [Deprecating a metric](https://kubernetes.io/docs/reference/using-api/deprecation-policy/#deprecating-a-metric)

この挙動については [#133429](https://github.com/kubernetes/kubernetes/issues/133429) にて修正が予定されており、1.35ではメトリクスのstability levelに応じたdeprecationが行われるようです。
:::

## Summary
- **Added**: 26 metrics
- **Removed**: 7 metrics
- **Updated**: 36 metrics
- **Total Changes**: 69 metrics

## Changed Metrics

| Metric Name | Type | Change Type | Stability Level | Description |
|-------------|------|-------------|----------------|-------------|
| [apiserver_authentication_config_controller_automatic_reload_last_timestamp_seconds](#apiserver_authentication_config_controller_automatic_reload_last_timestamp_seconds) | Gauge | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [apiserver_authentication_config_controller_automatic_reloads_total](#apiserver_authentication_config_controller_automatic_reloads_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [apiserver_authentication_config_controller_last_config_info](#apiserver_authentication_config_controller_last_config_info) | Custom | Added | `ALPHA` |  |
| [apiserver_authorization_config_controller_automatic_reload_last_timestamp_seconds](#apiserver_authorization_config_controller_automatic_reload_last_timestamp_seconds) | Gauge | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [apiserver_authorization_config_controller_automatic_reloads_total](#apiserver_authorization_config_controller_automatic_reloads_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [apiserver_authorization_config_controller_last_config_info](#apiserver_authorization_config_controller_last_config_info) | Custom | Added | `ALPHA` |  |
| [apiserver_cache_list_fetched_objects_total](#apiserver_cache_list_fetched_objects_total) | Counter | Updated | `ALPHA` | Added labels: [`group`, `resource`]. <br> Removed labels: [`resource_prefix`]. |
| [apiserver_cache_list_returned_objects_total](#apiserver_cache_list_returned_objects_total) | Counter | Updated | `ALPHA` | Added labels: [`group`, `resource`]. <br> Removed labels: [`resource_prefix`]. |
| [apiserver_cache_list_total](#apiserver_cache_list_total) | Counter | Updated | `ALPHA` | Added labels: [`group`, `resource`]. <br> Removed labels: [`resource_prefix`]. |
| [apiserver_encryption_config_controller_automatic_reload_failures_total](#apiserver_encryption_config_controller_automatic_reload_failures_total) | Counter | Removed | `ALPHA` |  |
| [apiserver_encryption_config_controller_automatic_reload_success_total](#apiserver_encryption_config_controller_automatic_reload_success_total) | Counter | Removed | `ALPHA` |  |
| [apiserver_encryption_config_controller_last_config_info](#apiserver_encryption_config_controller_last_config_info) | Custom | Added | `ALPHA` |  |
| [apiserver_flowcontrol_work_estimated_seats](#apiserver_flowcontrol_work_estimated_seats) | Histogram | Updated | `ALPHA` | Buckets changed. |
| [apiserver_init_events_total](#apiserver_init_events_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_mutating_admission_policy_check_duration_seconds](#apiserver_mutating_admission_policy_check_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [apiserver_mutating_admission_policy_check_total](#apiserver_mutating_admission_policy_check_total) | Counter | Added | `ALPHA` |  |
| [apiserver_request_body_size_bytes](#apiserver_request_body_size_bytes) | Histogram | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_resource_objects](#apiserver_resource_objects) | Gauge | Added | `ALPHA` |  |
| [apiserver_resource_size_estimate_bytes](#apiserver_resource_size_estimate_bytes) | Gauge | Added | `ALPHA` |  |
| [apiserver_selfrequest_total](#apiserver_selfrequest_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_storage_consistency_checks_total](#apiserver_storage_consistency_checks_total) | Counter | Added | `ALPHA` |  |
| [apiserver_storage_decode_errors_total](#apiserver_storage_decode_errors_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_storage_events_received_total](#apiserver_storage_events_received_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_storage_list_evaluated_objects_total](#apiserver_storage_list_evaluated_objects_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_storage_list_fetched_objects_total](#apiserver_storage_list_fetched_objects_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_storage_list_returned_objects_total](#apiserver_storage_list_returned_objects_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_storage_list_total](#apiserver_storage_list_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_storage_objects](#apiserver_storage_objects) | Gauge | Updated | `STABLE` | Help text changed. |
| [apiserver_terminated_watchers_total](#apiserver_terminated_watchers_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_watch_cache_consistent_read_total](#apiserver_watch_cache_consistent_read_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_watch_cache_events_dispatched_total](#apiserver_watch_cache_events_dispatched_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_watch_cache_events_received_total](#apiserver_watch_cache_events_received_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_watch_cache_initializations_total](#apiserver_watch_cache_initializations_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_watch_cache_read_wait_seconds](#apiserver_watch_cache_read_wait_seconds) | Histogram | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_watch_cache_resource_version](#apiserver_watch_cache_resource_version) | Gauge | Updated | `ALPHA` | Added labels: [`group`]. |
| [apiserver_watch_events_sizes](#apiserver_watch_events_sizes) | Histogram | Updated | `ALPHA` | Added labels: [`resource`]. <br> Removed labels: [`kind`]. |
| [apiserver_watch_events_total](#apiserver_watch_events_total) | Counter | Updated | `ALPHA` | Added labels: [`resource`]. <br> Removed labels: [`kind`]. |
| [container_swap_limit_bytes](#container_swap_limit_bytes) | Custom | Added | `ALPHA` |  |
| [dra_resource_claims_in_use](#dra_resource_claims_in_use) | Custom | Added | `ALPHA` |  |
| [etcd_bookmark_counts](#etcd_bookmark_counts) | Gauge | Updated | `ALPHA` | Added labels: [`group`]. |
| [etcd_request_duration_seconds](#etcd_request_duration_seconds) | Histogram | Updated | `ALPHA` | Added labels: [`group`, `resource`]. <br> Removed labels: [`type`]. |
| [etcd_request_errors_total](#etcd_request_errors_total) | Counter | Updated | `ALPHA` | Added labels: [`group`, `resource`]. <br> Removed labels: [`type`]. |
| [etcd_requests_total](#etcd_requests_total) | Counter | Updated | `ALPHA` | Added labels: [`group`, `resource`]. <br> Removed labels: [`type`]. |
| [kubelet_container_requested_resizes_total](#kubelet_container_requested_resizes_total) | Counter | Added | `ALPHA` |  |
| [kubelet_credential_provider_config_info](#kubelet_credential_provider_config_info) | Custom | Added | `ALPHA` |  |
| [kubelet_credential_provider_plugin_errors](#kubelet_credential_provider_plugin_errors) | Counter | Removed | `ALPHA` |  |
| [kubelet_credential_provider_plugin_errors_total](#kubelet_credential_provider_plugin_errors_total) | Counter | Added | `ALPHA` |  |
| [kubelet_cri_losing_support](#kubelet_cri_losing_support) | Gauge | Added | `ALPHA` |  |
| [kubelet_pod_deferred_accepted_resizes_total](#kubelet_pod_deferred_accepted_resizes_total) | Counter | Added | `ALPHA` |  |
| [kubelet_pod_in_progress_resizes](#kubelet_pod_in_progress_resizes) | Gauge | Added | `ALPHA` |  |
| [kubelet_pod_infeasible_resizes_total](#kubelet_pod_infeasible_resizes_total) | Counter | Added | `ALPHA` |  |
| [kubelet_pod_pending_resizes](#kubelet_pod_pending_resizes) | Gauge | Added | `ALPHA` |  |
| [kubelet_pod_resize_duration_milliseconds](#kubelet_pod_resize_duration_milliseconds) | Histogram | Added | `ALPHA` |  |
| [kubelet_started_user_namespaced_pods_errors_total](#kubelet_started_user_namespaced_pods_errors_total) | Counter | Added | `ALPHA` |  |
| [kubelet_started_user_namespaced_pods_total](#kubelet_started_user_namespaced_pods_total) | Counter | Added | `ALPHA` |  |
| [prober_probe_total](#prober_probe_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [resourceclaim_controller_allocated_resource_claims](#resourceclaim_controller_allocated_resource_claims) | Gauge | Removed | `ALPHA` |  |
| [resourceclaim_controller_create_attempts_total](#resourceclaim_controller_create_attempts_total) | Counter | Removed | `ALPHA` |  |
| [resourceclaim_controller_create_failures_total](#resourceclaim_controller_create_failures_total) | Counter | Removed | `ALPHA` |  |
| [resourceclaim_controller_creates_total](#resourceclaim_controller_creates_total) | Counter | Added | `ALPHA` |  |
| [resourceclaim_controller_resource_claims](#resourceclaim_controller_resource_claims) | Custom | Updated | `ALPHA` | Help text changed. <br> Type changed from `Gauge` to `Custom`. <br> Added labels: [`admin_access`, `allocated`]. |
| [scheduler_async_api_call_execution_duration_seconds](#scheduler_async_api_call_execution_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [scheduler_async_api_call_execution_total](#scheduler_async_api_call_execution_total) | Counter | Added | `ALPHA` |  |
| [scheduler_pending_async_api_calls](#scheduler_pending_async_api_calls) | Gauge | Added | `ALPHA` |  |
| [scheduler_scheduler_cache_size](#scheduler_scheduler_cache_size) | Gauge | Removed | `ALPHA` |  |
| [version_info](#version_info) | Gauge | Added | `ALPHA` |  |
| [watch_cache_capacity](#watch_cache_capacity) | Gauge | Updated | `ALPHA` | Added labels: [`group`]. |
| [watch_cache_capacity_decrease_total](#watch_cache_capacity_decrease_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
| [watch_cache_capacity_increase_total](#watch_cache_capacity_increase_total) | Counter | Updated | `ALPHA` | Added labels: [`group`]. |
## Detailed Changes

### apiserver_authentication_config_controller_automatic_reload_last_timestamp_seconds
```diff
 - name: automatic_reload_last_timestamp_seconds
   subsystem: authentication_config_controller
   namespace: apiserver
   help: Timestamp of the last automatic reload of authentication configuration split by status and apiserver identity.
   type: Gauge
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - apiserver_id_hash
     - status
```

### apiserver_authentication_config_controller_automatic_reloads_total
```diff
 - name: automatic_reloads_total
   subsystem: authentication_config_controller
   namespace: apiserver
   help: Total number of automatic reloads of authentication configuration split by status and apiserver identity.
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - apiserver_id_hash
     - status
```

### apiserver_authentication_config_controller_last_config_info
```diff
+- name: apiserver_authentication_config_controller_last_config_info
+  help: Information about the last applied authentication configuration with hash as label, split by apiserver identity.
+  type: Custom
+  stabilityLevel: ALPHA
+  labels:
+    - apiserver_id_hash
+    - hash
```

### apiserver_authorization_config_controller_automatic_reload_last_timestamp_seconds
```diff
 - name: automatic_reload_last_timestamp_seconds
   subsystem: authorization_config_controller
   namespace: apiserver
   help: Timestamp of the last automatic reload of authorization configuration split by status and apiserver identity.
   type: Gauge
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - apiserver_id_hash
     - status
```

### apiserver_authorization_config_controller_automatic_reloads_total
```diff
 - name: automatic_reloads_total
   subsystem: authorization_config_controller
   namespace: apiserver
   help: Total number of automatic reloads of authorization configuration split by status and apiserver identity.
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - apiserver_id_hash
     - status
```

### apiserver_authorization_config_controller_last_config_info
```diff
+- name: apiserver_authorization_config_controller_last_config_info
+  help: Information about the last applied authorization configuration with hash as label, split by apiserver identity.
+  type: Custom
+  stabilityLevel: ALPHA
+  labels:
+    - apiserver_id_hash
+    - hash
```

### apiserver_cache_list_fetched_objects_total
```diff
 - name: cache_list_fetched_objects_total
   namespace: apiserver
   help: Number of objects read from watch cache in the course of serving a LIST request
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - index
-    - resource_prefix
+    - resource
```

### apiserver_cache_list_returned_objects_total
```diff
 - name: cache_list_returned_objects_total
   namespace: apiserver
   help: Number of objects returned for a LIST request from watch cache
   type: Counter
   stabilityLevel: ALPHA
   labels:
-    - resource_prefix
+    - group
+    - resource
```

### apiserver_cache_list_total
```diff
 - name: cache_list_total
   namespace: apiserver
   help: Number of LIST requests served from watch cache
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - index
-    - resource_prefix
+    - resource
```

### apiserver_encryption_config_controller_automatic_reload_failures_total
```diff
-- name: automatic_reload_failures_total
-  subsystem: encryption_config_controller
-  namespace: apiserver
-  help: Total number of failed automatic reloads of encryption configuration split by apiserver identity.
-  type: Counter
-  deprecatedVersion: 1.30.0
-  stabilityLevel: ALPHA
-  labels:
-    - apiserver_id_hash
```

### apiserver_encryption_config_controller_automatic_reload_success_total
```diff
-- name: automatic_reload_success_total
-  subsystem: encryption_config_controller
-  namespace: apiserver
-  help: Total number of successful automatic reloads of encryption configuration split by apiserver identity.
-  type: Counter
-  deprecatedVersion: 1.30.0
-  stabilityLevel: ALPHA
-  labels:
-    - apiserver_id_hash
```

### apiserver_encryption_config_controller_last_config_info
```diff
+- name: apiserver_encryption_config_controller_last_config_info
+  help: Information about the last applied encryption configuration with hash as label, split by apiserver identity.
+  type: Custom
+  stabilityLevel: ALPHA
+  labels:
+    - apiserver_id_hash
+    - hash
```

### apiserver_flowcontrol_work_estimated_seats
```diff
 - name: work_estimated_seats
   subsystem: flowcontrol
   namespace: apiserver
   help: Number of estimated seats (maximum of initial and final seats) associated with requests in API Priority and Fairness
   type: Histogram
   stabilityLevel: ALPHA
   labels:
     - flow_schema
     - priority_level
   buckets:
     - 1
     - 2
     - 4
-    - 10
+    - 8
+    - 16
+    - 32
+    - 64
+    - 100
```

### apiserver_init_events_total
```diff
 - name: init_events_total
   namespace: apiserver
   help: Counter of init events processed in watch cache broken by resource type.
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### apiserver_mutating_admission_policy_check_duration_seconds
```diff
+- name: check_duration_seconds
+  subsystem: mutating_admission_policy
+  namespace: apiserver
+  help: Mutation admission latency for individual mutation expressions in seconds, labeled by policy and binding.
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - error_type
+    - policy
+    - policy_binding
+  buckets:
+    - 5e-07
+    - 0.001
+    - 0.01
+    - 0.1
+    - 1
```

### apiserver_mutating_admission_policy_check_total
```diff
+- name: check_total
+  subsystem: mutating_admission_policy
+  namespace: apiserver
+  help: Mutation admission policy check total, labeled by policy and further identified by binding.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - error_type
+    - policy
+    - policy_binding
```

### apiserver_request_body_size_bytes
```diff
 - name: request_body_size_bytes
   subsystem: apiserver
   help: Apiserver request body size in bytes broken out by resource and verb.
   type: Histogram
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
     - verb
   buckets:
     - 50000
     - 150000
     - 250000
     - 350000
     - 450000
     - 550000
     - 650000
     - 750000
     - 850000
     - 950000
     - 1.05e+06
     - 1.15e+06
     - 1.25e+06
     - 1.35e+06
     - 1.45e+06
     - 1.55e+06
     - 1.65e+06
     - 1.75e+06
     - 1.85e+06
     - 1.95e+06
     - 2.05e+06
     - 2.15e+06
     - 2.25e+06
     - 2.35e+06
     - 2.45e+06
     - 2.55e+06
     - 2.65e+06
     - 2.75e+06
     - 2.85e+06
     - 2.95e+06
     - 3.05e+06
```

### apiserver_resource_objects
```diff
+- name: apiserver_resource_objects
+  help: Number of stored objects at the time of last check split by kind. In case of a fetching error, the value will be -1.
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - group
+    - resource
```

### apiserver_resource_size_estimate_bytes
```diff
+- name: apiserver_resource_size_estimate_bytes
+  help: Estimated size of stored objects in database. Estimate is based on sum of last observed sizes of serialized objects. In case of a fetching error, the value will be -1.
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - group
+    - resource
```

### apiserver_selfrequest_total
```diff
 - name: selfrequest_total
   subsystem: apiserver
   help: Counter of apiserver self-requests broken out for each verb, API resource and subresource.
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
     - subresource
     - verb
```

### apiserver_storage_consistency_checks_total
```diff
+- name: storage_consistency_checks_total
+  namespace: apiserver
+  help: Counter for status of consistency checks between etcd and watch cache
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - group
+    - resource
+    - status
```

### apiserver_storage_decode_errors_total
```diff
 - name: storage_decode_errors_total
   namespace: apiserver
   help: Number of stored object decode errors split by object type
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### apiserver_storage_events_received_total
```diff
 - name: storage_events_received_total
   subsystem: apiserver
   help: Number of etcd events received split by kind.
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### apiserver_storage_list_evaluated_objects_total
```diff
 - name: apiserver_storage_list_evaluated_objects_total
   help: Number of objects tested in the course of serving a LIST request from storage
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### apiserver_storage_list_fetched_objects_total
```diff
 - name: apiserver_storage_list_fetched_objects_total
   help: Number of objects read from storage in the course of serving a LIST request
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### apiserver_storage_list_returned_objects_total
```diff
 - name: apiserver_storage_list_returned_objects_total
   help: Number of objects returned for a LIST request from storage
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### apiserver_storage_list_total
```diff
 - name: apiserver_storage_list_total
   help: Number of LIST requests served from storage
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### apiserver_storage_objects
```diff
 - name: apiserver_storage_objects
-  help: Number of stored objects at the time of last check split by kind. In case of a fetching error, the value will be -1.
+  help: '[DEPRECATED, consider using apiserver_resource_objects instead] Number of stored objects at the time of last check split by kind. In case of a fetching error, the value will be -1.'
   type: Gauge
   stabilityLevel: STABLE
   labels:
     - resource
```

### apiserver_terminated_watchers_total
```diff
 - name: terminated_watchers_total
   namespace: apiserver
   help: Counter of watchers closed due to unresponsiveness broken by resource type.
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### apiserver_watch_cache_consistent_read_total
```diff
 - name: consistent_read_total
   subsystem: watch_cache
   namespace: apiserver
   help: Counter for consistent reads from cache.
   type: Counter
   stabilityLevel: ALPHA
   labels:
     - fallback
+    - group
     - resource
     - success
```

### apiserver_watch_cache_events_dispatched_total
```diff
 - name: events_dispatched_total
   subsystem: watch_cache
   namespace: apiserver
   help: Counter of events dispatched in watch cache broken by resource type.
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### apiserver_watch_cache_events_received_total
```diff
 - name: events_received_total
   subsystem: watch_cache
   namespace: apiserver
   help: Counter of events received in watch cache broken by resource type.
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### apiserver_watch_cache_initializations_total
```diff
 - name: initializations_total
   subsystem: watch_cache
   namespace: apiserver
   help: Counter of watch cache initializations broken by resource type.
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### apiserver_watch_cache_read_wait_seconds
```diff
 - name: read_wait_seconds
   subsystem: watch_cache
   namespace: apiserver
   help: Histogram of time spent waiting for a watch cache to become fresh.
   type: Histogram
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
   buckets:
     - 0.005
     - 0.025
     - 0.05
     - 0.1
     - 0.2
     - 0.4
     - 0.6
     - 0.8
     - 1
     - 1.25
     - 1.5
     - 2
     - 3
```

### apiserver_watch_cache_resource_version
```diff
 - name: resource_version
   subsystem: watch_cache
   namespace: apiserver
   help: Current resource version of watch cache broken by resource type.
   type: Gauge
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### apiserver_watch_events_sizes
```diff
 - name: watch_events_sizes
   subsystem: apiserver
   help: Watch event size distribution in bytes
   type: Histogram
   stabilityLevel: ALPHA
   labels:
     - group
-    - kind
+    - resource
     - version
   buckets:
     - 1024
     - 2048
     - 4096
     - 8192
     - 16384
     - 32768
     - 65536
     - 131072
```

### apiserver_watch_events_total
```diff
 - name: watch_events_total
   subsystem: apiserver
   help: Number of events sent in watch clients
   type: Counter
   stabilityLevel: ALPHA
   labels:
     - group
-    - kind
+    - resource
     - version
```

### container_swap_limit_bytes
```diff
+- name: container_swap_limit_bytes
+  help: Current amount of the container swap limit in bytes. Reported only on non-windows systems
+  type: Custom
+  stabilityLevel: ALPHA
+  labels:
+    - container
+    - pod
+    - namespace
```

### dra_resource_claims_in_use
```diff
+- name: dra_resource_claims_in_use
+  help: The number of ResourceClaims that are currently in use on the node, by driver name (driver_name label value) and across all drivers (special value <any> for driver_name). Note that the sum of all by-driver counts is not the total number of in-use ResourceClaims because the same ResourceClaim might use devices from different drivers. Instead, use the count for the <any> driver_name.
+  type: Custom
+  stabilityLevel: ALPHA
+  labels:
+    - driver_name
```

### etcd_bookmark_counts
```diff
 - name: etcd_bookmark_counts
   help: Number of etcd bookmarks (progress notify events) split by kind.
   type: Gauge
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### etcd_request_duration_seconds
```diff
 - name: etcd_request_duration_seconds
   help: Etcd request latency in seconds for each operation and object type.
   type: Histogram
   stabilityLevel: ALPHA
   labels:
+    - group
     - operation
-    - type
+    - resource
   buckets:
     - 0.005
     - 0.025
     - 0.05
     - 0.1
     - 0.2
     - 0.4
     - 0.6
     - 0.8
     - 1
     - 1.25
     - 1.5
     - 2
     - 3
     - 4
     - 5
     - 6
     - 8
     - 10
     - 15
     - 20
     - 30
     - 45
     - 60
```

### etcd_request_errors_total
```diff
 - name: etcd_request_errors_total
   help: Etcd failed request counts for each operation and object type.
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - operation
-    - type
+    - resource
```

### etcd_requests_total
```diff
 - name: etcd_requests_total
   help: Etcd request counts for each operation and object type.
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - operation
-    - type
+    - resource
```

### kubelet_container_requested_resizes_total
```diff
+- name: container_requested_resizes_total
+  subsystem: kubelet
+  help: Number of requested resizes, counted at the container level. Different resources on the same container are counted separately. The 'requirement' label refers to 'memory' or 'limits'; the 'operation' label can be one of 'add', 'remove', 'increase' or 'decrease'.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - operation
+    - requirement
+    - resource
```

### kubelet_credential_provider_config_info
```diff
+- name: kubelet_credential_provider_config_info
+  help: Information about the last applied credential provider configuration with hash as label
+  type: Custom
+  stabilityLevel: ALPHA
+  labels:
+    - hash
```

### kubelet_credential_provider_plugin_errors
```diff
-- name: credential_provider_plugin_errors
-  subsystem: kubelet
-  help: Number of errors from credential provider plugin
-  type: Counter
-  stabilityLevel: ALPHA
-  labels:
-    - plugin_name
```

### kubelet_credential_provider_plugin_errors_total
```diff
+- name: credential_provider_plugin_errors_total
+  subsystem: kubelet
+  help: Number of errors from credential provider plugin
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - plugin_name
```

### kubelet_cri_losing_support
```diff
+- name: cri_losing_support
+  subsystem: kubelet
+  help: the Kubernetes version that the currently running CRI implementation will lose support on if not upgraded.
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - version
```

### kubelet_pod_deferred_accepted_resizes_total
```diff
+- name: pod_deferred_accepted_resizes_total
+  subsystem: kubelet
+  help: Cumulative number of resizes that were accepted after being deferred.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - retry_trigger
```

### kubelet_pod_in_progress_resizes
```diff
+- name: pod_in_progress_resizes
+  subsystem: kubelet
+  help: Number of in-progress resizes for pods.
+  type: Gauge
+  stabilityLevel: ALPHA
```

### kubelet_pod_infeasible_resizes_total
```diff
+- name: pod_infeasible_resizes_total
+  subsystem: kubelet
+  help: Number of infeasible resizes for pods.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - reason_detail
```

### kubelet_pod_pending_resizes
```diff
+- name: pod_pending_resizes
+  subsystem: kubelet
+  help: Number of pending resizes for pods.
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - reason
```

### kubelet_pod_resize_duration_milliseconds
```diff
+- name: pod_resize_duration_milliseconds
+  subsystem: kubelet
+  help: Duration in milliseconds to actuate a pod resize
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - success
+  buckets:
+    - 10
+    - 50
+    - 100
+    - 500
+    - 1000
+    - 2000
+    - 5000
+    - 10000
+    - 20000
+    - 30000
+    - 60000
+    - 120000
+    - 300000
+    - 600000
```

### kubelet_started_user_namespaced_pods_errors_total
```diff
+- name: started_user_namespaced_pods_errors_total
+  subsystem: kubelet
+  help: Cumulative number of errors when starting pods with user namespaces. This metric will only be collected on Linux.
+  type: Counter
+  stabilityLevel: ALPHA
```

### kubelet_started_user_namespaced_pods_total
```diff
+- name: started_user_namespaced_pods_total
+  subsystem: kubelet
+  help: Cumulative number of pods with user namespaces started. This metric will only be collected on Linux.
+  type: Counter
+  stabilityLevel: ALPHA
```

### prober_probe_total
```diff
 - name: probe_total
   subsystem: prober
   help: Cumulative number of a liveness, readiness or startup probe for a container by result.
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - container
     - namespace
     - pod
     - pod_uid
     - probe_type
     - result
```

### resourceclaim_controller_allocated_resource_claims
```diff
-- name: allocated_resource_claims
-  subsystem: resourceclaim_controller
-  help: Number of allocated ResourceClaims
-  type: Gauge
-  stabilityLevel: ALPHA
```

### resourceclaim_controller_create_attempts_total
```diff
-- name: create_attempts_total
-  subsystem: resourceclaim_controller
-  help: Number of ResourceClaims creation requests
-  type: Counter
-  stabilityLevel: ALPHA
```

### resourceclaim_controller_create_failures_total
```diff
-- name: create_failures_total
-  subsystem: resourceclaim_controller
-  help: Number of ResourceClaims creation request failures
-  type: Counter
-  stabilityLevel: ALPHA
```

### resourceclaim_controller_creates_total
```diff
+- name: creates_total
+  subsystem: resourceclaim_controller
+  help: Number of ResourceClaims creation requests, categorized by creation status and admin access
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - admin_access
+    - status
```

### resourceclaim_controller_resource_claims
```diff
-- name: resource_claims
-  subsystem: resourceclaim_controller
-  help: Number of ResourceClaims
-  type: Gauge
+- name: resourceclaim_controller_resource_claims
+  help: Number of ResourceClaims, categorized by allocation status and admin access
+  type: Custom
   stabilityLevel: ALPHA
+  labels:
+    - allocated
+    - admin_access
```

### scheduler_async_api_call_execution_duration_seconds
```diff
+- name: async_api_call_execution_duration_seconds
+  subsystem: scheduler
+  help: Duration in seconds for executing API call in the async dispatcher.
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - call_type
+    - result
+  buckets:
+    - 0.001
+    - 0.002
+    - 0.004
+    - 0.008
+    - 0.016
+    - 0.032
+    - 0.064
+    - 0.128
+    - 0.256
+    - 0.512
+    - 1.024
+    - 2.048
+    - 4.096
+    - 8.192
+    - 16.384
```

### scheduler_async_api_call_execution_total
```diff
+- name: async_api_call_execution_total
+  subsystem: scheduler
+  help: Total number of API calls executed by the async dispatcher.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - call_type
+    - result
```

### scheduler_pending_async_api_calls
```diff
+- name: pending_async_api_calls
+  subsystem: scheduler
+  help: Number of API calls currently pending in the async queue.
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - call_type
```

### scheduler_scheduler_cache_size
```diff
-- name: scheduler_cache_size
-  subsystem: scheduler
-  help: Number of nodes, pods, and assumed (bound) pods in the scheduler cache.
-  type: Gauge
-  deprecatedVersion: 1.33.0
-  stabilityLevel: ALPHA
-  labels:
-    - type
```

### version_info
```diff
+- name: version_info
+  help: Provides the compatibility version info of the component. The component label is the name of the component, usually kube, but is relevant for aggregated-apiservers.
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - binary
+    - component
+    - emulation
+    - min_compat
```

### watch_cache_capacity
```diff
 - name: capacity
   subsystem: watch_cache
   help: Total capacity of watch cache broken by resource type.
   type: Gauge
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### watch_cache_capacity_decrease_total
```diff
 - name: capacity_decrease_total
   subsystem: watch_cache
   help: Total number of watch cache capacity decrease events broken by resource type.
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

### watch_cache_capacity_increase_total
```diff
 - name: capacity_increase_total
   subsystem: watch_cache
   help: Total number of watch cache capacity increase events broken by resource type.
   type: Counter
   stabilityLevel: ALPHA
   labels:
+    - group
     - resource
```

