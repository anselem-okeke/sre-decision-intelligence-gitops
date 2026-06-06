# Phase 8 — SLI/SLO Discovery Validation

## Objective

Validate available metrics for Bank of Anthos SLI/SLO design.

## Workload namespace

```text
fintech-workload
```

## Discovery query: all workload metrics

```promql
count by (__name__) (
  {namespace="fintech-workload"}
)
```

## Result

```shell
namespace_memory:kube_pod_container_resource_requests:sum{}	1
namespace_cpu:kube_pod_container_resource_requests:sum{}	1
namespace_memory:kube_pod_container_resource_limits:sum{}	1
namespace_cpu:kube_pod_container_resource_limits:sum{}	1
node_namespace_pod:kube_pod_info:{}	9
namespace_workload_pod:kube_pod_owner:relabel{}	9
kube_configmap_info{}	6
kube_configmap_created{}	6
kube_configmap_metadata_resource_version{}	6
kube_deployment_owner{}	7
kube_deployment_created{}	7
kube_deployment_status_replicas{}	7
kube_deployment_status_replicas_ready{}	7
kube_deployment_status_replicas_available{}	7
kube_deployment_status_replicas_unavailable{}	7
kube_deployment_status_replicas_updated{}	7
kube_deployment_status_terminating_replicas{}	7
kube_deployment_status_observed_generation{}	7
kube_deployment_status_condition{}	42
kube_deployment_spec_replicas{}	7
kube_deployment_spec_paused{}	7
kube_deployment_spec_strategy_rollingupdate_max_unavailable{}	7
kube_deployment_spec_strategy_rollingupdate_max_surge{}	7
kube_deployment_metadata_generation{}	7
kube_namespace_created{}	1
kube_namespace_status_phase{}	2
kube_replicaset_created{}	7
kube_replicaset_status_replicas{}	7
kube_replicaset_status_fully_labeled_replicas{}	7
kube_replicaset_status_ready_replicas{}	7
kube_replicaset_status_terminating_replicas{}	7
kube_replicaset_status_observed_generation{}	7
kube_replicaset_spec_replicas{}	7
kube_replicaset_metadata_generation{}	7
kube_replicaset_owner{}	7
kube_secret_info{}	1
kube_secret_type{}	1
kube_secret_created{}	1
kube_secret_metadata_resource_version{}	1
kube_secret_owner{}	1
kube_statefulset_created{}	2
kube_statefulset_status_replicas{}	2
kube_statefulset_status_replicas_available{}	2
kube_statefulset_status_replicas_current{}	2
kube_statefulset_status_replicas_ready{}	2
kube_statefulset_status_replicas_updated{}	2
kube_statefulset_status_observed_generation{}	2
kube_statefulset_replicas{}	2
kube_statefulset_metadata_generation{}	2
kube_statefulset_persistentvolumeclaim_retention_policy{}	2
node_namespace_pod_container:container_cpu_usage_seconds_total:sum_rate5m{}	18
node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{}	18
container_blkio_device_usage_total{}	38
container_cpu_cfs_periods_total{}	18
container_cpu_cfs_throttled_periods_total{}	18
container_cpu_load_d_average_10s{}	27
container_cpu_usage_seconds_total{}	27
container_fs_reads_bytes_total{}	19
container_fs_reads_total{}	19
container_fs_writes_bytes_total{}	19
container_fs_writes_total{}	19
container_last_seen{}	27
container_memory_cache{}	27
container_memory_failcnt{}	27
container_memory_failures_total{}	54
container_memory_kernel_usage{}	27
container_memory_max_usage_bytes{}	27
container_memory_rss{}	27
container_memory_total_active_file_bytes{}	27
container_memory_total_inactive_file_bytes{}	27
container_memory_usage_bytes{}	27
container_memory_working_set_bytes{}	27
container_network_receive_bytes_total{}	27
container_network_receive_errors_total{}	27
container_network_receive_packets_dropped_total{}	27
container_network_receive_packets_total{}	27
container_network_transmit_bytes_total{}	27
container_network_transmit_errors_total{}	27
container_network_transmit_packets_dropped_total{}	27
container_network_transmit_packets_total{}	27
container_oom_events_total{}	27
container_pressure_cpu_stalled_seconds_total{}	27
container_pressure_cpu_waiting_seconds_total{}	27
container_pressure_io_stalled_seconds_total{}	27
container_pressure_io_waiting_seconds_total{}	27
container_pressure_memory_stalled_seconds_total{}	27
container_pressure_memory_waiting_seconds_total{}	27
container_processes{}	27
container_sockets{}	27
container_start_time_seconds{}	27
container_threads{}	27
container_ulimits_soft{}	18
kube_endpointslice_info{}	8
kube_endpointslice_created{}	8
kube_endpointslice_endpoints{}	8
kube_endpointslice_ports{}	8
kube_pod_container_info{}	9
kube_pod_container_resource_limits{}	23
kube_pod_container_resource_requests{}	23
kube_pod_container_state_started{}	9
kube_pod_container_status_ready{}	9
kube_pod_container_status_restarts_total{}	9
kube_pod_container_status_running{}	9
kube_pod_container_status_terminated{}	9
kube_pod_container_status_waiting{}	9
kube_pod_created{}	9
kube_pod_info{}	9
kube_pod_ips{}	9
kube_pod_owner{}	9
kube_pod_restart_policy{}	9
kube_pod_start_time{}	9
kube_pod_status_phase{}	45
kube_pod_status_qos_class{}	27
kube_pod_status_ready{}	27
kube_pod_status_ready_time{}	9
kube_pod_status_initialized_time{}	9
kube_pod_status_container_ready_time{}	9
kube_pod_status_reason{}	45
kube_pod_status_scheduled{}	27
kube_pod_status_scheduled_time{}	9
kube_pod_tolerations{}	18
kube_pod_service_account{}	9
kube_pod_scheduler{}	9
kube_service_info{}	8
kube_service_created{}	8
kube_service_spec_type{}	8
kube_service_status_load_balancer_ingress{}	1
cluster:namespace:pod_memory:active:kube_pod_container_resource_requests{}	9
cluster:namespace:pod_cpu:active:kube_pod_container_resource_requests{}	9
cluster:namespace:pod_memory:active:kube_pod_container_resource_limits{}	9
cluster:namespace:pod_cpu:active:kube_pod_container_resource_limits{}	9
prober_probe_duration_seconds_bucket{}	144
prober_probe_duration_seconds_sum{}	12
prober_probe_duration_seconds_count{}	12
prober_probe_total{}	19
kubelet_container_log_filesystem_used_bytes{}	9
node_namespace_pod_container:container_memory_working_set_bytes{}	18
ALERTS_FOR_STATE{}	2
node_namespace_pod_container:container_memory_cache{}	18
node_namespace_pod_container:container_memory_rss{}	18
ALERTS{}	2
kube_statefulset_status_current_revision{}	2
kube_statefulset_status_update_revision{}	2
kube_pod_container_status_last_terminated_reason{}	2
kube_pod_container_status_last_terminated_exitcode{}	2
kube_pod_container_status_last_terminated_timestamp{}
```

