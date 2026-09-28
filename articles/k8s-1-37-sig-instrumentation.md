---
title: "Kubernetes 1.37: SIG-Instrumentationの変更内容" # 記事のタイトル
emoji: "☸️" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["kubernetes"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---

# はじめに
本記事では Kubernetes v1.37 の [Changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.37.md) から、SIG-Instrumentation 関連及びメトリクスの変更点について取り上げ、まとめています。

* [Kubernetes 1.35: SIG-Instrumentationの変更内容](https://zenn.dev/tsuzu/articles/k8s-1-35-sig-instrumentation)
* [Kubernetes 1.36: SIG-Instrumentationの変更内容](https://zenn.dev/tsuzu/articles/k8s-1-36-sig-instrumentation)

# Changes by kind
## Upgrade Notes

- kubeletに組み込まれている cAdvisor が軽量な `github.com/google/cadvisor/lib` モジュールへ移行しました。これに伴い、長期間非推奨だった cAdvisor のフラグが受け付けられなくなり、`userDefinedMetrics`、`container_application_*`、`container_cpu_load_average_10s`、`container_cpu_load_d_average_10s`、`container_tasks_state` などが提供されなくなりました。既存の kubelet 設定や監視ルールを確認する必要があります。 ([#139870](https://github.com/kubernetes/kubernetes/pull/139870))

- cAdvisorではなくCRIからPod・コンテナの統計情報を取得する `PodAndContainerStatsFromCRI` がBetaに昇格しました。デフォルトでは無効のため、利用する場合はFeature gateを有効化します。 ([#140081](https://github.com/kubernetes/kubernetes/pull/140081))

## Deprecation

- `apiserver_cache_list_total`、`apiserver_cache_list_fetched_objects_total`、`apiserver_cache_list_returned_objects_total` がdeprecatedになりました。Watch cache経由のLISTに関する情報は、`storage="watchcache"` を指定した `apiserver_storage_list_*` メトリクスへの移行が推奨されています。 ([#139154](https://github.com/kubernetes/kubernetes/pull/139154))

- `scheduler_preemption_goroutines_duration_seconds` がdeprecatedになりました。詳細は後述のメトリクス差分に記載しています。
  * `preemption_execution_duration_seconds` を利用してください。

## API Changes

- `metrics.k8s.io` APIが `v1beta1` から `v1` に昇格しました。APIのフィールドや返却内容に変更はなく、API versionの昇格です。既存の `v1beta1` も引き続き利用できます。 ([#139223](https://github.com/kubernetes/kubernetes/pull/139223))

- `kubectl top` が `metrics.k8s.io/v1` に対応しました。メトリクスサーバーなどの実装が `v1` を提供している場合、`kubectl top` から利用できます。 ([#139726](https://github.com/kubernetes/kubernetes/pull/139726))
  - metrics-server側では [`metrics.k8s.io/v1` のAPIサポート](https://github.com/kubernetes-sigs/metrics-server/pull/1855) が実装され、2026年9月25日にmasterへマージされました。
    - ただし、本記事執筆時点（2026年9月26日）のmetrics-server本体の最新リリースは `v0.9.0` です。公式の互換性表では `0.9.x` が提供するMetrics APIは `metrics.k8s.io/v1beta1` と記載されており、PR #1855の `v1` 対応を含むリリースはまだ公開されていません。 ([リリース一覧](https://github.com/kubernetes-sigs/metrics-server/releases), [互換性表](https://github.com/kubernetes-sigs/metrics-server#compatibility-matrix))
    - そのため、Kubernetes 1.37側で `metrics.k8s.io/v1` と `kubectl top` がサポートされても、metrics-serverのリリース版が `v1` を提供するまでは、クラスタ上で `metrics.k8s.io/v1` のエンドポイントを利用できない点に注意が必要です。

## Features

- Prometheus Native HistogramsのFeature gateがBetaに昇格しました。従来のclassic histogramとのdual expositionにより、既存のメトリクス利用者への互換性を保ちながらnative histogramを利用できます。 ([#140124](https://github.com/kubernetes/kubernetes/pull/140124))
  - Native Histogramについては [こちらの記事](https://qiita.com/yosshi_/items/482a4dcc8ce08e19318e) が詳しいのでぜひご覧ください。
  * 1.37時点では classic histogram と native histogram の両方が公開されます。
  * 動的に適切なbucket分割が自動で行われるようになるためより正確なhistogramを見ることができるようになります。
  * https://kubernetes.io/docs/reference/instrumentation/native-histograms/

- API serverのLISTおよびwatch cacheを観測するメトリクスが拡充されました。
  - `apiserver_storage_list_duration_seconds` でetcdの読み出しからオブジェクトのdecodeまでのLISTレイテンシを計測できます。 ([#140697](https://github.com/kubernetes/kubernetes/pull/140697))
  - `apiserver_storage_list_*` に `storage` と `index` ラベルが追加され、利用したストレージバックエンドと検索経路を区別できます。 ([#139125](https://github.com/kubernetes/kubernetes/pull/139125))
    * 1つのmetricでcacheから読んだか、etcdから読んだかを区別できるようになります。
  - `apiserver_watch_cache_initialization_duration_seconds` でwatch cacheの初期化時間を確認できます。 ([#138767](https://github.com/kubernetes/kubernetes/pull/138767))
  - `apiserver_watch_events_dispatch_duration_seconds` と `storage_to_cache`、`cache_to_watcher` のstageにより、watch eventのdecodeからwatcherへの配送までを分解して確認できます。 ([#140336](https://github.com/kubernetes/kubernetes/pull/140336), [#140851](https://github.com/kubernetes/kubernetes/pull/140851), [#140860](https://github.com/kubernetes/kubernetes/pull/140860))
  - `watch_list_duration_seconds` に90秒、120秒、180秒、300秒のbucketが追加されました。 ([#140757](https://github.com/kubernetes/kubernetes/pull/140757))
  - ( API Serverのlatency系metricsが沢山増えたのでv1.37時点での対応関係を可視化するページをClaudeと作りました https://claude.ai/artifact/8wRmCdbFPSvmjeDDy398vz ) 

- kube-apiserverでinformer activityのメトリクスが利用できるようになりました。さらにkube-schedulerでも、`name="kube-scheduler"` のラベル付きで `informer_store_resource_version`、`informer_queued_items`、`informer_processing_latency_seconds` が公開されます。 ([#139968](https://github.com/kubernetes/kubernetes/pull/139968), [#140511](https://github.com/kubernetes/kubernetes/pull/140511))

- schedulerのメトリクスが拡充されました。
  - Topology-aware schedulingのplacement phase向けに、`scheduler_generated_placements_total`、`scheduler_placement_evaluations_total`、`scheduler_placement_evaluation_duration_seconds` が追加されました。 ([#139604](https://github.com/kubernetes/kubernetes/pull/139604))
  - Workload-aware preemption向けのメトリクスがAlphaとして追加されました。 ([#139373](https://github.com/kubernetes/kubernetes/pull/139373))
  - scheduling queueにあるPodまたはPodGroupの数を示す `scheduler_queued_entities` と、queueに追加された総数を示す `scheduler_queue_incoming_entities_total` が追加されました。 ([#139840](https://github.com/kubernetes/kubernetes/pull/139840))
  - `SchedulerPreQueueingHints` に対応する `scheduler_pre_queueing_hint_evaluations_total` などが追加されました。PreQueueingHint拡張点と関連メトリクスは、この機能を追加したPRで導入されています。 ([#138916](https://github.com/kubernetes/kubernetes/pull/138916))
  - その後、既知の問題を受けて `SchedulerPreQueueingHints` をBetaからAlphaへ変更した経緯は、[SchedulerPreQueueingHintsをBetaからAlphaへ変更したPR #140959](https://github.com/kubernetes/kubernetes/pull/140959)で説明されています。

- kubeletのPod resizeとPod-Level Resourcesに関するメトリクスが追加・更新されました。
  - `kubelet_pod_deferred_resize_duration_seconds` が追加され、`kubelet_pod_pending_resizes` に `priority_bucket` ラベルが追加されました。 ([#140122](https://github.com/kubernetes/kubernetes/pull/140122))
  - `kubelet_pod_level_resources_admission_total` が追加され、Pod admission時のPod-Level Resourcesの利用状況をresource configuration modeとQoS class別に確認できます。 ([#140463](https://github.com/kubernetes/kubernetes/pull/140463))

- Windows kube-proxyのロードバランサー操作失敗を示すメトリクスが追加されました。 `ip_family`、`lb_type`、`error` ラベルで失敗内容を区別できます。 ([#137767](https://github.com/kubernetes/kubernetes/pull/137767))
  - `kubeproxy_sync_proxy_rules_winkernel_lb_create_failures_total`
  - `kubeproxy_sync_proxy_rules_winkernel_lb_update_failures_total`
  - `kubeproxy_sync_proxy_rules_winkernel_lb_delete_failures_total`

- route controllerの `route_controller_route_sync_total` に `trigger` と `outcome` ラベルが追加され、`periodic` / `node_change` のどちらで起動したか、`changed` / `noop` / `error` のどの結果になったかを区別できるようになりました。 ([#140147](https://github.com/kubernetes/kubernetes/pull/140147), [#140824](https://github.com/kubernetes/kubernetes/pull/140824))

- `scheduler_plugin_execution_duration_seconds`、`scheduler_scheduling_algorithm_duration_seconds` がAlphaからBetaに昇格しました。また、`apiserver_watch_events_total`、`apiserver_watch_events_sizes`、kubeletの `storage_operation_duration_seconds`、`volume_operation_total_seconds`、service account関連の3メトリクス、webhook関連の2メトリクスもBetaに昇格しました。 ([#138176](https://github.com/kubernetes/kubernetes/pull/138176), [#137116](https://github.com/kubernetes/kubernetes/pull/137116), [#136189](https://github.com/kubernetes/kubernetes/pull/136189), [#137072](https://github.com/kubernetes/kubernetes/pull/137072), [#136894](https://github.com/kubernetes/kubernetes/pull/136894))

### API server・controller・ストレージ

- API serverのLIST・watch cache関連では、既存メトリクスの観測範囲も広がっています。
  - `apiserver_storage_list_evaluated_objects_total`、`apiserver_storage_list_fetched_objects_total`、`apiserver_storage_list_returned_objects_total`、`apiserver_storage_list_total` に `storage` ラベルが追加され、fetched / returned / evaluated の内訳をストレージ種別ごとに比較できるようになりました。`fetched` と `total` には `index` ラベルも追加されています。 ([#139125](https://github.com/kubernetes/kubernetes/pull/139125))
  - `apiserver_watch_cache_initialization_errors_total` が追加され、リソース種別ごとのwatch cache初期化失敗をカウントできるようになりました。
  - `apiserver_webhooks_x509_insecure_sha1_total` と `apiserver_webhooks_x509_missing_san_total` はBetaに昇格しました。webhookの証明書互換性問題を監視するメトリクスとして、Alpha前提の監視から安定性の高い監視へ移行できます。 ([#136894](https://github.com/kubernetes/kubernetes/pull/136894))

- controller・cloud provider・ストレージの状態を観測するメトリクスも追加されました。
  - `cloud_node_lifecycle_controller_cloud_provider_calls_total` はcloud provider APIの呼び出しをoperationとresult別に、`cloud_node_lifecycle_controller_monitor_nodes_duration_seconds` はNode監視ループの実行時間を記録します。
  - `horizontal_pod_autoscaler_controller_stale_sync_skips_total` により、watch cacheが古いためHPAのsyncを延期した回数を確認できます。1.36で導入されたstale controller mitigationの対象がHPAにも広がった形です。
  - `csi_node_storage_health_status` はCSI driverごとのnode側ストレージ状態を、`driver_name`、`reason`、`status` ラベルで公開します。Volume Healthを監視する際に利用できます。

### DRA

- DRAのメトリクスは、ResourceClaimの作成元と所有者を区別しやすい形に整理されました。
  - `dra_node_prepare_skips_total` と `dra_node_unprepare_skips_total` は、`SkipNodeOperations` によってNodePrepareResources / NodeUnprepareResourcesを省略した回数をdriverごとに記録します。
  - `dynamic_resource_allocation_resource_claims` はResourceClaimの割り当て状態・admin access・sourceを公開します。sourceはResourceClaimTemplate、extended resource、手動作成を区別します。
  - `dynamic_resource_allocation_resourceclaim_creates_total` はcontrollerとschedulerで共通のメトリクスになり、`owner_api_group` と `owner_api_kind` によってPodとPodGroupを区別できます。既存の `resourceclaim_controller_creates_total`、`resourceclaim_controller_resource_claims`、`scheduler_resourceclaim_creates_total` は削除対象です。

### kubelet・kube-proxy

- kubeletのPodsAPIとexclusive resource allocationの利用状況が観測できるようになりました。
  - `kubelet_pod_requests_total`、`kubelet_pod_requests_get_total`、`kubelet_pod_requests_list_total`、`kubelet_pod_requests_watch_total` は、PodsAPIへのリクエストをserver API versionとHTTP status code別に記録します。
  - `kubelet_resource_manager_allocations_total`、`kubelet_resource_manager_allocation_errors_total`、`kubelet_resource_manager_container_assignments_total` により、resource managerの割り当て成功・失敗とcontainerへのassignment typeを確認できます。旧名の `resource_manager_*` は削除されているため、ダッシュボードやアラートのメトリクス名を更新する必要があります。

- nftables kube-proxyのlocalhost NodePort userspace proxy向けメトリクスが追加されました。
  - `kubeproxy_nftables_localhost_nodeport_listeners` でactiveなlistener数を、`kubeproxy_nftables_localhost_nodeport_listener_creation_failures_total` でlistener作成失敗を確認できます。
  - `kubeproxy_nftables_localhost_nodeport_rejected_packets_total` は、IP familyとprotocol別にlocalhost NodePortで拒否されたpacket数を記録します。localhost NodePortを利用する場合は、proxyが動作していない状態やUDP/SCTPの拒否を切り分けるのに使えます。 ([#138427](https://github.com/kubernetes/kubernetes/pull/138427))

- schedulerではPodGroup・preemption・opportunistic batchingの観測対象が増えました。
  - `scheduler_batch_rescore_attempts_total` と `scheduler_batch_rescore_duration_seconds` により、opportunistic batchingでnodeをrescoreした回数とレイテンシを確認できます。
  - `scheduler_preemption_evaluation_duration_seconds`、`scheduler_preemption_execution_duration_seconds`、`scheduler_preemption_pdb_violations_total` は、preemption対象の評価・実行時間とPDB違反を記録します。
  - `scheduler_preemption_workload_disruptions` と `scheduler_workload_preemption_attempts_total`、`scheduler_workload_preemption_victims` は、Pod単位のpreemptionとWorkload / PodGroup単位のpreemptionを分けて観測するためのメトリクスです。
  - Stableの `scheduler_pending_pods` はhelpが更新され、PodGroup schedulingに対応して `incomplete` や `pending` のqueue状態も説明されるようになりました。`scheduler_preemption_victims` も単一Pod起点のpreemptionを表すことが明記されています。

- 既存メトリクスの説明だけが更新されたものもあります。`replicaset_controller_sorting_deletion_age_ratio` から古いFeature gateへの言及が削除され、`scheduler_pod_scheduled_after_flush_total` などではPodだけでなくscheduling entityを扱う現在の実装に合わせてhelpが更新されました。メトリクス名や型が変わらなくても、運用者向けドキュメントやダッシュボードの説明を確認しておく必要があります。

## Documentation

- kube-proxyの `--metrics-bind-address` のドキュメントから、空文字を指定するとmetrics serverが無効になるという誤った説明が削除されました。 ([#138940](https://github.com/kubernetes/kubernetes/pull/138940))

## Bug or Regression

- schedulerのPriorityQueueでPod削除時に `UnschedulableReason` メトリクスが減算されず、値が蓄積する問題が修正されました。 ([#138482](https://github.com/kubernetes/kubernetes/pull/138482))

- StatefulSet controllerのskipメトリクスが正しくregisterされない問題が修正されました。 ([#138451](https://github.com/kubernetes/kubernetes/pull/138451))

- SELinuxのメトリクス出力で不要な比較を行わないようになり、メトリクス出力時のコストが削減されました。 ([#138981](https://github.com/kubernetes/kubernetes/pull/138981))

- API serverのLISTレイテンシを示す `apiserver_storage_list_duration_seconds` が追加されました。etcd RangeStreamの利用有無もラベルで区別できるため、streamed / non-streamed LISTの比較に利用できます。 ([#140697](https://github.com/kubernetes/kubernetes/pull/140697))

## Other (Cleanup or Flake)

- 非推奨の `apiserver_cache_list_*` メトリクスがデフォルトでは公開されなくなりました。Watch cacheを利用したLISTについては、`storage="watchcache"` の `apiserver_storage_list_*` へ移行します。 ([#139154](https://github.com/kubernetes/kubernetes/pull/139154))

- v1.37で削除されたメトリクスは次の15個です。監視ルールやダッシュボードで利用している場合は、削除理由と移行先を確認してください。
  - API server: `apiserver_flowcontrol_request_concurrency_in_use`、`apiserver_flowcontrol_request_concurrency_limit`、`apiserver_request_slo_duration_seconds`、`apiserver_storage_db_total_size_in_bytes`
  - etcd / kube-apiserver: `etcd_version_info`、`kube_apiserver_pod_logs_pods_logs_backend_tls_failure_total`、`kube_apiserver_pod_logs_pods_logs_insecure_backend_total`
  - kubelet / resource manager: `kubelet_volume_stats_health_status_abnormal`、`resource_manager_allocation_errors_total`、`resource_manager_allocations_total`、`resource_manager_container_assignments`
  - その他: `latency`、`resourceclaim_controller_creates_total`、`resourceclaim_controller_resource_claims`、`scheduler_resourceclaim_creates_total`
  - ResourceClaim関連は前述の `dynamic_resource_allocation_*` へ統合されていますが、それ以外は一律の置換先があるとは限りません。削除されたメトリクスを参照するPromQLの有無を確認するのが安全です。

- service account関連の `serviceaccount_legacy_tokens_total`、`serviceaccount_stale_tokens_total`、`serviceaccount_valid_tokens_total` はBetaに昇格しました。Alphaのため許容していたメトリクス名・ラベル変更の余地が小さくなり、長期的なtoken利用状況の監視に使いやすくなっています。 ([#137072](https://github.com/kubernetes/kubernetes/pull/137072))

# Kubernetes Metrics Changes: v1.36.0 → v1.37.0

## Summary
- **Added**: 41 metrics
- **Removed**: 15 metrics
- **Updated**: 28 metrics
- **Total Changes**: 84 metrics

## Changed Metrics

| Metric Name | Type | Change Type | Stability Level | Description |
|-------------|------|-------------|----------------|-------------|
| [apiserver_cache_list_fetched_objects_total](#apiserver_cache_list_fetched_objects_total) | Counter | Updated | `ALPHA` | Marked as deprecated in version `1.37.0`. |
| [apiserver_cache_list_returned_objects_total](#apiserver_cache_list_returned_objects_total) | Counter | Updated | `ALPHA` | Marked as deprecated in version `1.37.0`. |
| [apiserver_cache_list_total](#apiserver_cache_list_total) | Counter | Updated | `ALPHA` | Marked as deprecated in version `1.37.0`. |
| [apiserver_flowcontrol_request_concurrency_in_use](#apiserver_flowcontrol_request_concurrency_in_use) | Gauge | Removed | `ALPHA` |  |
| [apiserver_flowcontrol_request_concurrency_limit](#apiserver_flowcontrol_request_concurrency_limit) | Gauge | Removed | `ALPHA` |  |
| [apiserver_request_slo_duration_seconds](#apiserver_request_slo_duration_seconds) | Histogram | Removed | `ALPHA` |  |
| [apiserver_storage_db_total_size_in_bytes](#apiserver_storage_db_total_size_in_bytes) | Gauge | Removed | `ALPHA` |  |
| [apiserver_storage_list_duration_seconds](#apiserver_storage_list_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [apiserver_storage_list_evaluated_objects_total](#apiserver_storage_list_evaluated_objects_total) | Counter | Updated | `ALPHA` | Added labels: [`storage`]. |
| [apiserver_storage_list_fetched_objects_total](#apiserver_storage_list_fetched_objects_total) | Counter | Updated | `ALPHA` | Added labels: [`index`, `storage`]. |
| [apiserver_storage_list_returned_objects_total](#apiserver_storage_list_returned_objects_total) | Counter | Updated | `ALPHA` | Added labels: [`storage`]. |
| [apiserver_storage_list_total](#apiserver_storage_list_total) | Counter | Updated | `ALPHA` | Added labels: [`index`, `storage`]. |
| [apiserver_watch_cache_initialization_duration_seconds](#apiserver_watch_cache_initialization_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [apiserver_watch_cache_initialization_errors_total](#apiserver_watch_cache_initialization_errors_total) | Counter | Added | `ALPHA` |  |
| [apiserver_watch_events_dispatch_duration_seconds](#apiserver_watch_events_dispatch_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [apiserver_watch_events_sizes](#apiserver_watch_events_sizes) | Histogram | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [apiserver_watch_events_total](#apiserver_watch_events_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [apiserver_watch_list_duration_seconds](#apiserver_watch_list_duration_seconds) | Histogram | Updated | `BETA` | Buckets changed. |
| [apiserver_webhooks_x509_insecure_sha1_total](#apiserver_webhooks_x509_insecure_sha1_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [apiserver_webhooks_x509_missing_san_total](#apiserver_webhooks_x509_missing_san_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [cloud_node_lifecycle_controller_cloud_provider_calls_total](#cloud_node_lifecycle_controller_cloud_provider_calls_total) | Counter | Added | `ALPHA` |  |
| [cloud_node_lifecycle_controller_monitor_nodes_duration_seconds](#cloud_node_lifecycle_controller_monitor_nodes_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [csi_node_storage_health_status](#csi_node_storage_health_status) | Gauge | Added | `ALPHA` |  |
| [dra_node_prepare_skips_total](#dra_node_prepare_skips_total) | Counter | Added | `ALPHA` |  |
| [dra_node_unprepare_skips_total](#dra_node_unprepare_skips_total) | Counter | Added | `ALPHA` |  |
| [dynamic_resource_allocation_resource_claims](#dynamic_resource_allocation_resource_claims) | Custom | Added | `ALPHA` |  |
| [dynamic_resource_allocation_resourceclaim_creates_total](#dynamic_resource_allocation_resourceclaim_creates_total) | Counter | Added | `ALPHA` |  |
| [etcd_version_info](#etcd_version_info) | Gauge | Removed | `ALPHA` |  |
| [horizontal_pod_autoscaler_controller_stale_sync_skips_total](#horizontal_pod_autoscaler_controller_stale_sync_skips_total) | Counter | Added | `ALPHA` |  |
| [kube_apiserver_pod_logs_pods_logs_backend_tls_failure_total](#kube_apiserver_pod_logs_pods_logs_backend_tls_failure_total) | Counter | Removed | `ALPHA` |  |
| [kube_apiserver_pod_logs_pods_logs_insecure_backend_total](#kube_apiserver_pod_logs_pods_logs_insecure_backend_total) | Counter | Removed | `ALPHA` |  |
| [kubelet_pod_deferred_resize_duration_seconds](#kubelet_pod_deferred_resize_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [kubelet_pod_level_resources_admission_total](#kubelet_pod_level_resources_admission_total) | Counter | Added | `ALPHA` |  |
| [kubelet_pod_pending_resizes](#kubelet_pod_pending_resizes) | Gauge | Updated | `ALPHA` | Help text changed. <br> Added labels: [`priority_bucket`]. |
| [kubelet_pod_requests_get_total](#kubelet_pod_requests_get_total) | Counter | Added | `ALPHA` |  |
| [kubelet_pod_requests_list_total](#kubelet_pod_requests_list_total) | Counter | Added | `ALPHA` |  |
| [kubelet_pod_requests_total](#kubelet_pod_requests_total) | Counter | Added | `ALPHA` |  |
| [kubelet_pod_requests_watch_total](#kubelet_pod_requests_watch_total) | Counter | Added | `ALPHA` |  |
| [kubelet_resource_manager_allocation_errors_total](#kubelet_resource_manager_allocation_errors_total) | Counter | Added | `ALPHA` |  |
| [kubelet_resource_manager_allocations_total](#kubelet_resource_manager_allocations_total) | Counter | Added | `ALPHA` |  |
| [kubelet_resource_manager_container_assignments_total](#kubelet_resource_manager_container_assignments_total) | Counter | Added | `ALPHA` |  |
| [kubelet_volume_stats_health_status_abnormal](#kubelet_volume_stats_health_status_abnormal) | Custom | Removed | `ALPHA` |  |
| [kubeproxy_nftables_localhost_nodeport_listener_creation_failures_total](#kubeproxy_nftables_localhost_nodeport_listener_creation_failures_total) | Counter | Added | `ALPHA` |  |
| [kubeproxy_nftables_localhost_nodeport_listeners](#kubeproxy_nftables_localhost_nodeport_listeners) | Gauge | Added | `ALPHA` |  |
| [kubeproxy_nftables_localhost_nodeport_rejected_packets_total](#kubeproxy_nftables_localhost_nodeport_rejected_packets_total) | Custom | Added | `ALPHA` |  |
| [kubeproxy_sync_proxy_rules_winkernel_lb_create_failures_total](#kubeproxy_sync_proxy_rules_winkernel_lb_create_failures_total) | Counter | Added | `ALPHA` |  |
| [kubeproxy_sync_proxy_rules_winkernel_lb_delete_failures_total](#kubeproxy_sync_proxy_rules_winkernel_lb_delete_failures_total) | Counter | Added | `ALPHA` |  |
| [kubeproxy_sync_proxy_rules_winkernel_lb_update_failures_total](#kubeproxy_sync_proxy_rules_winkernel_lb_update_failures_total) | Counter | Added | `ALPHA` |  |
| [latency](#latency) | Summary | Removed | `ALPHA` |  |
| [replicaset_controller_sorting_deletion_age_ratio](#replicaset_controller_sorting_deletion_age_ratio) | Histogram | Updated | `ALPHA` | Help text changed. |
| [resource_manager_allocation_errors_total](#resource_manager_allocation_errors_total) | Counter | Removed | `ALPHA` |  |
| [resource_manager_allocations_total](#resource_manager_allocations_total) | Counter | Removed | `ALPHA` |  |
| [resource_manager_container_assignments](#resource_manager_container_assignments) | Counter | Removed | `ALPHA` |  |
| [resourceclaim_controller_creates_total](#resourceclaim_controller_creates_total) | Counter | Removed | `ALPHA` |  |
| [resourceclaim_controller_resource_claims](#resourceclaim_controller_resource_claims) | Custom | Removed | `ALPHA` |  |
| [route_controller_route_sync_total](#route_controller_route_sync_total) | Counter | Updated | `ALPHA` | Help text changed. <br> Added labels: [`outcome`, `trigger`]. |
| [scheduler_batch_rescore_attempts_total](#scheduler_batch_rescore_attempts_total) | Counter | Added | `ALPHA` |  |
| [scheduler_batch_rescore_duration_seconds](#scheduler_batch_rescore_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [scheduler_generated_placements_total](#scheduler_generated_placements_total) | Counter | Added | `ALPHA` |  |
| [scheduler_pending_pods](#scheduler_pending_pods) | Gauge | Updated | `STABLE` | Help text changed. |
| [scheduler_placement_evaluation_duration_seconds](#scheduler_placement_evaluation_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [scheduler_placement_evaluations_total](#scheduler_placement_evaluations_total) | Counter | Added | `ALPHA` |  |
| [scheduler_plugin_execution_duration_seconds](#scheduler_plugin_execution_duration_seconds) | Histogram | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [scheduler_pod_scheduled_after_flush_total](#scheduler_pod_scheduled_after_flush_total) | Counter | Updated | `ALPHA` | Help text changed. |
| [scheduler_podgroup_scheduling_attempt_duration_seconds](#scheduler_podgroup_scheduling_attempt_duration_seconds) | Histogram | Updated | `ALPHA` | Help text changed. |
| [scheduler_pre_queueing_hint_evaluations_total](#scheduler_pre_queueing_hint_evaluations_total) | Counter | Added | `ALPHA` |  |
| [scheduler_preemption_evaluation_duration_seconds](#scheduler_preemption_evaluation_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [scheduler_preemption_execution_duration_seconds](#scheduler_preemption_execution_duration_seconds) | Histogram | Added | `ALPHA` |  |
| [scheduler_preemption_goroutines_duration_seconds](#scheduler_preemption_goroutines_duration_seconds) | Histogram | Updated | `ALPHA` | Marked as deprecated in version `1.37.0`. |
| [scheduler_preemption_pdb_violations_total](#scheduler_preemption_pdb_violations_total) | Counter | Added | `ALPHA` |  |
| [scheduler_preemption_victims](#scheduler_preemption_victims) | Histogram | Updated | `STABLE` | Help text changed. |
| [scheduler_preemption_workload_disruptions](#scheduler_preemption_workload_disruptions) | Histogram | Added | `ALPHA` |  |
| [scheduler_queue_incoming_entities_total](#scheduler_queue_incoming_entities_total) | Counter | Added | `ALPHA` |  |
| [scheduler_queued_entities](#scheduler_queued_entities) | Gauge | Added | `ALPHA` |  |
| [scheduler_resourceclaim_creates_total](#scheduler_resourceclaim_creates_total) | Counter | Removed | `ALPHA` |  |
| [scheduler_scheduling_algorithm_duration_seconds](#scheduler_scheduling_algorithm_duration_seconds) | Histogram | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [scheduler_store_schedule_results_duration_seconds](#scheduler_store_schedule_results_duration_seconds) | Histogram | Updated | `ALPHA` | Help text changed. |
| [scheduler_workload_preemption_attempts_total](#scheduler_workload_preemption_attempts_total) | Counter | Added | `ALPHA` |  |
| [scheduler_workload_preemption_victims](#scheduler_workload_preemption_victims) | Histogram | Added | `ALPHA` |  |
| [serviceaccount_legacy_tokens_total](#serviceaccount_legacy_tokens_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [serviceaccount_stale_tokens_total](#serviceaccount_stale_tokens_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [serviceaccount_valid_tokens_total](#serviceaccount_valid_tokens_total) | Counter | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [storage_operation_duration_seconds](#storage_operation_duration_seconds) | Histogram | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
| [volume_operation_total_seconds](#volume_operation_total_seconds) | Histogram | Updated | `BETA` | Stability level changed from `ALPHA` to `BETA`. |
## Detailed Changes

### apiserver_cache_list_fetched_objects_total
```diff
 - name: cache_list_fetched_objects_total
   namespace: apiserver
   help: Number of objects read from watch cache in the course of serving a LIST request
   type: Counter
+  deprecatedVersion: 1.37.0
   stabilityLevel: ALPHA
   labels:
     - group
     - index
     - resource
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```
### apiserver_cache_list_returned_objects_total
```diff
 - name: cache_list_returned_objects_total
   namespace: apiserver
   help: Number of objects returned for a LIST request from watch cache
   type: Counter
+  deprecatedVersion: 1.37.0
   stabilityLevel: ALPHA
   labels:
     - group
     - resource
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### apiserver_cache_list_total
```diff
 - name: cache_list_total
   namespace: apiserver
   help: Number of LIST requests served from watch cache
   type: Counter
+  deprecatedVersion: 1.37.0
   stabilityLevel: ALPHA
   labels:
     - group
     - index
     - resource
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### apiserver_flowcontrol_request_concurrency_in_use
```diff
-- name: request_concurrency_in_use
-  subsystem: flowcontrol
-  namespace: apiserver
-  help: Concurrency (number of seats) occupied by the currently executing (initial stage for a WATCH, any stage otherwise) requests in the API Priority and Fairness subsystem
-  type: Gauge
-  deprecatedVersion: 1.31.0
-  stabilityLevel: ALPHA
-  labels:
-    - flow_schema
-    - priority_level
-  componentEndpoints:
-    - component: kube-apiserver
-      endpoint: /metrics
```

### apiserver_flowcontrol_request_concurrency_limit
```diff
-- name: request_concurrency_limit
-  subsystem: flowcontrol
-  namespace: apiserver
-  help: Nominal number of execution seats configured for each priority level
-  type: Gauge
-  deprecatedVersion: 1.30.0
-  stabilityLevel: ALPHA
-  labels:
-    - priority_level
-  componentEndpoints:
-    - component: kube-apiserver
-      endpoint: /metrics
```

### apiserver_request_slo_duration_seconds
```diff
-- name: request_slo_duration_seconds
-  subsystem: apiserver
-  help: Response latency distribution (not counting webhook duration and priority & fairness queue wait times) in seconds for each verb, group, version, resource, subresource, scope and component.
-  type: Histogram
-  deprecatedVersion: 1.27.0
-  stabilityLevel: ALPHA
-  labels:
-    - component
-    - group
-    - resource
-    - scope
-    - subresource
-    - verb
-    - version
-  buckets:
-    - 0.05
-    - 0.1
-    - 0.2
-    - 0.4
-    - 0.6
-    - 0.8
-    - 1
-    - 1.25
-    - 1.5
-    - 2
-    - 3
-    - 4
-    - 5
-    - 6
-    - 8
-    - 10
-    - 15
-    - 20
-    - 30
-    - 45
-    - 60
-  componentEndpoints:
-    - component: kube-apiserver
-      endpoint: /metrics
```

### apiserver_storage_db_total_size_in_bytes
```diff
-- name: storage_db_total_size_in_bytes
-  subsystem: apiserver
-  help: Total size of the storage database file physically allocated in bytes.
-  type: Gauge
-  deprecatedVersion: 1.28.0
-  stabilityLevel: ALPHA
-  labels:
-    - endpoint
-  componentEndpoints:
-    - component: kube-apiserver
-      endpoint: /metrics
```

### apiserver_storage_list_duration_seconds
```diff
+- name: storage_list_duration_seconds
+  namespace: apiserver
+  help: Latency of the storage layer GetList call in seconds, including object decode, split by whether etcd RangeStream was used.
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - group
+    - resource
+    - streamed
+  buckets:
+    - 0.005
+    - 0.025
+    - 0.05
+    - 0.1
+    - 0.2
+    - 0.4
+    - 0.6
+    - 0.8
+    - 1
+    - 1.25
+    - 1.5
+    - 2
+    - 3
+    - 4
+    - 5
+    - 6
+    - 8
+    - 10
+    - 15
+    - 20
+    - 30
+    - 45
+    - 60
+  componentEndpoints:
+    - component: kube-apiserver
+      endpoint: /metrics
```

### apiserver_storage_list_evaluated_objects_total
```diff
 - name: apiserver_storage_list_evaluated_objects_total
   help: Number of objects tested in the course of serving a LIST request from storage
   type: Counter
   stabilityLevel: ALPHA
   labels:
     - group
     - resource
+    - storage
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### apiserver_storage_list_fetched_objects_total
```diff
 - name: apiserver_storage_list_fetched_objects_total
   help: Number of objects read from storage in the course of serving a LIST request
   type: Counter
   stabilityLevel: ALPHA
   labels:
     - group
+    - index
     - resource
+    - storage
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### apiserver_storage_list_returned_objects_total
```diff
 - name: apiserver_storage_list_returned_objects_total
   help: Number of objects returned for a LIST request from storage
   type: Counter
   stabilityLevel: ALPHA
   labels:
     - group
     - resource
+    - storage
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### apiserver_storage_list_total
```diff
 - name: apiserver_storage_list_total
   help: Number of LIST requests served from storage
   type: Counter
   stabilityLevel: ALPHA
   labels:
     - group
+    - index
     - resource
+    - storage
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### apiserver_watch_cache_initialization_duration_seconds
```diff
+- name: initialization_duration_seconds
+  subsystem: watch_cache
+  namespace: apiserver
+  help: Histogram of watch cache initialization duration in seconds, broken by resource type.
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - group
+    - resource
+  buckets:
+    - 0.05
+    - 0.1
+    - 0.25
+    - 0.5
+    - 1
+    - 2.5
+    - 5
+    - 10
+    - 30
+    - 60
+    - 180
+    - 600
+  componentEndpoints:
+    - component: kube-apiserver
+      endpoint: /metrics
```

### apiserver_watch_cache_initialization_errors_total
```diff
+- name: initialization_errors_total
+  subsystem: watch_cache
+  namespace: apiserver
+  help: Counter of watch cache initialization errors broken by resource type.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - group
+    - resource
+  componentEndpoints:
+    - component: kube-apiserver
+      endpoint: /metrics
```

### apiserver_watch_events_dispatch_duration_seconds
```diff
+- name: dispatch_duration_seconds
+  subsystem: watch_events
+  namespace: apiserver
+  help: Histogram of watch event dispatch latency broken by resource type and pipeline stage. The 'total' stage is the end-to-end latency of a delivered event.
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - group
+    - resource
+    - stage
+  buckets:
+    - 0.0001
+    - 0.0005
+    - 0.001
+    - 0.005
+    - 0.01
+    - 0.025
+    - 0.05
+    - 0.1
+    - 0.25
+    - 0.5
+    - 1
+    - 2.5
+    - 5
+  componentEndpoints:
+    - component: kube-apiserver
+      endpoint: /metrics
```

### apiserver_watch_events_sizes
```diff
 - name: watch_events_sizes
   subsystem: apiserver
   help: Watch event size distribution in bytes
   type: Histogram
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - group
     - resource
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
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### apiserver_watch_events_total
```diff
 - name: watch_events_total
   subsystem: apiserver
   help: Number of events sent in watch clients
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - group
     - resource
     - version
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### apiserver_watch_list_duration_seconds
```diff
 - name: watch_list_duration_seconds
   subsystem: apiserver
   help: Response latency distribution in seconds for watch list requests broken by group, version, resource and scope.
   type: Histogram
   stabilityLevel: BETA
   labels:
     - group
     - resource
     - scope
     - version
   buckets:
     - 0.05
     - 0.1
     - 0.2
     - 0.4
     - 0.6
     - 0.8
     - 1
+    - 1.25
+    - 1.5
     - 2
+    - 3
     - 4
+    - 5
     - 6
     - 8
     - 10
     - 15
     - 20
     - 30
     - 45
     - 60
+    - 90
+    - 120
+    - 180
+    - 300
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### apiserver_webhooks_x509_insecure_sha1_total
```diff
 - name: x509_insecure_sha1_total
   subsystem: webhooks
   namespace: apiserver
   help: Counts the number of requests to servers with insecure SHA1 signatures in their serving certificate OR the number of connection failures due to the insecure SHA1 signatures (either/or, based on the runtime environment)
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### apiserver_webhooks_x509_missing_san_total
```diff
 - name: x509_missing_san_total
   subsystem: webhooks
   namespace: apiserver
   help: Counts the number of requests to servers missing SAN extension in their serving certificate OR the number of connection failures due to the lack of x509 certificate SAN extension missing (either/or, based on the runtime environment)
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### cloud_node_lifecycle_controller_cloud_provider_calls_total
```diff
+- name: cloud_provider_calls_total
+  subsystem: cloud_node_lifecycle_controller
+  help: The total number of cloud provider API calls, labeled by operation and result.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - operation
+    - result
+  componentEndpoints:
+    - component: cloud-controller-manager
+      endpoint: /metrics
```

### cloud_node_lifecycle_controller_monitor_nodes_duration_seconds
```diff
+- name: monitor_nodes_duration_seconds
+  subsystem: cloud_node_lifecycle_controller
+  help: A metric measuring the duration in seconds of a single MonitorNodes loop execution.
+  type: Histogram
+  stabilityLevel: ALPHA
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
+  componentEndpoints:
+    - component: cloud-controller-manager
+      endpoint: /metrics
```

### csi_node_storage_health_status
```diff
+- name: csi_node_storage_health_status
+  help: CSI node storage backend health conditions currently reported (1 = present).
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - driver_name
+    - reason
+    - status
+  componentEndpoints:
+    - component: kubelet
+      endpoint: /metrics
```

### dra_node_prepare_skips_total
```diff
+- name: node_prepare_skips_total
+  subsystem: dra
+  help: Cumulative number of times NodePrepareResources gRPC calls were skipped due to SkipNodeOperations.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - driver_name
+  componentEndpoints:
+    - component: kubelet
+      endpoint: /metrics
```

### dra_node_unprepare_skips_total
```diff
+- name: node_unprepare_skips_total
+  subsystem: dra
+  help: Cumulative number of times NodeUnprepareResources gRPC calls were skipped due to SkipNodeOperations.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - driver_name
+  componentEndpoints:
+    - component: kubelet
+      endpoint: /metrics
```

### dynamic_resource_allocation_resource_claims
```diff
+- name: dynamic_resource_allocation_resource_claims
+  help: Number of ResourceClaims, categorized by allocation status, admin access, and source. Source can be 'resource_claim_template' (created from a template), 'extended_resource' (extended resources), or empty (manually created by a user).
+  type: Custom
+  stabilityLevel: ALPHA
+  labels:
+    - allocated
+    - admin_access
+    - source
+  componentEndpoints:
+    - component: kube-controller-manager
+      endpoint: /metrics
```

### dynamic_resource_allocation_resourceclaim_creates_total
```diff
+- name: resourceclaim_creates_total
+  subsystem: dynamic_resource_allocation
+  help: Number of ResourceClaims creation requests, categorized by creation status, admin access, and owning API group and kind (either Pod or scheduling.k8s.io/PodGroup)
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - admin_access
+    - owner_api_group
+    - owner_api_kind
+    - status
+  componentEndpoints:
+    - component: kube-controller-manager
+      endpoint: /metrics
+    - component: kube-scheduler
+      endpoint: /metrics
```

### etcd_version_info
```diff
-- name: version_info
-  namespace: etcd
-  help: Etcd server's binary version
-  type: Gauge
-  stabilityLevel: ALPHA
-  labels:
-    - binary_version
-  componentEndpoints:
-    - component: etcd-version-monitor
-      endpoint: /metrics
```

### horizontal_pod_autoscaler_controller_stale_sync_skips_total
```diff
+- name: stale_sync_skips_total
+  subsystem: horizontal_pod_autoscaler_controller
+  help: Total number of HPA syncs skipped due to a stale watch cache.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - group
+    - resource
+  componentEndpoints:
+    - component: kube-controller-manager
+      endpoint: /metrics
```

### kube_apiserver_pod_logs_pods_logs_backend_tls_failure_total
```diff
-- name: pods_logs_backend_tls_failure_total
-  subsystem: pod_logs
-  namespace: kube_apiserver
-  help: Total number of requests for pods/logs that failed due to kubelet server TLS verification
-  type: Counter
-  deprecatedVersion: 1.27.0
-  stabilityLevel: ALPHA
-  componentEndpoints:
-    - component: kube-apiserver
-      endpoint: /metrics
```

### kube_apiserver_pod_logs_pods_logs_insecure_backend_total
```diff
-- name: pods_logs_insecure_backend_total
-  subsystem: pod_logs
-  namespace: kube_apiserver
-  help: 'Total number of requests for pods/logs sliced by usage type: enforce_tls, skip_tls_allowed, skip_tls_denied'
-  type: Counter
-  deprecatedVersion: 1.27.0
-  stabilityLevel: ALPHA
-  labels:
-    - usage
-  componentEndpoints:
-    - component: kube-apiserver
-      endpoint: /metrics
```

### kubelet_pod_deferred_resize_duration_seconds
```diff
+- name: pod_deferred_resize_duration_seconds
+  subsystem: kubelet
+  help: 'Duration in seconds that a pod resize remains deferred before completion. Label ''priority_bucket'' classifies the pod priority (system-critical: >=2000000000, high: 100000..1999999999, medium: 1..99999, normal: 0/default, low: -999..-1, very-low: <=-1000, unknown: nil pod). Label ''resolution'' describes how the deferred resize was resolved (accepted, reverted, terminated).'
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - priority_bucket
+    - resolution
+  buckets:
+    - 5
+    - 10
+    - 15
+    - 20
+    - 30
+    - 45
+    - 60
+    - 90
+    - 120
+    - 150
+    - 180
+    - 300
+    - 600
+    - 1200
+    - 1800
+    - 3600
+  componentEndpoints:
+    - component: kubelet
+      endpoint: /metrics
```

### kubelet_pod_level_resources_admission_total
```diff
+- name: pod_level_resources_admission_total
+  subsystem: kubelet
+  help: Total number of pods admitted during Kubelet admission, categorized by resource configuration mode (pod_and_container_level, pod_level, container_level, or empty for unconfigured/BestEffort) and QoS class.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - config_mode
+    - qos_class
+  componentEndpoints:
+    - component: kubelet
+      endpoint: /metrics
```

### kubelet_pod_pending_resizes
```diff
 - name: pod_pending_resizes
   subsystem: kubelet
-  help: Number of pending resizes for pods.
+  help: 'Number of pending resizes for pods. Label ''priority_bucket'' classifies the pod priority (system-critical: >=2000000000, high: 100000..1999999999, medium: 1..99999, normal: 0/default, low: -999..-1, very-low: <=-1000, unknown: nil pod). Label ''reason'' describes the state (deferred, infeasible).'
   type: Gauge
   stabilityLevel: ALPHA
   labels:
+    - priority_bucket
     - reason
   componentEndpoints:
     - component: kubelet
       endpoint: /metrics
```

### kubelet_pod_requests_get_total
```diff
+- name: pod_requests_get_total
+  subsystem: kubelet
+  help: Number of requests to the PodsAPI Get endpoint. Broken down by server api version and status code.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - server_api_version
+    - status_code
+  componentEndpoints:
+    - component: kubelet
+      endpoint: /metrics
```

### kubelet_pod_requests_list_total
```diff
+- name: pod_requests_list_total
+  subsystem: kubelet
+  help: Number of requests to the PodsAPI List endpoint. Broken down by server api version and status code.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - server_api_version
+    - status_code
+  componentEndpoints:
+    - component: kubelet
+      endpoint: /metrics
```

### kubelet_pod_requests_total
```diff
+- name: pod_requests_total
+  subsystem: kubelet
+  help: Cumulative number of requests to the PodsAPI endpoint. Broken down by server api version and status code.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - server_api_version
+    - status_code
+  componentEndpoints:
+    - component: kubelet
+      endpoint: /metrics
```

### kubelet_pod_requests_watch_total
```diff
+- name: pod_requests_watch_total
+  subsystem: kubelet
+  help: Number of requests to the PodsAPI Watch endpoint. Broken down by server api version and status code.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - server_api_version
+    - status_code
+  componentEndpoints:
+    - component: kubelet
+      endpoint: /metrics
```

### kubelet_resource_manager_allocation_errors_total
```diff
+- name: resource_manager_allocation_errors_total
+  subsystem: kubelet
+  help: Number of errors encountered during exclusive resource allocation.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - resource_name
+    - source
+  componentEndpoints:
+    - component: kubelet
+      endpoint: /metrics
```

### kubelet_resource_manager_allocations_total
```diff
+- name: resource_manager_allocations_total
+  subsystem: kubelet
+  help: Number of exclusive resource allocations performed by a resource manager.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - resource_name
+    - source
+  componentEndpoints:
+    - component: kubelet
+      endpoint: /metrics
```

### kubelet_resource_manager_container_assignments_total
```diff
+- name: resource_manager_container_assignments_total
+  subsystem: kubelet
+  help: Number of containers with a specific type of resource assignment.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - assignment_type
+    - resource_name
+  componentEndpoints:
+    - component: kubelet
+      endpoint: /metrics
```

### kubelet_volume_stats_health_status_abnormal
```diff
-- name: kubelet_volume_stats_health_status_abnormal
-  help: Abnormal volume health status. The count is either 1 or 0. 1 indicates the volume is unhealthy, 0 indicates volume is healthy
-  type: Custom
-  stabilityLevel: ALPHA
-  labels:
-    - namespace
-    - persistentvolumeclaim
-  componentEndpoints:
-    - component: kubelet
-      endpoint: /metrics
```

### kubeproxy_nftables_localhost_nodeport_listener_creation_failures_total
```diff
+- name: nftables_localhost_nodeport_listener_creation_failures_total
+  subsystem: kubeproxy
+  help: Number of localhost NodePort userspace proxy listener creation failures.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - ip_family
+  componentEndpoints:
+    - component: kube-proxy
+      endpoint: /metrics
```

### kubeproxy_nftables_localhost_nodeport_listeners
```diff
+- name: nftables_localhost_nodeport_listeners
+  subsystem: kubeproxy
+  help: Number of active localhost NodePort userspace proxy listeners.
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - ip_family
+  componentEndpoints:
+    - component: kube-proxy
+      endpoint: /metrics
```

### kubeproxy_nftables_localhost_nodeport_rejected_packets_total
```diff
+- name: kubeproxy_nftables_localhost_nodeport_rejected_packets_total
+  help: Number of packets rejected on localhost NodePorts. UDP and SCTP are always rejected; TCP is rejected when the localhost NodePort proxy is not running
+  type: Custom
+  stabilityLevel: ALPHA
+  labels:
+    - ip_family
+    - protocol
+  componentEndpoints:
+    - component: kube-proxy
+      endpoint: /metrics
```

### kubeproxy_sync_proxy_rules_winkernel_lb_create_failures_total
```diff
+- name: sync_proxy_rules_winkernel_lb_create_failures_total
+  subsystem: kubeproxy
+  help: Cumulative proxy winkernel lb create failures
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - error
+    - ip_family
+    - lb_type
+  componentEndpoints:
+    - component: kube-proxy
+      endpoint: /metrics
```

### kubeproxy_sync_proxy_rules_winkernel_lb_delete_failures_total
```diff
+- name: sync_proxy_rules_winkernel_lb_delete_failures_total
+  subsystem: kubeproxy
+  help: Cumulative proxy winkernel lb delete failures
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - error
+    - ip_family
+    - lb_type
+  componentEndpoints:
+    - component: kube-proxy
+      endpoint: /metrics
```

### kubeproxy_sync_proxy_rules_winkernel_lb_update_failures_total
```diff
+- name: sync_proxy_rules_winkernel_lb_update_failures_total
+  subsystem: kubeproxy
+  help: Cumulative proxy winkernel lb update failures
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - error
+    - ip_family
+    - lb_type
+  componentEndpoints:
+    - component: kube-proxy
+      endpoint: /metrics
```

### latency
```diff
-- name: latency
-  type: Summary
-  stabilityLevel: ALPHA
-  labels:
-    - node
-  objectives:
-    0.5: 0.05
-    0.75: 0.025
-    0.9: 0.01
-    0.99: 0.001
```

### replicaset_controller_sorting_deletion_age_ratio
```diff
 - name: sorting_deletion_age_ratio
   subsystem: replicaset_controller
-  help: The ratio of chosen deleted pod's ages to the current youngest pod's age (at the time). Should be <2. The intent of this metric is to measure the rough efficacy of the LogarithmicScaleDown feature gate's effect on the sorting (and deletion) of pods when a replicaset scales down. This only considers Ready pods when calculating and reporting.
+  help: The ratio of chosen deleted pod's ages to the current youngest pod's age (at the time). Should be <2. The intent of this metric is to measure the rough efficacy of the sorting (and deletion) of pods when a replicaset scales down. This only considers Ready pods when calculating and reporting.
   type: Histogram
   stabilityLevel: ALPHA
   buckets:
     - 0.25
     - 0.5
     - 1
     - 2
     - 4
     - 8
   componentEndpoints:
     - component: kube-controller-manager
       endpoint: /metrics
```

### resource_manager_allocation_errors_total
```diff
-- name: resource_manager_allocation_errors_total
-  help: Number of errors encountered during exclusive resource allocation.
-  type: Counter
-  stabilityLevel: ALPHA
-  labels:
-    - resource_name
-    - source
-  componentEndpoints:
-    - component: kubelet
-      endpoint: /metrics
```

### resource_manager_allocations_total
```diff
-- name: resource_manager_allocations_total
-  help: Number of exclusive resource allocations performed by a resource manager.
-  type: Counter
-  stabilityLevel: ALPHA
-  labels:
-    - resource_name
-    - source
-  componentEndpoints:
-    - component: kubelet
-      endpoint: /metrics
```

### resource_manager_container_assignments
```diff
-- name: resource_manager_container_assignments
-  help: Number of containers with a specific type of resource assignment.
-  type: Counter
-  stabilityLevel: ALPHA
-  labels:
-    - assignment_type
-    - resource_name
-  componentEndpoints:
-    - component: kubelet
-      endpoint: /metrics
```

### resourceclaim_controller_creates_total
```diff
-- name: creates_total
-  subsystem: resourceclaim_controller
-  help: Number of ResourceClaims creation requests, categorized by creation status and admin access
-  type: Counter
-  stabilityLevel: ALPHA
-  labels:
-    - admin_access
-    - status
-  componentEndpoints:
-    - component: kube-controller-manager
-      endpoint: /metrics
```

### resourceclaim_controller_resource_claims
```diff
-- name: resourceclaim_controller_resource_claims
-  help: Number of ResourceClaims, categorized by allocation status, admin access, and source. Source can be 'resource_claim_template' (created from a template), 'extended_resource' (extended resources), or empty (manually created by a user).
-  type: Custom
-  stabilityLevel: ALPHA
-  labels:
-    - allocated
-    - admin_access
-    - source
-  componentEndpoints:
-    - component: kube-controller-manager
-      endpoint: /metrics
```

### route_controller_route_sync_total
```diff
 - name: route_sync_total
   subsystem: route_controller
-  help: A metric counting the amount of times routes have been synced with the cloud provider.
+  help: A metric counting the number of route reconciles with the cloud provider, labeled by the trigger (periodic or node_change) and the outcome (error if the reconcile failed, changed if at least one route was created or deleted, otherwise noop).
   type: Counter
   stabilityLevel: ALPHA
+  labels:
+    - outcome
+    - trigger
   componentEndpoints:
     - component: cloud-controller-manager
       endpoint: /metrics
```

### scheduler_batch_rescore_attempts_total
```diff
+- name: batch_rescore_attempts_total
+  subsystem: scheduler
+  help: Counts of rescore attempts during opportunistic batching.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - profile
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_batch_rescore_duration_seconds
```diff
+- name: batch_rescore_duration_seconds
+  subsystem: scheduler
+  help: Latency for rescoring a node during opportunistic batching.
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
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_generated_placements_total
```diff
+- name: generated_placements_total
+  subsystem: scheduler
+  help: Number of candidate placements generated when scheduling pod groups.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - profile
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_pending_pods
```diff
 - name: pending_pods
   subsystem: scheduler
-  help: Number of pending pods, by the queue type. 'active' means number of pods in activeQ; 'backoff' means number of pods in backoffQ; 'unschedulable' means number of pods in unschedulablePods that the scheduler attempted to schedule and failed; 'gated' is the number of unschedulable pods that the scheduler never attempted to schedule because they are gated.
+  help: Number of pending pods, by the queue type. 'active' means number of pods in activeQ; 'backoff' means number of pods in backoffQ; 'unschedulable' means number of pods in unschedulableEntities that the scheduler attempted to schedule and failed; 'gated' is the number of unschedulable pods that the scheduler never attempted to schedule because they are gated; 'incomplete' means number of pods in incompletePodGroupPods; 'pending' means number of pods in pendingPodGroupPods.
   type: Gauge
   stabilityLevel: STABLE
   labels:
     - queue
   componentEndpoints:
     - component: kube-scheduler
       endpoint: /metrics
```

### scheduler_placement_evaluation_duration_seconds
```diff
+- name: placement_evaluation_duration_seconds
+  subsystem: scheduler
+  help: Latency in seconds of evaluating a single candidate placement when scheduling pod groups, by result. 'feasible' means the pod group fit into the placement, while 'infeasible' means it did not.
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - profile
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
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_placement_evaluations_total
```diff
+- name: placement_evaluations_total
+  subsystem: scheduler
+  help: Number of candidate placements evaluated when scheduling pod groups, by result. 'feasible' means the pod group fit into the placement, while 'infeasible' means it did not.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - profile
+    - result
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_plugin_execution_duration_seconds
```diff
 - name: plugin_execution_duration_seconds
   subsystem: scheduler
   help: Duration for running a plugin at a specific extension point.
   type: Histogram
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - extension_point
     - plugin
     - status
   buckets:
     - 1e-05
     - 1.5000000000000002e-05
     - 2.2500000000000005e-05
     - 3.375000000000001e-05
     - 5.062500000000001e-05
     - 7.593750000000002e-05
     - 0.00011390625000000003
     - 0.00017085937500000006
     - 0.0002562890625000001
     - 0.00038443359375000017
     - 0.0005766503906250003
     - 0.0008649755859375004
     - 0.0012974633789062506
     - 0.0019461950683593758
     - 0.0029192926025390638
     - 0.004378938903808595
     - 0.006568408355712893
     - 0.009852612533569338
     - 0.014778918800354007
     - 0.02216837820053101
   componentEndpoints:
     - component: kube-scheduler
       endpoint: /metrics
```

### scheduler_pod_scheduled_after_flush_total
```diff
 - name: pod_scheduled_after_flush_total
   subsystem: scheduler
-  help: Number of pods that were successfully scheduled after being flushed from unschedulablePods due to timeout. This metric helps detect potential queueing hint misconfigurations or event handling issues.
+  help: Number of pods that were successfully scheduled after being flushed from unschedulableEntities due to timeout. This metric helps detect potential queueing hint misconfigurations or event handling issues.
   type: Counter
   stabilityLevel: ALPHA
   componentEndpoints:
     - component: kube-scheduler
       endpoint: /metrics
```

### scheduler_podgroup_scheduling_attempt_duration_seconds
```diff
 - name: podgroup_scheduling_attempt_duration_seconds
   subsystem: scheduler
-  help: Pod group scheduling attempt latency in seconds (scheduling algorithm + binding)
+  help: Pod group scheduling attempt latency in seconds
   type: Histogram
   stabilityLevel: ALPHA
   labels:
     - profile
     - result
   buckets:
     - 0.001
     - 0.002
     - 0.004
     - 0.008
     - 0.016
     - 0.032
     - 0.064
     - 0.128
     - 0.256
     - 0.512
     - 1.024
     - 2.048
     - 4.096
     - 8.192
     - 16.384
   componentEndpoints:
     - component: kube-scheduler
       endpoint: /metrics
```

### scheduler_pre_queueing_hint_evaluations_total
```diff
+- name: pre_queueing_hint_evaluations_total
+  subsystem: scheduler
+  help: Number of PreQueueingHint evaluations, labeled by plugin and result (all_pods or narrowed).
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - plugin
+    - result
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_preemption_evaluation_duration_seconds
```diff
+- name: preemption_evaluation_duration_seconds
+  subsystem: scheduler
+  help: Duration in seconds for identifying the target preemption victims.
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - preemptor
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
+    - 32.768
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_preemption_execution_duration_seconds
```diff
+- name: preemption_execution_duration_seconds
+  subsystem: scheduler
+  help: Duration in seconds for preempting the target preemption victims. With async preemption enabled, preemption execution does not block the scheduling of other pods.
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - preemptor
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
+    - 32.768
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_preemption_goroutines_duration_seconds
```diff
 - name: preemption_goroutines_duration_seconds
   subsystem: scheduler
   help: Duration in seconds for running goroutines for the preemption.
   type: Histogram
+  deprecatedVersion: 1.37.0
   stabilityLevel: ALPHA
   labels:
     - result
   buckets:
     - 0.01
     - 0.02
     - 0.04
     - 0.08
     - 0.16
     - 0.32
     - 0.64
     - 1.28
     - 2.56
     - 5.12
     - 10.24
     - 20.48
     - 40.96
     - 81.92
     - 163.84
     - 327.68
     - 655.36
     - 1310.72
     - 2621.44
     - 5242.88
   componentEndpoints:
     - component: kube-scheduler
       endpoint: /metrics
```

### scheduler_preemption_pdb_violations_total
```diff
+- name: preemption_pdb_violations_total
+  subsystem: scheduler
+  help: Total number of pod disruption budget violations caused by preemption.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - preemptor
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_preemption_victims
```diff
 - name: preemption_victims
   subsystem: scheduler
-  help: Number of selected preemption victims
+  help: Number of selected preemption victims for preemption initiated by a single pod
   type: Histogram
   stabilityLevel: STABLE
   buckets:
     - 1
     - 2
     - 4
     - 8
     - 16
     - 32
     - 64
   componentEndpoints:
     - component: kube-scheduler
       endpoint: /metrics
```

### scheduler_preemption_workload_disruptions
```diff
+- name: preemption_workload_disruptions
+  subsystem: scheduler
+  help: Number of workload preemption units being preempted. A single preemption unit can be all pods in a pod group (in case of DisruptionMode=all), or a single pod (in case of DisruptionMode=single).
+  type: Histogram
+  stabilityLevel: ALPHA
+  labels:
+    - preemptor
+  buckets:
+    - 1
+    - 2
+    - 4
+    - 8
+    - 16
+    - 32
+    - 64
+    - 128
+    - 256
+    - 512
+    - 1024
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_queue_incoming_entities_total
```diff
+- name: queue_incoming_entities_total
+  subsystem: scheduler
+  help: Number of scheduling entities added to scheduling queues by event, queue type, and entity type. Entity types are either 'pod' (for individual pods that are not members of any podgroup) or 'podgroup'.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - event
+    - queue
+    - type
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_queued_entities
```diff
+- name: queued_entities
+  subsystem: scheduler
+  help: Number of queued scheduling entities ('pod' or 'podgroup'; 'pod' stands for individual pods that are not members of any podgroup) by the queue type. 'active' means number of entities in activeQ; 'backoff' means number of entities in backoffQ; 'unschedulable' means number of entities in unschedulableEntities that the scheduler attempted to schedule and failed; 'gated' is the number of unschedulable entities that the scheduler never attempted to schedule because they are gated.
+  type: Gauge
+  stabilityLevel: ALPHA
+  labels:
+    - queue
+    - type
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_resourceclaim_creates_total
```diff
-- name: resourceclaim_creates_total
-  subsystem: scheduler
-  help: Number of ResourceClaims creation requests within scheduler
-  type: Counter
-  stabilityLevel: ALPHA
-  labels:
-    - status
-  componentEndpoints:
-    - component: kube-scheduler
-      endpoint: /metrics
```

### scheduler_scheduling_algorithm_duration_seconds
```diff
 - name: scheduling_algorithm_duration_seconds
   subsystem: scheduler
   help: Scheduling algorithm latency in seconds
   type: Histogram
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   buckets:
     - 0.001
     - 0.002
     - 0.004
     - 0.008
     - 0.016
     - 0.032
     - 0.064
     - 0.128
     - 0.256
     - 0.512
     - 1.024
     - 2.048
     - 4.096
     - 8.192
     - 16.384
   componentEndpoints:
     - component: kube-scheduler
       endpoint: /metrics
```

### scheduler_store_schedule_results_duration_seconds
```diff
 - name: store_schedule_results_duration_seconds
   subsystem: scheduler
-  help: Latency for getting a no.
+  help: Latency for storing scheduling results.
   type: Histogram
   stabilityLevel: ALPHA
   labels:
     - profile
   buckets:
     - 1e-05
     - 2e-05
     - 4e-05
     - 8e-05
     - 0.00016
     - 0.00032
     - 0.00064
     - 0.00128
     - 0.00256
     - 0.00512
     - 0.01024
     - 0.02048
   componentEndpoints:
     - component: kube-scheduler
       endpoint: /metrics
```

### scheduler_workload_preemption_attempts_total
```diff
+- name: workload_preemption_attempts_total
+  subsystem: scheduler
+  help: Total preemption attempts initiated by workload (including pod groups) in the cluster till now.
+  type: Counter
+  stabilityLevel: ALPHA
+  labels:
+    - result
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### scheduler_workload_preemption_victims
```diff
+- name: workload_preemption_victims
+  subsystem: scheduler
+  help: Number of pod preemption victims caused by workload preemption.
+  type: Histogram
+  stabilityLevel: ALPHA
+  buckets:
+    - 1
+    - 2
+    - 4
+    - 8
+    - 16
+    - 32
+    - 64
+    - 128
+    - 256
+    - 512
+    - 1024
+  componentEndpoints:
+    - component: kube-scheduler
+      endpoint: /metrics
```

### serviceaccount_legacy_tokens_total
```diff
 - name: legacy_tokens_total
   subsystem: serviceaccount
   help: Cumulative legacy service account tokens used
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### serviceaccount_stale_tokens_total
```diff
 - name: stale_tokens_total
   subsystem: serviceaccount
   help: Cumulative stale projected service account tokens used
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### serviceaccount_valid_tokens_total
```diff
 - name: valid_tokens_total
   subsystem: serviceaccount
   help: Cumulative valid projected service account tokens used
   type: Counter
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   componentEndpoints:
     - component: kube-apiserver
       endpoint: /metrics
```

### storage_operation_duration_seconds
```diff
 - name: storage_operation_duration_seconds
   help: Storage operation duration
   type: Histogram
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - migrated
     - operation_name
     - status
     - volume_plugin
   buckets:
     - 0.1
     - 0.25
     - 0.5
     - 1
     - 2.5
     - 5
     - 10
     - 15
     - 25
     - 50
     - 120
     - 300
     - 600
   componentEndpoints:
     - component: kubelet
       endpoint: /metrics
```

### volume_operation_total_seconds
```diff
 - name: volume_operation_total_seconds
   help: Storage operation end to end duration in seconds
   type: Histogram
-  stabilityLevel: ALPHA
+  stabilityLevel: BETA
   labels:
     - operation_name
     - plugin_name
   buckets:
     - 0.1
     - 0.25
     - 0.5
     - 1
     - 2.5
     - 5
     - 10
     - 15
     - 25
     - 50
     - 120
     - 300
     - 600
   componentEndpoints:
     - component: kubelet
       endpoint: /metrics
```
