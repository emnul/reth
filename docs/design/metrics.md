## Metrics

### Metrics or traces?

A **metric** is a numeric representation of data measured over intervals of time. Metrics are malleable to statistical transformations such as sampling, aggregation and correlation, which make them suited to report the overall health of a system.

A **trace** is a representation of a series of causally related distributed events that encode information about the end-to-end request flow through a distributed system. Traces are used to identify the amount of work done at each layer in an application while preserving causality.

The main difference between metrics and traces is therefore that metrics are system-centric and traces are request-centric: metrics give you insight into how a particular system is doing, while traces help teams identify the path of requests through various services.

**For most things, you likely want a metric**, except for two scenarios:

- For contributors, traces are a good profiling tool
- For end-users that run complicated infrastructure, traces in the RPC component makes sense

### How to add a metric

To add metrics use the [`metrics`][metrics] crate.

1. Add the code emitting the metric.
2. Add the metrics description in the crate's metrics describer module, e.g.: [network metrics describer](https://github.com/paradigmxyz/reth/blob/main/crates/net/network/src/metrics.rs).
3. Document the metric in this file.

#### Metric anatomy

There are three types of metrics:

- **Counters**: Represent (ideally) monotonically increasing values, e.g. the number of errors that have occurred, the number of blocks processed, etc.
- **Gauges**: Represent metrics that can go up or down arbitrarily over time. Usually they are used to measure things like resource usage (memory, CPU, ...) and throughput.
- **Histograms**: Used to store an arbitrary number of observations of a specific measurement, and provides statistical analysis over the observed values. A typical use case is latency of some operation (writing to disk, responding to a request, ...).

Each metric is identified by a [`Key`][metrics.Key], which itself is composed of a [`KeyName`][metrics.KeyName] and an arbitrary number of [`Label`][metrics.Label]s.

The `KeyName` represents the actual metric name, and the labels are used to further drill down into the metric.

For example, a metric that represents stage progress would have a key name of `stage_progress` and a `stage` label that can be used to get the progress of individual stages.

There will only ever exist one description per metric `KeyName`; it is not possible to add a description for a label, or a `KeyName`/`Label` group.

#### Creating metrics

The `metrics` crate provides three macros per metric variant: `register_<metric>!`, `<metric>!`, and `describe_<metric>!`. Prefer to use these where possible, since they generate the code necessary to register and update metrics under various conditions.

- The `register_<metric>!` macro simply creates the metric and returns a handle to it (e.g. a `Counter`). These metric structs are thread-safe and cheap to clone.
- The `<metric>!` macro registers the metric if it does not exist, and updates it's value.
- The `describe_<metric>!` macro adds an end-user description for the metric.

How the metrics are exposed to the end-user is determined by the CLI.

### Metric best practices

- Use `.` to namespace metrics
  - The top-level namespace should **NOT** be `reth`[^1]
- Metric names should not contain spaces
- Add a unit to the metric where appropriate
  - Use the Prometheus [base units][prom_base_units]
- Do not add rate-metrics
  - Rates can be calculated by e.g. Prometheus on the fly
- Avoid duplicate metrics
  - An example would be adding two metrics for connections: `reth.p2p.connections` for current connections and `reth.p2p.connections.total` for total connections. One of these metrics can be used to infer the other.

[^1]: The top-level namespace is added by the CLI using [`metrics_util::layers::PrefixLayer`][metrics_util.PrefixLayer].

### Current metrics

This list may be non-exhaustive.

#### Stage: Headers

- `stages.headers.counter`: Number of headers successfully retrieved
- `stages.headers.timeout_error`: Number of timeout errors while requesting headers
- `stages.headers.validation_errors`: Number of validation errors while requesting headers
- `stages.headers.unexpected_errors`: Number of unexpected errors while requesting headers
- `stages.headers.request_time`: Elapsed time of successful header requests

#### Component: Transaction Pool

- `transaction_pool.inserted_transactions`: Number of transactions inserted in the pool
- `transaction_pool.invalid_transactions`: Number of invalid transactions
- `transaction_pool.removed_transactions`: Number of removed transactions from the pool
- `transaction_pool.pending_pool_transactions`: Number of transactions in the pending sub-pool
- `transaction_pool.pending_pool_size_bytes`: Total amount of memory used by the transactions in the pending sub-pool in bytes
- `transaction_pool.basefee_pool_transactions`: Number of transactions in the basefee sub-pool
- `transaction_pool.basefee_pool_size_bytes`: Total amount of memory used by the transactions in the basefee sub-pool in bytes
- `transaction_pool.queued_pool_transactions`: Number of transactions in the queued sub-pool
- `transaction_pool.queued_pool_size_bytes`: Total amount of memory used by the transactions in the queued sub-pool in bytes
- `transaction_pool.blob_pool_transactions`: Number of transactions in the blob sub-pool
- `transaction_pool.blob_pool_size_bytes`: Total amount of memory used by the transactions in the blob sub-pool in bytes

#### Component: Network

##### struct NetworkMetrics
Metrics for the entire network, handled by `NetworkManager`