## Discovery query: HTTP/request metrics

```promql
count by (__name__) (
  {__name__=~".*http.*|.*request.*|.*response.*", namespace="fintech-workload"}
)
```

## Result

```shell
namespace_memory:kube_pod_container_resource_requests:sum{}	1
namespace_cpu:kube_pod_container_resource_requests:sum{}	1
kube_pod_container_resource_requests{}	23
cluster:namespace:pod_memory:active:kube_pod_container_resource_requests{}	9
cluster:namespace:pod_cpu:active:kube_pod_container_resource_requests{}
```

## Discovery query: latency metrics

```promql
count by (__name__) (
  {__name__=~".*duration.*|.*latency.*|.*seconds.*", namespace="fintech-workload"}
)
```

## Result

```shell
node_namespace_pod_container:container_cpu_usage_seconds_total:sum_rate5m{}	18
node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{}	18
container_cpu_usage_seconds_total{}	27
container_pressure_cpu_stalled_seconds_total{}	27
container_pressure_cpu_waiting_seconds_total{}	27
container_pressure_io_stalled_seconds_total{}	27
container_pressure_io_waiting_seconds_total{}	27
container_pressure_memory_stalled_seconds_total{}	27
container_pressure_memory_waiting_seconds_total{}	27
container_start_time_seconds{}	27
prober_probe_duration_seconds_bucket{}	144
prober_probe_duration_seconds_sum{}	12
prober_probe_duration_seconds_count{}
```

## Discovery query: ingress/gateway metrics

```promql
count by (__name__) (
  {__name__=~".*ingress.*|.*nginx.*|.*gateway.*|.*http.*"}
)
```

## Result

