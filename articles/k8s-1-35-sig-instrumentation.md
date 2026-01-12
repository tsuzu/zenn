---
title: "Kubernetes 1.35: SIG-Instrumentationの変更内容" # 記事のタイトル
emoji: "☸️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["kubernetes"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

# はじめに
本記事では Kubernetes v1.34 の [Changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.35.md) から、SIG-Instrumentation 関連及びメトリクスの変更点について取り上げ、まとめています。

* [Kubernetes 1.31 SIG Instrumentation の変更内容](https://qiita.com/watawuwu/items/ae739b0a2085bd16c0f9)
* [Kubernetes 1.32 SIG Instrumentation の変更内容](https://qiita.com/watawuwu/items/5280af2088bbc1a86aca)
* [Kubernetes 1.33: SIG-Instrumentationの変更内容](https://zenn.dev/tsuzu/articles/k8s-1-33-sig-instrumentation)
* [Kubernetes 1.33: SIG-Instrumentationの変更内容](https://zenn.dev/tsuzu/articles/k8s-1-34-sig-instrumentation)

:::message
メトリクス名の横に絵文字で変更の種類を記載しています。

* 🆕: 新規で追加されたメトリクス
* 🆙: ラベルなどに更新があったメトリクス
* ⚠️: 非推奨になったメトリクス
* 🆑: 削除になったメトリクス
:::

# Changes by kind
## Upgrade Notes


## Deprecation


## API Changes

- Added validation to ensure `log-flush-frequency` is a positive value, returning an error instead of causing a panic. ([#133540](https://github.com/kubernetes/kubernetes/pull/133540), [@BenTheElder](https://github.com/BenTheElder)) [SIG Architecture, Instrumentation, Network and Node] [sig/network,sig/node,sig/instrumentation,sig/architecture]
  - 不正値（例: --log-flush-frequency=-0）で time.NewTicker が panic する事象が報告されており、コンポーネントがクラッシュするより「設定エラーとして弾く」方が安全なため修正

- Changed kuberc configuration schema. Two new optional fields added to kuberc configuration, `credPluginPolicy` and `credPluginAllowlist`. This is documented in [KEP-3104](https://github.com/kubernetes/enhancements/blob/master/keps/sig-cli/3104-introduce-kuberc/README.md#allowlist-design-details) and documentation is added to the website by [kubernetes/website#52877](https://github.com/kubernetes/website/pull/52877) ([#134870](https://github.com/kubernetes/kubernetes/pull/134870), [@pmengelbert](https://github.com/pmengelbert)) [SIG API Machinery, Architecture, Auth, CLI, Instrumentation and Testing] [sig/api-machinery,sig/auth,sig/cli,sig/instrumentation,sig/testing,sig/architecture]
  * client-goで利用可能なcredential pluginのポリシーを指定できる機能がkubercに追加されました。
  * それに伴い [rest_client_exec_plugin_policy_call_total](#rest_client_exec_plugin_policy_call_total) メトリクスにてpolicyの評価結果をメトリクスとして取得できるようになりました。

- Enabled in-place resizing of pod-level resources.
- Added `Resources` in `PodStatus` to capture resources set in the pod-level cgroup.
- Added `AllocatedResources` in `PodStatus` to capture resources requested in the `PodSpec`. ([#132919](https://github.com/kubernetes/kubernetes/pull/132919), [@ndixita](https://github.com/ndixita)) [SIG API Machinery, Apps, Architecture, Auth, CLI, Instrumentation, Node, Scheduling and Testing] [sig/scheduling,sig/node,sig/api-machinery,sig/auth,sig/apps,sig/cli,sig/instrumentation,sig/testing,sig/architecture]
  * Podレベルのin-place resizeがAlphaとして導入されました。
  * 合わせてResizeができない場合に `pod_infeasible_resizes_total` メトリクスに反映されています。

- Introduced a structured and versioned `v1alpha1` response for the `statusz` endpoint. ([#134313](https://github.com/kubernetes/kubernetes/pull/134313), [@richabanker](https://github.com/richabanker)) [SIG API Machinery, Architecture, Instrumentation, Network, Node, Scheduling and Testing] [sig/network,sig/scheduling,sig/node,sig/api-machinery,sig/instrumentation,sig/testing,sig/architecture]
  * `Accept: application/json;v=v1alpha1;g=config.k8s.io;as=Statusz` のHeaderを付与するとstructuredかつversionedな kube-apiserver等のバージョン情報が取得できるようになりました。これにより機械的に取得がしやすくなります。

- Introduced a structured and versioned `v1alpha1` response format for the `flagz` endpoint. ([#134995](https://github.com/kubernetes/kubernetes/pull/134995), [@yongruilin](https://github.com/yongruilin)) [SIG API Machinery, Architecture, Instrumentation, Network, Node, Scheduling and Testing] [sig/network,sig/scheduling,sig/node,sig/api-machinery,sig/instrumentation,sig/testing,sig/architecture]
  * `Accept: application/json;v=v1alpha1;g=config.k8s.io;as=Flagz`


```bash
✦ ❯ curl localhost:8001/statusz

apiserver statusz
Warning: This endpoint is not meant to be machine parseable, has no formatting compatibility guarantees and is for debugging purposes only.

Started:  Mon Jan 12 11:37:40 UTC 2026
Up:  0 hr 00 min 27 sec
Go version:  go1.25.5
Binary version:  1.35.0
Emulation version:  1.35
Paths:  /flagz /healthz /livez /metrics /readyz /statusz /version

zenn on  main [!+?] on ☁  (ap-northeast-1)
✦ ❯ curl localhost:8001/statusz  -H "Accept: application/json;v=v1alpha1;g=config.k8s.io;as=Statusz"
{
  "kind": "Statusz",
  "apiVersion": "config.k8s.io/v1alpha1",
  "metadata": {
    "name": "apiserver"
  },
  "startTime": "2026-01-12T11:37:40Z",
  "uptimeSeconds": 54,
  "goVersion": "go1.25.5",
  "binaryVersion": "1.35.0",
  "emulationVersion": "1.35",
  "paths": [
    "/flagz",
    "/healthz",
    "/livez",
    "/metrics",
    "/readyz",
    "/statusz",
    "/version"
  ]
}%
```


## Features

- Added a `source` label to the `resourceclaim_controller_resource_claims` metric.
Added the `scheduler_resourceclaim_creates_total` metric for `DRAExtendedResource`. ([#134523](https://github.com/kubernetes/kubernetes/pull/134523), [@bitoku](https://github.com/bitoku)) [SIG Apps, Instrumentation, Node and Scheduling] [sig/scheduling,sig/node,sig/apps,sig/instrumentation]
  * [scheduler_resourceclaim_creates_total](#scheduler_resourceclaim_creates_total)
    * スケジューラの PreBind フェーズで Kubernetes API を呼び出して ResourceClaim を作成しているため、resourceclaim_controller_creates_total とは別のメトリクスが必要です。
    * このメトリクスは、API 呼び出しの結果（成功または失敗）に応じてカウントが増加します。
  * [resourceclaim_controller_resource_claims](#resourceclaim_controller_resource_claims)
    * `source` labelが追加されました。
      * `resource_claim_template`
        * ResourceClaimTemplateで要求されたリソース
      * `extended_resource` 
        * DRA以前の `pods.spec.containers[].resources.requests` で定義されるリソースリクエスト
      * empty
        * ユーザが手動でResourceClaimを作成した場合

- Added a counter metric `kubelet_image_manager_ensure_image_requests_total{present_locally, pull_policy, pull_required}` that exposes details about `kubelet` ensuring an image exists on the node. ([#132644](https://github.com/kubernetes/kubernetes/pull/132644), [@stlaz](https://github.com/stlaz)) [SIG Auth and Node] [sig/node,sig/auth]
  * [kubelet_image_manager_ensure_image_requests_total](#kubelet_image_manager_ensure_image_requests_total)
  * Podがimageを利用する際に権限を持っていることを保証するKEP-2535 Ensure Secret Pulled Imageの一環です。
  * イメージがローカルにあるか、Pull policyの種類、pullが必要かどうかをmetricsで出力します。
    * どのイメージについてかのlabelはないためイメージ単位での計測はできなそうです。

- Added metrics for the `MaxUnavailable` feature in `StatefulSet`. ([#130951](https://github.com/kubernetes/kubernetes/pull/130951), [@Edwinhr716](https://github.com/Edwinhr716)) [SIG Apps and Instrumentation] [sig/apps,sig/instrumentation]
  * [statefulset_controller_statefulset_max_unavailable](#statefulset_controller_statefulset_max_unavailable)
  * `KEP-961 maxUnavailable for StatefulSets` のbeta昇格に合わせてStatefulSetのmaxUnavailableがmetricsとして取得できるようになりました。

- Added paths section to scheduler `statusz` endpoint. ([#132606](https://github.com/kubernetes/kubernetes/pull/132606), [@Peac36](https://github.com/Peac36)) [SIG API Machinery, Architecture, Instrumentation, Network, Node, Scheduling and Testing] [sig/network,sig/scheduling,sig/node,sig/api-machinery,sig/instrumentation,sig/testing,sig/architecture]
  * [z-pages](https://kubernetes.io/docs/reference/instrumentation/zpages/) の `/statusz` エンドポイントに、その他の利用可能な `/healthz` 等のパスが返却されるようになりました。
    * Example: `Paths: /flagz /healthz /livez /metrics /readyz /statusz /version`

- Graduated the image volume source feature to Beta and enabled it by default. ([#135195](https://github.com/kubernetes/kubernetes/pull/135195), [@haircommander](https://github.com/haircommander)) [SIG Apps, Instrumentation, Node and Testing] [sig/node,sig/apps,sig/instrumentation,sig/testing]
  * ImageVolume FeatureGateのBETA昇格に合わせて以下のmetricsがbetaに昇格しました。
    * [kubelet_image_volume_requested_total](#kubelet_image_volume_requested_total)
    * [kubelet_image_volume_mounted_errors_total](#kubelet_image_volume_mounted_errors_total)
    * [kubelet_image_volume_mounted_succeed_total](#kubelet_image_volume_mounted_succeed_total)

- Metrics: Excluded `dryRun` requests from `apiserver_request_sli_duration_seconds`. ([#131092](https://github.com/kubernetes/kubernetes/pull/131092), [@aldudko](https://github.com/aldudko)) [SIG API Machinery and Instrumentation] [sig/api-machinery,sig/instrumentation]
  * `apiserver_request_sli_duration_seconds` metricsから `--dry-run=server` 等でdry-runリクエストが除外されるようになりました。
    * このmetricsを用いてS監視している場合値が悪化する可能性があります。

- Moved the Pod Certificates feature to beta. Added `UserAnnotations` to the `PodCertificateProjection` API and `UnverifiedUserAnnotations` to the `PodCertificateRequest` API. The `PodCertificateRequest` feature gate remains disabled by default and requires enabling the v1beta1 certificates API groups. ([#134790](https://github.com/kubernetes/kubernetes/pull/134790), [@yt2985](https://github.com/yt2985)) [SIG Auth, Instrumentation and Testing] [sig/auth,sig/instrumentation,sig/testing]
  * [kubelet_podcertificate_states](#kubelet_podcertificate_states)
  * KEP-4317: Pod Certificatesで導入されたPodCertificateProjectionがBetaに昇格するにあたり、kubeletがprojecterd volumeとしてmountされた証明書についての情報をreportするようになりました。
    * stateは以下の5種類を取ります。
      * `fresh`: 証明書は最新です
      * `overdue_for_refresh`: 現在時刻は、最後に発行された証明書の beginRefreshAt タイムスタンプから10分以上経過しています。
      * `expired`: 現在時刻は、最も最近発行された証明書の notAfter タイムスタンプを過ぎています。
      * `failed`: 最新の発行または更新試行は「Failed」条件で終了しました。
      * `denied`: 最新の発行または更新試行は「Denied」状態で終了しました。

- The JWT authenticator in `kube-apiserver` now reports the following metrics when the `StructuredAuthenticationConfiguration` feature gate is enabled:
- `apiserver_authentication_jwt_authenticator_jwks_fetch_last_timestamp_seconds`
- `apiserver_authentication_jwt_authenticator_jwks_fetch_last_key_set_info`. ([#123642](https://github.com/kubernetes/kubernetes/pull/123642), [@aramase](https://github.com/aramase)) [SIG API Machinery, Auth and Testing] [sig/api-machinery,sig/auth,sig/testing]
  * [apiserver_authentication_jwt_authenticator_jwks_fetch_last_timestamp_seconds](#apiserver_authentication_jwt_authenticator_jwks_fetch_last_timestamp_seconds)
  * [apiserver_authentication_jwt_authenticator_jwks_fetch_last_key_set_info](#apiserver_authentication_jwt_authenticator_jwks_fetch_last_key_set_info)
  * KEP-3331: Structured Authentication Configに関連し、JWTを検証するためのJWKSを最後に取得した時刻と取得したJWKSの情報(ハッシュ値など)をメトリクスで出力するようになりました。


## Documentation


## Bug or Regression

- Deprecated metrics will be hidden as per the metrics deprecation policy. https://kubernetes.io/docs/reference/using-api/deprecation-policy/#deprecating-a-metric . ([#133436](https://github.com/kubernetes/kubernetes/pull/133436), [@richabanker](https://github.com/richabanker)) [SIG Architecture, Instrumentation and Network] [sig/network,sig/instrumentation,sig/architecture]
  * 対応する安定性クラスにおいて、メトリクスは以下の期間にわたって機能し続ける必要があります：
    * STABLE：4回のリリースまたは12ヶ月（いずれか長い方）
    * BETA：2回のリリースまたは8ヶ月（いずれか長い方）
    * ALPHA：リリース0回
* 廃止が正式に告知された後のメトリクスについては、以下の期間にわたって機能し続ける必要があります：
  * STABLE：3回のリリースまたは9ヶ月（いずれか長い方）
  * BETA：1回のリリースまたは4ヶ月（いずれか長い方）
  * ALPHA：リリース0回

- Fixed a possible data race during metrics registration. ([#134390](https://github.com/kubernetes/kubernetes/pull/134390), [@liggitt](https://github.com/liggitt)) [SIG Architecture and Instrumentation] [sig/instrumentation,sig/architecture]
  * メトリクスの収集機構周りにバグがあり、booleanへのアクセスに排他制御がありませんでした。 `atomic.Bool` を使って修正されました。

- Fixed missing `kubelet_volume_stats_*` metrics. ([#133890](https://github.com/kubernetes/kubernetes/pull/133890), [@huww98](https://github.com/huww98)) [SIG Instrumentation and Node] [sig/node,sig/instrumentation]
  * `kubelet_volume_stats_*` がFeatureGateを有効化されていてもmetricsが公開されていなかった問題が修正されました。

## Other (Cleanup or Flake)
- Removed the `ComponentSLIs` feature gate, as it was promoted to stable in the Kubernetes `v1.32` release. ([#133742](https://github.com/kubernetes/kubernetes/pull/133742), [@carlory](https://github.com/carlory)) [SIG Architecture and Instrumentation] [sig/instrumentation,sig/architecture]
  - 1.32でstableになった `ComponentSLIs` FeatureGateが削除され常に有効化されるようになりました。

- Specified the deprecated version of `apiserver_storage_objects` metric in metrics docs. ([#134028](https://github.com/kubernetes/kubernetes/pull/134028), [@richabanker](https://github.com/richabanker)) [SIG API Machinery, Etcd and Instrumentation] [sig/api-machinery,sig/instrumentation,sig/etcd]
  * #133436 に関連し `apiserver_storage_objects` にdeprecatedVersionが設定されました。

# Kubernetes Metrics Changes: v1.34.0 → v1.35.0
自動生成したメトリクスの差分の一覧を掲載しています。
実装はこちらにありますので表示形式などフィードバックがあれば歓迎いたします。 https://github.com/tsuzu/k8s-metrics-changes

:::message
deprecationVersionの挙動についての修正が [#133429](https://github.com/kubernetes/kubernetes/issues/133429) にて行われました。この影響で1.34のメトリクスのdeprecationが1.35の変更として出ています。
実際には1.34からdeprecateされています。
:::


## Summary
- **Added**: 25 metrics
- **Removed**: 0 metrics
- **Updated**: 12 metrics
- **Total Changes**: 37 metrics

## Changed Metrics

| Metric Name | Type | Change Type | Stability Level | Description |
|-------------|------|-------------|----------------|-------------|
| [aggregator_discovery_nopeer_requests_total](#aggregator_discovery_nopeer_requests_total) | Counter | Added | `ALPHA` |  |
| [aggregator_discovery_peer_aggregated_cache_hits_total](#aggregator_discovery_peer_aggregated_cache_hits_total) | Counter | Added | `ALPHA` |  |
| [aggregator_discovery_peer_aggregated_cache_misses_total](#aggregator_discovery_peer_aggregated_cache_misses_total) | Counter | Added | `ALPHA` |  |
| [apiserver_authentication_jwt_authenticator_jwks_fetch_last_key_set_info](#apiserver_authentication_jwt_authenticator_jwks_fetch_last_key_set_info) | Custom | Added | `ALPHA` |  |
| [apiserver_authentication_jwt_authenticator_jwks_fetch_last_timestamp_seconds](#apiserver_authentication_jwt_authenticator_jwks_fetch_last_timestamp_seconds) | Gauge | Added | `ALPHA` |  |
| [apiserver_storage_objects](#apiserver_storage_objects) | Gauge | Updated | `STABLE` | Marked as deprecated in version `1.34.0`. |
| [apiserver_validation_declarative_validation_panics_total](#apiserver_validation_declarative_validation_panics_total) | Counter | Added | `ALPHA` |  |
| [apiserver_validation_declarative_validation_parity_discrepancies_total](#apiserver_validation_declarative_validation_parity_discrepancies_total) | Counter | Added | `ALPHA` |  |
| [horizontal_pod_autoscaler_controller_desired_replicas](#horizontal_pod_autoscaler_controller_desired_replicas) | Gauge | Added | `ALPHA` |  |
| [horizontal_pod_autoscaler_controller_num_horizontal_pod_autoscalers](#horizontal_pod_autoscaler_controller_num_horizontal_pod_autoscalers) | Gauge | Added | `ALPHA` |  |
| [kubelet_image_manager_ensure_image_requests_total](#kubelet_image_manager_ensure_image_requests_total) | Counter | Added | `ALPHA` |  |
| [kubelet_image_volume_mounted_errors_total](#kubelet_image_volume_mounted_errors_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [kubelet_image_volume_mounted_succeed_total](#kubelet_image_volume_mounted_succeed_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [kubelet_image_volume_requested_total](#kubelet_image_volume_requested_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [kubelet_imagemanager_image_mustpull_checks_total](#kubelet_imagemanager_image_mustpull_checks_total) | Counter | Added | `ALPHA` |  |
| [kubelet_imagemanager_inmemory_pulledrecords_usage_percent](#kubelet_imagemanager_inmemory_pulledrecords_usage_percent) | Gauge | Added | `ALPHA` |  |
| [kubelet_imagemanager_inmemory_pullintents_usage_percent](#kubelet_imagemanager_inmemory_pullintents_usage_percent) | Gauge | Added | `ALPHA` |  |
| [kubelet_imagemanager_ondisk_pulledrecords](#kubelet_imagemanager_ondisk_pulledrecords) | Gauge | Added | `ALPHA` |  |
| [kubelet_imagemanager_ondisk_pullintents](#kubelet_imagemanager_ondisk_pullintents) | Gauge | Added | `ALPHA` |  |
| [kubelet_podcertificate_states](#kubelet_podcertificate_states) | Custom | Added | `ALPHA` |  |
| [resourceclaim_controller_resource_claims](#resourceclaim_controller_resource_claims) | Custom | Updated | `ALPHA` | Help text changed. <br> Added labels: [`source`]. |
| [rest_client_exec_plugin_policy_call_total](#rest_client_exec_plugin_policy_call_total) | Counter | Added | `ALPHA` |  |
| [scheduler_batch_attempts_total](#scheduler_batch_attempts_total) | Counter | Added | `ALPHA` |  |
| [scheduler_batch_cache_flushed_total](#scheduler_batch_cache_flushed_total) | Counter | Added | `ALPHA` |  |
| [scheduler_get_node_hint_duration_seconds](#scheduler_get_node_hint_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [scheduler_pod_scheduled_after_flush_total](#scheduler_pod_scheduled_after_flush_total) | Counter | Added | `ALPHA` |  |
| [scheduler_resourceclaim_creates_total](#scheduler_resourceclaim_creates_total) | Counter | Added | `ALPHA` |  |
| [scheduler_store_schedule_results_duration_seconds](#scheduler_store_schedule_results_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [statefulset_controller_statefulset_max_unavailable](#statefulset_controller_statefulset_max_unavailable) | Gauge | Added | `ALPHA` |  |
| [statefulset_controller_statefulset_unavailable_replicas](#statefulset_controller_statefulset_unavailable_replicas) | Gauge | Added | `ALPHA` |  |
| [workqueue_adds_total](#workqueue_adds_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [workqueue_depth](#workqueue_depth) | Gauge | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [workqueue_longest_running_processor_seconds](#workqueue_longest_running_processor_seconds) | Gauge | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [workqueue_queue_duration_seconds](#workqueue_queue_duration_seconds) | Histogram | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [workqueue_retries_total](#workqueue_retries_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [workqueue_unfinished_work_seconds](#workqueue_unfinished_work_seconds) | Gauge | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [workqueue_work_duration_seconds](#workqueue_work_duration_seconds) | Histogram | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
## Detailed Changes

### aggregator_discovery_nopeer_requests_total
```diff
+- name: nopeer_requests_total
+  subsystem: aggregator_discovery
+  help: Counter of number of times no-peer (non peer-aggregated) discovery was requested
+  type: Counter
+  stabilityLevel: ALPHA
```

### aggregator_discovery_peer_aggregated_cache_hits_total
```diff
+- name: peer_aggregated_cache_hits_total
+  subsystem: aggregator_discovery
+  help: Counter of number of times discovery was served from peer-aggregated cache
+  type: Counter
+  stabilityLevel: ALPHA
```

### aggregator_discovery_peer_aggregated_cache_misses_total
```diff
+- name: peer_aggregated_cache_misses_total
+  subsystem: aggregator_discovery
+  help: Counter of number of times discovery was aggregated across all API servers
+  type: Counter
+  stabilityLevel: ALPHA
```

### apiserver_authentication_jwt_authenticator_jwks_fetch_last_key_set_info
```diff
+- name: apiserver_authentication_jwt_authenticator_jwks_fetch_last_key_set_info
+  help: Information about the last JWKS fetched by the JWT authenticator with hash as label, split by api server identity and jwt issuer.
+  type: Custom
+  stabilityLevel: ALPHA
+  labels:
+    - jwt_issuer_hash
+    - apiserver_id_hash
+    - hash
```

### apiserver_authentication_jwt_authenticator_jwks_fetch_last_timestamp_seconds
```diff
+- name: jwt_authenticator_jwks_fetch_last_timestamp_seconds
+  subsystem: authentication
+  namespace: apiserver
+  help: Timestamp of the last successful or failed JWKS fetch split by result, api server identity and jwt issuer for the JWT authenticator.
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - apiserver_id_hash
+    - jwt_issuer_hash
+    - result
```

### apiserver_storage_objects
```diff
 - name: apiserver_storage_objects
   help: '[DEPRECATED, consider using apiserver_resource_objects instead] Number of stored objects at the time of last check split by kind. In case of a fetching error, the value will be -1.'
   type: Gauge
+  deprecatedVersion: 1.34.0
   stabilityLevel: STABLE
   labels:
     - resource
```

### apiserver_validation_declarative_validation_panics_total
```diff
+- name: declarative_validation_panics_total
+  subsystem: validation
+  namespace: apiserver
+  help: Number of panics in declarative validation, broken down by validation identifier.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - validation_identifier
```

### apiserver_validation_declarative_validation_parity_discrepancies_total
```diff
+- name: declarative_validation_parity_discrepancies_total
+  subsystem: validation
+  namespace: apiserver
+  help: Number of discrepancies between declarative and handwritten validation, broken down by validation identifier.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - validation_identifier
```

### horizontal_pod_autoscaler_controller_desired_replicas
```diff
+- name: desired_replicas
+  subsystem: horizontal_pod_autoscaler_controller
+  help: Current desired replica count for HPA objects.
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - hpa_name
+    - namespace
```

### horizontal_pod_autoscaler_controller_num_horizontal_pod_autoscalers
```diff
+- name: num_horizontal_pod_autoscalers
+  subsystem: horizontal_pod_autoscaler_controller
+  help: Current number of controlled HPA objects.
+  type: Gauge
+  stabilityLevel: ALPHA
```

### kubelet_image_manager_ensure_image_requests_total
```diff
+- name: image_manager_ensure_image_requests_total
+  subsystem: kubelet
+  help: Number of ensure-image requests processed by the kubelet.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - present_locally
+    - pull_policy
+    - pull_required
```

### kubelet_image_volume_mounted_errors_total
```diff
 - name: image_volume_mounted_errors_total
   subsystem: kubelet
   help: Number of failed image volume mounts.
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
```

### kubelet_image_volume_mounted_succeed_total
```diff
 - name: image_volume_mounted_succeed_total
   subsystem: kubelet
   help: Number of successful image volume mounts.
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
```

### kubelet_image_volume_requested_total
```diff
 - name: image_volume_requested_total
   subsystem: kubelet
   help: Number of requested image volumes.
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
```

### kubelet_imagemanager_image_mustpull_checks_total
```diff
+- name: imagemanager_image_mustpull_checks_total
+  subsystem: kubelet
+  help: Counter for how many times kubelet checked whether credentials need to be re-verified to access an image
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - result
```

### kubelet_imagemanager_inmemory_pulledrecords_usage_percent
```diff
+- name: imagemanager_inmemory_pulledrecords_usage_percent
+  subsystem: kubelet
+  help: The ImagePulledRecords in-memory cache usage in percent.
+  type: Gauge
+  stabilityLevel: ALPHA
```

### kubelet_imagemanager_inmemory_pullintents_usage_percent
```diff
+- name: imagemanager_inmemory_pullintents_usage_percent
+  subsystem: kubelet
+  help: The ImagePullIntents in-memory cache usage in percent.
+  type: Gauge
+  stabilityLevel: ALPHA
```

### kubelet_imagemanager_ondisk_pulledrecords
```diff
+- name: imagemanager_ondisk_pulledrecords
+  subsystem: kubelet
+  help: Number of ImagePulledRecords stored on disk.
+  type: Gauge
+  stabilityLevel: ALPHA
```

### kubelet_imagemanager_ondisk_pullintents
```diff
+- name: imagemanager_ondisk_pullintents
+  subsystem: kubelet
+  help: Number of ImagePullIntents stored on disk.
+  type: Gauge
+  stabilityLevel: ALPHA
```

### kubelet_podcertificate_states
```diff
+- name: kubelet_podcertificate_states
+  help: Gauge vector reporting the number of pod certificate projected volume sources, faceted by signer_name and state.
+  type: Custom
+  stabilityLevel: ALPHA
+  labels:
+    - signer_name
+    - state
```

### resourceclaim_controller_resource_claims
```diff
 - name: resourceclaim_controller_resource_claims
-  help: Number of ResourceClaims, categorized by allocation status and admin access
+  help: Number of ResourceClaims, categorized by allocation status, admin access, and source. Source can be 'resource_claim_template' (created from a template), 'extended_resource' (extended resources), or empty (manually created by a user).
   type: Custom
   stabilityLevel: ALPHA
   labels:
     - allocated
     - admin_access
+    - source
```

### rest_client_exec_plugin_policy_call_total
```diff
+- name: rest_client_exec_plugin_policy_call_total
+  help: Number of comparisons of an exec plugin to the plugin policy and allowlist (if any), partitioned by whether or not the policy permits the plugin
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - allowed
+    - denied
```

### scheduler_batch_attempts_total
```diff
+- name: batch_attempts_total
+  subsystem: scheduler
+  help: Counts of results when we attempt to use batching.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - profile
+    - result
```

### scheduler_batch_cache_flushed_total
```diff
+- name: batch_cache_flushed_total
+  subsystem: scheduler
+  help: Counts of cache flushes by reason.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - profile
+    - reason
```

### scheduler_get_node_hint_duration_seconds
```diff
+- name: get_node_hint_duration_seconds
+  subsystem: scheduler
+  help: Latency for getting a node hint.
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - hinted
+    - profile
+  buckets:
+    - 1e-05
+    - 2e-05
+    - 4e-05
+    - 8e-05
+    - 0.00016
+    - 0.00032
+    - 0.00064
+    - 0.00128
+    - 0.00256
+    - 0.00512
+    - 0.01024
+    - 0.02048
```

### scheduler_pod_scheduled_after_flush_total
```diff
+- name: pod_scheduled_after_flush_total
+  subsystem: scheduler
+  help: Number of pods that were successfully scheduled after being flushed from unschedulablePods due to timeout. This metric helps detect potential queueing hint misconfigurations or event handling issues.
+  type: Counter
+  stabilityLevel: ALPHA
```

### scheduler_resourceclaim_creates_total
```diff
+- name: resourceclaim_creates_total
+  subsystem: scheduler
+  help: Number of ResourceClaims creation requests within scheduler
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - status
```

### scheduler_store_schedule_results_duration_seconds
```diff
+- name: store_schedule_results_duration_seconds
+  subsystem: scheduler
+  help: Latency for getting a no.
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - profile
+  buckets:
+    - 1e-05
+    - 2e-05
+    - 4e-05
+    - 8e-05
+    - 0.00016
+    - 0.00032
+    - 0.00064
+    - 0.00128
+    - 0.00256
+    - 0.00512
+    - 0.01024
+    - 0.02048
```

### statefulset_controller_statefulset_max_unavailable
```diff
+- name: statefulset_max_unavailable
+  subsystem: statefulset_controller
+  help: Maximum number of unavailable pods allowed during StatefulSet rolling updates
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - pod_management_policy
+    - statefulset_name
+    - statefulset_namespace
```

### statefulset_controller_statefulset_unavailable_replicas
```diff
+- name: statefulset_unavailable_replicas
+  subsystem: statefulset_controller
+  help: Current number of unavailable pods in StatefulSet
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - pod_management_policy
+    - statefulset_name
+    - statefulset_namespace
```

### workqueue_adds_total
```diff
 - name: adds_total
   subsystem: workqueue
   help: Total number of adds handled by workqueue
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - name
```

### workqueue_depth
```diff
 - name: depth
   subsystem: workqueue
   help: Current depth of workqueue
   type: Gauge
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - name
```

### workqueue_longest_running_processor_seconds
```diff
 - name: longest_running_processor_seconds
   subsystem: workqueue
   help: How many seconds has the longest running processor for workqueue been running.
   type: Gauge
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - name
```

### workqueue_queue_duration_seconds
```diff
 - name: queue_duration_seconds
   subsystem: workqueue
   help: How long in seconds an item stays in workqueue before being requested.
   type: Histogram
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - name
   buckets:
     - 1e-08
     - 1e-07
     - 1e-06
     - 9.999999999999999e-06
     - 9.999999999999999e-05
     - 0.001
     - 0.01
     - 0.1
     - 1
     - 10
```

### workqueue_retries_total
```diff
 - name: retries_total
   subsystem: workqueue
   help: Total number of retries handled by workqueue
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - name
```

### workqueue_unfinished_work_seconds
```diff
 - name: unfinished_work_seconds
   subsystem: workqueue
   help: How many seconds of work has done that is in progress and hasn't been observed by work_duration. Large values indicate stuck threads. One can deduce the number of stuck threads by observing the rate at which this increases.
   type: Gauge
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - name
```

### workqueue_work_duration_seconds
```diff
 - name: work_duration_seconds
   subsystem: workqueue
   help: How long in seconds processing an item from workqueue takes.
   type: Histogram
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - name
   buckets:
     - 1e-08
     - 1e-07
     - 1e-06
     - 9.999999999999999e-06
     - 9.999999999999999e-05
     - 0.001
     - 0.01
     - 0.1
     - 1
     - 10
```