- `network.connected_peers`: Number of currently connected peers
- `network.backed_off_peers`: Number of currently backed off peers
- `network.tracked_peers`: Number of peers known to the node
- `network.pending_session_failures`: Cumulative number of failures of pending sessions
- `network.closed_sessions`: Total number of sessions closed
- `network.incoming_connections`: Number of active incoming connections
- `network.outgoing_connections`: Number of active outgoing connections
- `network.pending_outgoing_connections`: Number of currently pending outgoing connections
- `network.pending_connections_total`: Total number of pending connections, incoming and outgoing
- `network.incoming_connections_total`: Total number of incoming connections handled
- `network.outgoing_connections_total`: Total number of outgoing connections established
- `network.invalid_messages_received`: Number of invalid/malformed messages received from peers
- `network.dropped_eth_requests_at_full_capacity_total`: Number of Eth Requests dropped due to channel being at full capacity
- `network.network_manager_poll_duration_total_seconds`: Duration in seconds of call to `NetworkManager`'s poll function
- `network.network_handle_acc_poll_duration_seconds`: Time spent streaming messages sent over the `NetworkHandle`, which
can be cloned and shared via `NetworkManager::handle`, in one call to poll the `NetworkManager` future. At least `TransactionsManager` holds this handle
- `network.swarm_acc_poll_duration_seconds`: Time spent polling `Swarm`, in one call to poll the `NetworkManager` future

##### SessionManagerMetrics
Metrics for `SessionManager`

- `network.incoming_eth_handshake_peer_error_total`: Total number of errors related to incoming incorrect peer behaviour such as invalid message code, size, encoding, etc
- `network.outgoing_eth_handshake_peer_error_total`: Total number of errors related to outgoing incorrect peer behaviour such as invalid message code, size, encoding, etc
- `network.incoming_eth_handshake_timeout_total`: Total number of incoming timeout errors
- `network.outgoing_eth_handshake_timeout_total`: Total number of outgoing timeout errors
- `network.outgoing_eth_handshake_network_error_total`: Total number of outgoing network id mismatch errors
- `network.incoming_eth_handshake_version_error_total`: Total number of incoming protocol version mismatch errors
- `network.outgoing_eth_handshake_version_error_total`: Total number of outgoing protocol version mismatch errors
- `network.incoming_eth_handshake_genesis_error_total`: Total number of incoming genesis block mismatch errors
- `network.outgoing_eth_handshake_genesis_error_total`: Total number of outgoing genesis block mismatch errors
- `network.incoming_eth_handshake_forkid_error_total`: Total number of incoming fork id mismatch errors
- `network.outgoing_eth_handshake_forkid_error_total`: Total number of outgoing fork id mismatch errors
- `network.dial_successes_total`: Total number of successful outgoing dial attempts
- `network.outgoing_peer_messages_dropped_total`: Total number of dropped outgoing peer messages

##### TransactionsManagerMetrics
Metrics for the `TransactionsManager`.