```shell
cilium_feature_adv_connect_and_lb_egress_gateway_enabled{}	5
go_godebug_non_default_behavior_http2client_events_total{}	9
go_godebug_non_default_behavior_http2server_events_total{}	9
go_godebug_non_default_behavior_httpcookiemaxnum_events_total{}	9
go_godebug_non_default_behavior_httplaxcontentlength_events_total{}	9
go_godebug_non_default_behavior_httpmuxgo121_events_total{}	9
go_godebug_non_default_behavior_httpservecontentkeepheaders_events_total{}	9
kubelet_http_inflight_requests{}	28
kubelet_http_requests_duration_seconds_bucket{}	336
kubelet_http_requests_duration_seconds_sum{}	28
kubelet_http_requests_duration_seconds_count{}	28
kubelet_http_requests_total{}	28
kubelet_lifecycle_handler_http_fallbacks_total{}	5
promhttp_metric_handler_errors_total{}	14
promhttp_metric_handler_requests_in_flight{}	10
promhttp_metric_handler_requests_total{}	30
envoy_cluster_http1_dropped_headers_with_underscores{}	6
envoy_cluster_http1_metadata_not_supported_error{}	6
envoy_cluster_http1_requests_rejected_with_underscores_in_headers{}	6
envoy_cluster_http1_response_flood{}	6
envoy_cluster_http2_dropped_headers_with_underscores{}	5
envoy_cluster_http2_goaway_sent{}	5
envoy_cluster_http2_header_overflow{}	5
envoy_cluster_http2_headers_cb_no_stream{}	5
envoy_cluster_http2_inbound_empty_frames_flood{}	5
envoy_cluster_http2_inbound_priority_frames_flood{}	5
envoy_cluster_http2_inbound_window_update_frames_flood{}	5
envoy_cluster_http2_keepalive_timeout{}	5
envoy_cluster_http2_metadata_empty_frames{}	5
envoy_cluster_http2_outbound_control_flood{}	5
envoy_cluster_http2_outbound_flood{}	5
envoy_cluster_http2_requests_rejected_with_underscores_in_headers{}	5
envoy_cluster_http2_rx_messaging_error{}	5
envoy_cluster_http2_rx_reset{}	5
envoy_cluster_http2_stream_refused_errors{}	5
envoy_cluster_http2_trailers{}	5
envoy_cluster_http2_tx_flush_timeout{}	5
envoy_cluster_http2_tx_reset{}	5
envoy_cluster_outlier_detection_ejections_detected_consecutive_gateway_failure{}	5
envoy_cluster_outlier_detection_ejections_enforced_consecutive_gateway_failure{}	5
envoy_cluster_upstream_cx_http1_total{}	35
envoy_cluster_upstream_cx_http2_total{}	35
envoy_cluster_upstream_cx_http3_total{}	35
envoy_cluster_upstream_http3_broken{}	35
envoy_http_downstream_cx_delayed_close_timeout{}	25
envoy_http_downstream_cx_destroy{}	25
envoy_http_downstream_cx_destroy_active_rq{}	25
envoy_http_downstream_cx_destroy_local{}	25
envoy_http_downstream_cx_destroy_local_active_rq{}	25
envoy_http_downstream_cx_destroy_remote{}	25
envoy_http_downstream_cx_destroy_remote_active_rq{}	25
envoy_http_downstream_cx_drain_close{}	25
envoy_http_downstream_cx_http1_total{}	25
envoy_http_downstream_cx_http2_total{}	25
envoy_http_downstream_cx_http3_total{}	25
envoy_http_downstream_cx_idle_timeout{}	25
envoy_http_downstream_cx_max_duration_reached{}	25
envoy_http_downstream_cx_max_requests_reached{}	25
envoy_http_downstream_cx_overload_disable_keepalive{}	25
envoy_http_downstream_cx_protocol_error{}	25
envoy_http_downstream_cx_rx_bytes_total{}	25
envoy_http_downstream_cx_ssl_total{}	25
envoy_http_downstream_cx_total{}	25
envoy_http_downstream_cx_tx_bytes_total{}	25
envoy_http_downstream_cx_upgrades_total{}	25
envoy_http_downstream_flow_control_paused_reading_total{}	25
envoy_http_downstream_flow_control_resumed_reading_total{}	25
envoy_http_downstream_rq_completed{}	25
envoy_http_downstream_rq_failed_path_normalization{}	25
envoy_http_downstream_rq_header_timeout{}	25
envoy_http_downstream_rq_http1_total{}	25
envoy_http_downstream_rq_http2_total{}	25
envoy_http_downstream_rq_http3_total{}	25
envoy_http_downstream_rq_idle_timeout{}	25
envoy_http_downstream_rq_max_duration_reached{}	25
envoy_http_downstream_rq_non_relative_path{}	25
envoy_http_downstream_rq_overload_close{}	25
envoy_http_downstream_rq_redirected_with_normalized_path{}	25
envoy_http_downstream_rq_rejected_via_ip_detection{}	25
envoy_http_downstream_rq_response_before_rq_complete{}	25
envoy_http_downstream_rq_rx_reset{}	25
envoy_http_downstream_rq_timeout{}	25
envoy_http_downstream_rq_too_large{}	25
envoy_http_downstream_rq_too_many_premature_resets{}	25
envoy_http_downstream_rq_total{}	25
envoy_http_downstream_rq_tx_reset{}	25
envoy_http_downstream_rq_ws_on_non_ws_route{}	25
envoy_http_downstream_rq_xx{}	125
envoy_http_no_cluster{}	25
envoy_http_no_route{}	25
envoy_http_passthrough_internal_redirect_bad_location{}	25
envoy_http_passthrough_internal_redirect_no_route{}	25
envoy_http_passthrough_internal_redirect_predicate{}	25
envoy_http_passthrough_internal_redirect_too_many_redirects{}	25
envoy_http_passthrough_internal_redirect_unsafe_scheme{}	25
envoy_http_rds_config_reload{}	10
envoy_http_rds_init_fetch_timeout{}	10
envoy_http_rds_rate_limit_enforced{}	10
envoy_http_rds_update_attempt{}	10
envoy_http_rds_update_empty{}	10
envoy_http_rds_update_failure{}	10
envoy_http_rds_update_rejected{}	10
envoy_http_rds_update_success{}	10
envoy_http_rq_direct_response{}	25
envoy_http_rq_overload_local_reply{}	25
envoy_http_rq_redirect{}	25
envoy_http_rq_reset_after_downstream_response_started{}	25
envoy_http_rq_total{}	25
envoy_http_rs_too_large{}	25
envoy_http_tracing_client_enabled{}	20
envoy_http_tracing_health_check{}	20
envoy_http_tracing_not_traceable{}	20
envoy_http_tracing_random_sampling{}	20
envoy_http_tracing_service_forced{}	20
envoy_http1_dropped_headers_with_underscores{}	5
envoy_http1_metadata_not_supported_error{}	5
envoy_http1_requests_rejected_with_underscores_in_headers{}	5
envoy_http1_response_flood{}	5
envoy_listener_admin_http_downstream_rq_completed{}	5
envoy_listener_admin_http_downstream_rq_xx{}	25
envoy_listener_http_downstream_rq_completed{}	20
envoy_listener_http_downstream_rq_xx{}	100
envoy_cluster_http2_deferred_stream_close{}	5
envoy_cluster_http2_outbound_control_frames_active{}	5
envoy_cluster_http2_outbound_frames_active{}	5
envoy_cluster_http2_pending_send_bytes{}	5
envoy_cluster_http2_streams_active{}	5
envoy_http_downstream_cx_active{}	25
envoy_http_downstream_cx_http1_active{}	25
envoy_http_downstream_cx_http1_soft_drain{}	25
envoy_http_downstream_cx_http2_active{}	25
envoy_http_downstream_cx_http3_active{}	25
envoy_http_downstream_cx_rx_bytes_buffered{}	25
envoy_http_downstream_cx_ssl_active{}	25
envoy_http_downstream_cx_tx_bytes_buffered{}	25
envoy_http_downstream_cx_upgrades_active{}	25
envoy_http_downstream_rq_active{}	25
envoy_http_rds_config_reload_time_ms{}	10
envoy_http_rds_connected_state{}	10
envoy_http_rds_pending_requests{}	10
envoy_http_rds_update_time{}	10
envoy_http_rds_version{}	10
envoy_http_downstream_cx_length_ms_bucket{}	500
envoy_http_downstream_cx_length_ms_sum{}	25
envoy_http_downstream_cx_length_ms_count{}	25
envoy_http_downstream_rq_time_bucket{}	500
envoy_http_downstream_rq_time_sum{}	25
envoy_http_downstream_rq_time_count{}	25
envoy_http_rds_update_duration_bucket{}	200
envoy_http_rds_update_duration_sum{}	10
envoy_http_rds_update_duration_count{}	10
cilium_operator_feature_adv_connect_and_lb_gateway_api_enabled{}	2
cilium_operator_feature_adv_connect_and_lb_ingress_controller_enabled{}	2
kube_service_status_load_balancer_ingress{}	2
prometheus_operator_kubernetes_client_http_request_duration_seconds_sum{}	28
prometheus_operator_kubernetes_client_http_request_duration_seconds_count{}	28
prometheus_operator_kubernetes_client_http_requests_total{}	5
alertmanager_http_concurrency_limit_exceeded_total{}	1
alertmanager_http_request_duration_seconds_bucket{}	60
alertmanager_http_request_duration_seconds_sum{}	5
alertmanager_http_request_duration_seconds_count{}	5
alertmanager_http_requests_in_flight{}	1
alertmanager_http_response_size_bytes_bucket{}	32
alertmanager_http_response_size_bytes_sum{}	4
alertmanager_http_response_size_bytes_count{}	4
hubble_metrics_http_handler_request_duration_seconds_bucket{}	60
hubble_metrics_http_handler_request_duration_seconds_sum{}	5
hubble_metrics_http_handler_request_duration_seconds_count{}	5
hubble_metrics_http_handler_requests_total{}	5
grafana_http_request_duration_seconds_bucket{}	52
grafana_http_request_duration_seconds_sum{}	4
grafana_http_request_duration_seconds_count{}	4
grafana_http_request_in_flight{}	1
grafana_http_response_size_bytes_bucket{}	68
grafana_http_response_size_bytes_sum{}	4
grafana_http_response_size_bytes_count{}	4
hubble_http_request_duration_seconds_bucket{}	24
hubble_http_request_duration_seconds_sum{}	2
hubble_http_request_duration_seconds_count{}	2
hubble_http_requests_total{}	3
envoy_http_user_agent_downstream_cx_destroy_remote_active_rq{}	1
envoy_http_user_agent_downstream_cx_total{}	1
envoy_http_user_agent_downstream_rq_total{}	1
envoy_http_user_agent_downstream_cx_length_ms_bucket{}	20
envoy_http_user_agent_downstream_cx_length_ms_sum{}	1
envoy_http_user_agent_downstream_cx_length_ms_count{}	1
prometheus_http_request_duration_seconds_bucket{}	130
prometheus_http_request_duration_seconds_sum{}	13
prometheus_http_request_duration_seconds_count{}	13
prometheus_http_requests_total{}	62
prometheus_http_response_size_bytes_bucket{}	117
prometheus_http_response_size_bytes_sum{}	13
prometheus_http_response_size_bytes_count{}	13
prometheus_sd_http_failures_total{}	1
prometheus_sd_kubernetes_http_request_duration_seconds_sum{}	3
prometheus_sd_kubernetes_http_request_duration_seconds_count{}	3
prometheus_sd_kubernetes_http_request_total{}
```

## 5. Cilium/Hubble metrics

```promql
count by (__name__) (
  {__name__=~".*hubble.*|.*cilium.*"}
)
```

## Result

```shell
cilium_act_processing_time_seconds_bucket{}	60
cilium_act_processing_time_seconds_sum{}	5
cilium_act_processing_time_seconds_count{}	5
cilium_agent_bootstrap_seconds{}	20
cilium_bpf_map_capacity{}	117
cilium_bpf_map_ops_total{}	76
cilium_bpf_map_pressure{}	35
cilium_bpf_maps{}	5
cilium_bpf_maps_virtual_memory_max_bytes{}	5
cilium_bpf_progs{}	5
cilium_bpf_progs_virtual_memory_max_bytes{}	5
cilium_clustermesh_remote_clusters{}	5
cilium_controllers_failing{}	5
cilium_controllers_group_runs_total{}	5
cilium_controllers_runs_duration_seconds_bucket{}	72
cilium_controllers_runs_duration_seconds_sum{}	6
cilium_controllers_runs_duration_seconds_count{}	6
cilium_controllers_runs_total{}	6
cilium_drift_checker_config_delta{}	5
cilium_drop_bytes_total{}	9
cilium_drop_count_total{}	9
cilium_endpoint{}	5
cilium_endpoint_regenerations_total{}	10
cilium_endpoint_restoration_duration_seconds{}	25
cilium_endpoint_restoration_endpoints{}	70
cilium_endpoint_state{}	22
cilium_errors_warnings_total{}	20
cilium_event_ts{}	89
cilium_feature_adv_connect_and_lb_bandwidth_manager_enabled{}	5
cilium_feature_adv_connect_and_lb_bgp_enabled{}	5
cilium_feature_adv_connect_and_lb_cilium_envoy_config_enabled{}	5
cilium_feature_adv_connect_and_lb_cilium_node_config_enabled{}	5
cilium_feature_adv_connect_and_lb_egress_gateway_enabled{}	5
cilium_feature_adv_connect_and_lb_envoy_proxy_enabled{}	5
cilium_feature_adv_connect_and_lb_kube_proxy_replacement_enabled{}	5
cilium_feature_adv_connect_and_lb_l2_lb_enabled{}	5
cilium_feature_adv_connect_and_lb_l2_pod_announcement_enabled{}	5
cilium_feature_adv_connect_and_lb_node_port_configuration{}	5
cilium_feature_adv_connect_and_lb_sctp_enabled{}	5
cilium_feature_adv_connect_and_lb_vtep_enabled{}	5
cilium_feature_controlplane_cilium_endpoint_slices_enabled{}	5
cilium_feature_controlplane_identity_allocation{}	5
cilium_feature_controlplane_ipam{}	5
cilium_feature_datapath_chaining_enabled{}	5
cilium_feature_datapath_config{}	5
cilium_feature_datapath_endpoint_routes_enabled{}	5
cilium_feature_datapath_internet_protocol{}	5
cilium_feature_datapath_kernel_version{}	5
cilium_feature_datapath_network{}	5
cilium_feature_network_policies_cilium_envoy_config_total{}	5
cilium_feature_network_policies_host_firewall_enabled{}	5
cilium_feature_network_policies_internal_traffic_policy_services_total{}	5
cilium_feature_network_policies_local_redirect_policy_enabled{}	5
cilium_feature_network_policies_mutual_auth_enabled{}	5
cilium_feature_network_policies_non_defaultdeny_policies_enabled{}	5
cilium_forward_bytes_total{}	10
cilium_forward_count_total{}	10
cilium_fqdn_gc_deletions_total{}	5
cilium_fqdn_selectors{}	5
cilium_hive_jobs_observer_last_run_duration_seconds{}	20
cilium_hive_jobs_observer_run_duration_seconds_bucket{}	240
cilium_hive_jobs_observer_run_duration_seconds_sum{}	20
cilium_hive_jobs_observer_run_duration_seconds_count{}	20
cilium_hive_jobs_oneshot_last_run_duration_seconds{}	120
cilium_hive_jobs_runs_total{}	205
cilium_hive_jobs_timer_last_run_duration_seconds{}	65
cilium_hive_jobs_timer_run_duration_seconds_bucket{}	780
cilium_hive_jobs_timer_run_duration_seconds_sum{}	65
cilium_hive_jobs_timer_run_duration_seconds_count{}	65
cilium_hive_status{}	17
cilium_identity{}	10
cilium_identity_label_sources{}	10
cilium_identity_updater_timer_duration_bucket{}	60
cilium_identity_updater_timer_duration_sum{}	5
cilium_identity_updater_timer_duration_count{}	5
cilium_identity_updater_timer_trigger_folds_bucket{}	60
cilium_identity_updater_timer_trigger_folds_sum{}	5
cilium_identity_updater_timer_trigger_folds_count{}	5
cilium_identity_updater_timer_trigger_latency_bucket{}	60
cilium_identity_updater_timer_trigger_latency_sum{}	5
cilium_identity_updater_timer_trigger_latency_count{}	5
cilium_ip_addresses{}	10
cilium_ipam_capacity{}	5
cilium_ipam_events_total{}	7
cilium_ipcache_errors_total{}	6
cilium_k8s_client_api_calls_total{}	49
cilium_k8s_client_api_latency_time_seconds_bucket{}	648
cilium_k8s_client_api_latency_time_seconds_sum{}	54
cilium_k8s_client_api_latency_time_seconds_count{}	54
cilium_k8s_client_rate_limiter_duration_seconds_bucket{}	65
cilium_k8s_client_rate_limiter_duration_seconds_sum{}	5
cilium_k8s_client_rate_limiter_duration_seconds_count{}	5
cilium_k8s_terminating_endpoints_events_total{}	5
cilium_k8s_workqueue_adds_total{}	45
cilium_k8s_workqueue_depth{}	45
cilium_k8s_workqueue_longest_running_processor_seconds{}	45
cilium_k8s_workqueue_queue_duration_seconds_bucket{}	495
cilium_k8s_workqueue_queue_duration_seconds_sum{}	45
cilium_k8s_workqueue_queue_duration_seconds_count{}	45
cilium_k8s_workqueue_retries_total{}	45
cilium_k8s_workqueue_unfinished_work_seconds{}	45
cilium_k8s_workqueue_work_duration_seconds_bucket{}	495
cilium_k8s_workqueue_work_duration_seconds_sum{}	45
cilium_k8s_workqueue_work_duration_seconds_count{}	45
cilium_kubernetes_events_received_total{}	59
cilium_kubernetes_events_total{}	30
cilium_localredirectpolicy_controller_duration_seconds_bucket{}	60
cilium_localredirectpolicy_controller_duration_seconds_sum{}	5
cilium_localredirectpolicy_controller_duration_seconds_count{}	5
cilium_nat_endpoint_max_connection{}	10
cilium_nodes_all_datapath_validations_total{}	5
cilium_nodes_all_events_received_total{}	14
cilium_nodes_all_num{}	5
cilium_policy{}	5
cilium_policy_change_total{}	10
cilium_policy_implementation_delay_bucket{}	180
cilium_policy_implementation_delay_sum{}	15
cilium_policy_implementation_delay_count{}	15
cilium_policy_incremental_update_duration_bucket{}	45
cilium_policy_incremental_update_duration_sum{}	5
cilium_policy_incremental_update_duration_count{}	5
cilium_policy_l7_total{}	40
cilium_policy_max_revision{}	5
cilium_policy_selector_cache_identities{}	5
cilium_policy_selector_cache_operation_duration_seconds_bucket{}	90
cilium_policy_selector_cache_operation_duration_seconds_sum{}	10
cilium_policy_selector_cache_operation_duration_seconds_count{}	10
cilium_policy_selector_cache_selectors{}	5
cilium_policy_selector_match_count_max{}	20
cilium_process_cpu_seconds_total{}	5
cilium_process_max_fds{}	5
cilium_process_network_receive_bytes_total{}	5
cilium_process_network_transmit_bytes_total{}	5
cilium_process_open_fds{}	5
cilium_process_resident_memory_bytes{}	5
cilium_process_start_time_seconds{}	5
cilium_process_virtual_memory_bytes{}	5
cilium_process_virtual_memory_max_bytes{}	5
cilium_subprocess_start_total{}	5
cilium_version{}	5
hubble_flows_processed_total{}	586
hubble_lost_events_total{}	15
hubble_tcp_flags_total{}	20
cilium_agent_api_process_time_seconds_bucket{}	384
cilium_agent_api_process_time_seconds_sum{}	32
cilium_agent_api_process_time_seconds_count{}	32
cilium_datapath_conntrack_dump_resets_total{}	5
cilium_datapath_conntrack_gc_duration_seconds_bucket{}	120
cilium_datapath_conntrack_gc_duration_seconds_sum{}	10
cilium_datapath_conntrack_gc_duration_seconds_count{}	10
cilium_datapath_conntrack_gc_entries{}	20
cilium_datapath_conntrack_gc_interval_seconds{}	5
cilium_datapath_conntrack_gc_key_fallbacks_total{}	10
cilium_datapath_conntrack_gc_runs_total{}	10
cilium_endpoint_regeneration_time_stats_seconds_bucket{}	960
cilium_endpoint_regeneration_time_stats_seconds_sum{}	80
cilium_endpoint_regeneration_time_stats_seconds_count{}	80
cilium_policy_endpoint_enforcement_status{}	35
cilium_unreachable_health_endpoints{}	5
cilium_unreachable_nodes{}	5
cilium_xds_events_count{}	10
cilium_node_health_connectivity_latency_seconds_bucket{}	280
cilium_node_health_connectivity_latency_seconds_sum{}	20
cilium_node_health_connectivity_latency_seconds_count{}	20
cilium_node_health_connectivity_status{}	30
cilium_bpf_ratelimit_dropped_total{}	5
hubble_drop_total{}	12
cilium_api_limiter_adjustment_factor{}	7
cilium_api_limiter_processed_requests_total{}	7
cilium_api_limiter_processing_duration_seconds{}	14
cilium_api_limiter_rate_limit{}	14
cilium_api_limiter_requests_in_flight{}	14
cilium_api_limiter_wait_duration_seconds{}	21
envoy_cilium_access_denied{}	5
envoy_cilium_npds_control_plane_rate_limit_enforced{}	5
envoy_cilium_npds_init_fetch_timeout{}	5
envoy_cilium_npds_update_attempt{}	5
envoy_cilium_npds_update_failure{}	5
envoy_cilium_npds_update_rejected{}	5
envoy_cilium_npds_update_success{}	5
envoy_cilium_policy_tls_wrapper_missing_policy{}	5
envoy_cilium_policy_updates_rejected{}	5
envoy_cilium_policy_updates_total{}	5
envoy_cluster_cilium_access_denied{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_control_plane_rate_limit_enforced{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_init_fetch_timeout{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_key_rotation_failed{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_update_attempt{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_update_failure{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_update_rejected{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_update_success{}	5
envoy_cilium_npds_control_plane_connected_state{}	5
envoy_cilium_npds_control_plane_pending_requests{}	5
envoy_cilium_npds_update_time{}	5
envoy_cilium_npds_version{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_control_plane_connected_state{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_control_plane_pending_requests{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_update_time{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_version{}	5
envoy_cilium_npds_update_duration_bucket{}	100
envoy_cilium_npds_update_duration_sum{}	5
envoy_cilium_npds_update_duration_count{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_update_duration_bucket{}	100
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_update_duration_sum{}	5
envoy_sds_cilium_secrets_deploy_confidence_deploy_confidence_lan_tls_update_duration_count{}	5
cilium_operator_clustermesh_remote_clusters{}	2
cilium_operator_doublewrite_crd_identities{}	2
cilium_operator_doublewrite_crd_only_identities{}	2
cilium_operator_doublewrite_kvstore_identities{}	2
cilium_operator_doublewrite_kvstore_only_identities{}	2
cilium_operator_errors_warnings_total{}	6
cilium_operator_feature_adv_connect_and_lb_gateway_api_enabled{}	2
cilium_operator_feature_adv_connect_and_lb_ingress_controller_enabled{}	2
cilium_operator_feature_adv_connect_and_lb_l7_aware_traffic_management_enabled{}	2
cilium_operator_feature_adv_connect_and_lb_lb_ipam_enabled{}	2
cilium_operator_feature_adv_connect_and_lb_node_ipam_enabled{}	2
cilium_operator_lbipam_conflicting_pools{}	2
cilium_operator_lbipam_services_matching{}	2
cilium_operator_lbipam_services_unsatisfied{}	2
cilium_operator_number_of_ceps_per_ces_bucket{}	18
cilium_operator_number_of_ceps_per_ces_sum{}	2
cilium_operator_number_of_ceps_per_ces_count{}	2
cilium_operator_process_cpu_seconds_total{}	2
cilium_operator_process_max_fds{}	2
cilium_operator_process_network_receive_bytes_total{}	2
cilium_operator_process_network_transmit_bytes_total{}	2
cilium_operator_process_open_fds{}	2
cilium_operator_process_resident_memory_bytes{}	2
cilium_operator_process_start_time_seconds{}	2
cilium_operator_process_virtual_memory_bytes{}	2
cilium_operator_process_virtual_memory_max_bytes{}	2
cilium_operator_unmanaged_pods{}	2
cilium_operator_version{}	2
hubble_relay_pool_peer_connection_status{}	12
hubble_metrics_http_handler_request_duration_seconds_bucket{}	60
hubble_metrics_http_handler_request_duration_seconds_sum{}	5
hubble_metrics_http_handler_request_duration_seconds_count{}	5
hubble_metrics_http_handler_requests_total{}	5
cilium_operator_hive_jobs_runs_total{}	6
cilium_operator_hive_jobs_timer_last_run_duration_seconds{}	2
cilium_operator_hive_jobs_timer_run_duration_seconds_bucket{}	24
cilium_operator_hive_jobs_timer_run_duration_seconds_sum{}	2
cilium_operator_hive_jobs_timer_run_duration_seconds_count{}	2
hubble_http_request_duration_seconds_bucket{}	24
hubble_http_request_duration_seconds_sum{}	2
hubble_http_request_duration_seconds_count{}	2
hubble_http_requests_total{}	3
cilium_hive_jobs_runs_failed{}	4
cilium_operator_feature_controlplane_kubernetes_version{}	1
cilium_operator_hive_jobs_observer_last_run_duration_seconds{}	2
cilium_operator_hive_jobs_observer_run_duration_seconds_bucket{}	24
cilium_operator_hive_jobs_observer_run_duration_seconds_sum{}	2
cilium_operator_hive_jobs_observer_run_duration_seconds_count{}	2
cilium_operator_hive_jobs_oneshot_last_run_duration_seconds{}	2
cilium_operator_identity_gc_entries{}	2
cilium_operator_identity_gc_latency{}	1
cilium_operator_identity_gc_runs{}	1
cilium_operator_lbipam_event_processing_time_seconds_bucket{}	24
cilium_operator_lbipam_event_processing_time_seconds_sum{}	2
cilium_operator_lbipam_event_processing_time_seconds_count{}	2
cilium_operator_lbipam_ips_available{}	1
cilium_operator_lbipam_ips_used{}	1

```