- `network.propagated_transactions_total`: Total number of propagated transactions
- `network.reported_bad_transactions_total`: Total number of reported bad transactions
- `network.messages_with_hashes_already_seen_by_peer_total`: Total number of messages from a peer, announcing transactions that have already been marked as seen by that peer
- `network.messages_with_transactions_already_seen_by_peer_total`: Total number of messages from a peer, with transaction that have already been marked as seen by that peer.
- `network.hash_already_seen_by_peer_occurrences_total`: Total number of occurrences, of a peer announcing a transaction that has already been marked as seen by that peer
- `network.transaction_already_seen_by_peer_occurrences_total`: Total number of times a transaction is seen from a peer, that has already been marked as seen by that peer
- `network.hashes_already_in_pool_occurrences_total`: Total number of times a hash is announced that is already in the local pool
- `network.transactions_already_in_pool_occurrences_total`: Total number of times a transaction is sent that is already in the local pool
- `network.pending_pool_imports`: Number of transactions about to be imported into the pool
- `network.bad_imports_total`: Total number of bad imports, imports that fail because the transaction is badly formed(i.e. have no chance of passing validation, unlike imports that fail due to e.g. nonce gaps)
- `network.capacity_pending_pool_imports`: Number of inflight requests at which the `TransactionPool` is considered to be at capacity. Note, this is not a limit to the number of inflight requests, but a health measure
- `network.tx_manager_acc_poll_duration_seconds`: Duration in seconds of call to `TransactionsManager`'s poll function
- `network.network_events_acc_poll_duration_seconds`: Accumulated time spent streaming session updates and updating peers accordingly, in one call to poll the `TransactionsManager` future
- `network.pending_pool_imports_acc_poll_duration_seconds`: Accumulated time spent flushing the queue of batched pending pool imports into pool, in one call to poll the `TransactionsManager` future
- `network.transaction_events_acc_poll_duration_seconds`: Accumulated time spent streaming transaction and announcement broadcast, queueing for pool import or requesting respectively, in one call to poll the `TransactionsManager` future
- `network.fetch_events_acc_poll_duration_seconds`: Accumulated time spent streaming fetch events, queueing for pool import on successful fetch, in one call to poll the `TransactionsManager` future
- `network.imported_transactions_acc_poll_duration_seconds`: Accumulated time spent streaming and propagating transactions that were successfully imported into the pool, in one call to poll the `TransactionsManager` future
- `network.fetch_pending_hashes_acc_duration_seconds`: Accumulated time spent assembling and sending requests for hashes fetching pending, in one call to poll the [`TransactionsManager` future
- `network.poll_commands_acc_duration_seconds`: Accumulated time spent streaming commands and propagating, fetching and serving transactions accordingly, in one call to poll the `TransactionsManager` future

##### TransactionFetcherMetrics
Metrics for the `TransactionsManager`.

- `network.inflight_transaction_requests`: Currently active outgoing `GetPooledTransactions` requests
- `network.capacity_inflight_requests`: Number of inflight requests at which the `TransactionFetcher` is considered to be at capacity. Note, this is not a limit to the number of inflight requests, but a health measure
- `network.hashes_inflight_transaction_requests`: Hashes in currently active outgoing `GetPooledTransactions` requests
- `network.egress_peer_channel_full`: How often we failed to send a request to the peer because the channel was full
- `network.hashes_pending_fetch_total`: Total number of hashes pending fetch
- `network.fetched_transactions_total`: Total number of fetched transactions
- `network.unsolicited_transactions_total`: Total number of transactions that were received in `PooledTransactions` responses, that weren't requested
- `network.find_idle_fallback_peer_for_any_pending_hash_duration_seconds`: Time spent searching for an idle peer in call to `TransactionFetcher::find_any_idle_fallback_peer_for_any_pending_hash`
- `network.fill_request_from_hashes_pending_fetch_duration_seconds`: Time spent searching for hashes pending fetch, announced by a given peer in `TransactionFetcher::fill_request_from_hashes_pending_fetch`

##### DisconnectMetrics
Metrics for Disconnection types

These are just counters, and ideally we would implement these metrics on a peer-by-peer basis, in that we do not double-count peers for `TooManyPeers` if we make an outgoing connection and get disconnected twice

- `network.disconnect_requested`: Number of peer disconnects due to `DisconnectRequested` (0x00)
- `network.tcp_subsystem_error`: Number of peer disconnects due to `TcpSubsystemError` (0x01)
- `network.protocol_breach`: Number of peer disconnects due to `ProtocolBreach` (0x02)
- `network.useless_peer`: Number of peer disconnects due to `UselessPeer` (0x03)
- `network.too_many_peers`: Number of peer disconnects due to `TooManyPeers` (0x04)
- `network.already_connected`: Number of peer disconnects due to `AlreadyConnected` (0x05)
- `network.incompatible`: Number of peer disconnects due to `IncompatibleP2PProtocolVersion` (0x06)
- `network.null_node_identity`: Number of peer disconnects due to `NullNodeIdentity` (0x07)
- `network.client_quitting`: Number of peer disconnects due to `ClientQuitting` (0x08)
- `network.unexpected_identity`: Number of peer disconnects due to `UnexpectedHandshakeIdentity` (0x09)
- `network.connected_to_self`: Number of peer disconnects due to `ConnectedToSelf` (0x0a)
- `network.ping_timeout`: Number of peer disconnects due to `PingTimeout` (0x0b)
- `network.subprotocol_specific`: Number of peer disconnects due to `SubprotocolSpecific` (0x10)

##### EthRequestHandlerMetrics
Metrics for the `EthRequestHandler`

- `network.eth_headers_requests_received_total`: Number of `GetBlockHeaders` requests received
- `network.eth_receipts_requests_received_total`: Number of `GetReceipts` requests received
- `network.eth_bodies_requests_received_total`: Number of `GetBlockBodies` requests received
- `network.eth_node_data_requests_received_total`: Number of `GetNodeData` requests received
- `network.eth_req_handler_poll_duration_seconds`: Duration in seconds of call to poll `EthRequestHandler`

##### AnnouncedTxTypesMetrics
Eth67 announcement metrics, track entries by `TxType`

- `network.transaction_fetcher.legacy`: Histogram for tracking frequency of legacy transaction type
- `network.transaction_fetcher.eip2930`: Histogram for tracking frequency of EIP-2930 transaction type
- `network.transaction_fetcher.eip1559`: Histogram for tracking frequency of EIP-1559 transaction type
- `network.transaction_fetcher.eip4844`: Histogram for tracking frequency of EIP-4844 transaction type
- `network.transaction_fetcher.eip7702`: Histogram for tracking frequency of EIP-7702 transaction type

[metrics]: https://docs.rs/metrics
[metrics.Key]: https://docs.rs/metrics/latest/metrics/struct.Key.html
[metrics.KeyName]: https://docs.rs/metrics/latest/metrics/struct.KeyName.html
[metrics.Label]: https://docs.rs/metrics/latest/metrics/struct.Label.html
[prom_base_units]: https://prometheus.io/docs/practices/naming/#base-units
[metrics_util.PrefixLayer]: https://docs.rs/metrics-util/latest/metrics_util/layers/struct.PrefixLayer.html