## Initial conclusion

Pending.

The goal is to identify whether user-facing SLOs can be measured from:

- ingress/edge metrics
- frontend application metrics
- Cilium/Hubble metrics
- fallback workload runtime metrics

---

# Step 7 — Discovery in Prometheus

Run these in Prometheus UI or API.

## 1. All workload metrics

```promql
count by (__name__) (
  {namespace="fintech-workload"}
)
```

## 2. HTTP/request metrics

```promql
count by (__name__) (
  {__name__=~".*http.*|.*request.*|.*response.*", namespace="fintech-workload"}
)
```

## 3. Latency metrics

```promql
count by (__name__) (
  {__name__=~".*duration.*|.*latency.*|.*seconds.*", namespace="fintech-workload"}
)
```

## 4. Ingress/gateway/edge metrics

```promql
count by (__name__) (
  {__name__=~".*ingress.*|.*nginx.*|.*gateway.*|.*http.*"}
)
```

## 5. Cilium/Hubble metrics

```promql
count by (__name__) (
  {__name__=~".*hubble.*|.*cilium.*"}
)
```

## Result

The query returned metrics for the fintech-workload namespace.

Important available metric groups:

### Kubernetes workload state
- kube_pod_container_status_restarts_total 
- kube_pod_container_status_ready 
- kube_pod_status_phase 
- kube_deployment_status_replicas 
- kube_deployment_status_replicas_available 
- kube_deployment_status_replicas_unavailable

### Container resource metrics
- container_cpu_usage_seconds_total 
- container_memory_working_set_bytes 
- container_memory_usage_bytes 
- container_oom_events_total

### Container network metrics
- container_network_receive_bytes_total 
- container_network_receive_errors_total 
- container_network_transmit_bytes_total 
- container_network_transmit_errors_total

### Probe metrics
- prober_probe_total 
- prober_probe_duration_seconds_bucket 
- prober_probe_duration_seconds_sum 
- prober_probe_duration_seconds_count

## Interpretation

> Prometheus has strong workload runtime and platform investigation signals for fintech-workload.
> The available metrics can support investigation of pod restarts, readiness, deployment availability, CPU/memory pressure, OOM events, and network errors.
> The presence of prober_* metrics suggests that probe-based availability or latency SLIs may be possible.
> No obvious application-native HTTP request counter or request duration metric was confirmed from this first discovery output. Further HTTP/request and latency discovery queries are required.

## Final Phase 8 Conclusion

The discovery phase confirmed that the `fintech-workload` namespace currently exposes strong Kubernetes/runtime metrics, including deployment status, pod readiness, pod phase, pod restarts, CPU, memory, network, and OOM metrics.

However, no clean application-level HTTP request or latency metrics were discovered directly from Bank of Anthos in the `fintech-workload` namespace.

Global Envoy and Hubble HTTP metrics are available, but require further label inspection before they can be used as Bank of Anthos SLO sources.

The recommended next step is to implement the first user-facing SLO using blackbox/prober metrics against the Bank of Anthos frontend endpoint.