# Global Cache (GC) Tutorials

## Connect to Global Cache Brokers

Connect to Global Cache brokers; examine the messages being (re-)published.

There are 6 Global Caches:

1. **China:** `cn-cma-global-cache`
   - `BROKER=mqtts://gc.wis.cma.cn:8883`
   - `USERNAME=everyone`
   - `PASSWORD=Everyone@100081&`

2. **UK/USA:** `data-metoffice-noaa-global-cache`
   - `BROKER=mqtts://wis2cache.globaldata.nws.noaa.gov:8883`
   - `USERNAME=everyone`
   - `PASSWORD=everyone`

3. **Germany:** `de-dwd-global-cache`
   - `BROKER=mqtts://wis2.dwd.de:8883`
   - `USERNAME=everyone`
   - `PASSWORD=everyone`

4. **Japan:** `jp-jma-global-cache`
   - `BROKER=mqtts://mqtt.wis-jma.go.jp`
   - `USERNAME=france`
   - `PASSWORD=juz*ai~v0eiNieT8bi`

5. **South Korea:** `kr-kma-global-cache`
   - `BROKER=mqtts://wis2gc.kma.go.kr:8883`
   - `USERNAME=everyone`
   - `PASSWORD=everyone`

6. **Saudi Arabia:** `sa-ncm-global-cache`
   - `BROKER=mqtts://wis2.ncm.gov.sa:8883`
   - `USERNAME=everyone`
   - `PASSWORD=everyone`

---

### Configure a Connection to a Global Cache Broker

Use MQTT Explorer

| Field | Value |
|-------|-------|
| Name | `{xxx-global-cache}` |
| Validate certificate | yes |
| Encryption (tls) | yes |
| Protocol | `mqtt://` |
| Host | `{GLOBAL_CACHE_HOST}` |
| Port | `8883` |
| Username | `{USERNAME}` |
| Password | `{PASSWORD}` |

**ADVANCED:** Add a subscription to start receiving messages:

```
cache/a/wis2/#
```

Image: [data-metoffice-noaa-global-cache connection details for MQTT explorer](https://github.com/wmo-im/wis2-champions/blob/main/Bali-workshop-MAR2026/data-metoffice-noaa-global-cache-mqtt-connection.png)

Now investigate some of the messages being received.

---

### Browse the S3 Bucket of data-metoffice-noaa-global-cache

https://wis2globalcache.s3.amazonaws.com/index.html

---

## Design a Message / Data De-duplication Algorithm

### Lambda Manager — Pseudocode

The Lambda Manager sits in the WIS2 global cache pipeline. It consumes batches of data-availability notification messages from SQS, deduplicates them via Redis, downloads and archives the actual data to S3 when needed, then republishes the notification to downstream MQTT subscribers. Errors at any stage are caught, logged, and themselves published to a dedicated MQTT error topic so they remain observable.

---

#### Main Entry Point

`msg_handler(msg_batch)` — main entry point into the Lambda.

```
clean up any leftover files in /tmp

FOR EACH sqs_msg IN msg_batch:

    TRY:
        parse message body as JSON
        construct Wis2Message object from body

        extract centre_id from topic

        check Redis to see if this data_id has been seen before

        IF message is not unique (already cached with same or newer pubtime):
            skip (continue to next message)

        IF message should be cached (do_cache = true):
            TRY:
                download the data bytes
                upload to S3
                clean up tmp file and free memory

            EXCEPT invalid message structure (TypeError):
                increment error counter in Redis
                publish error notification to MQTT error topic
                skip message (no retry — structural issue)

            EXCEPT file/disk error (IOError):
                increment cache-override counter in Redis
                skip message (elect not to cache)

            EXCEPT anything else:
                re-raise (will be caught by outer try)

        re-check Redis uniqueness (race condition guard)
        IF still unique:
            write data_id → pubtime to Redis with 6hr TTL

        IF do_cache:
            increment downloaded_total counter in Redis
            update dataserver last-download timestamp in Redis
            set dataserver status flag = 1 (healthy) in Redis

        IF NOT do_cache:
            increment no_cache_total counter in Redis

        format the outbound WIS2 notification message

        FOR EACH broker:
            publish notification message to MQTT on new topic (MQTTv5, QoS 1, TLS)

    EXCEPT any unhandled error:
        log the error
        add message ID to batch_item_failures list
        publish error notification to MQTT error topic (with traceback)
        increment downloaded_errors_total counter in Redis
        set dataserver status flag = 0 (unhealthy) in Redis

return empty batchItemFailures response
```

---

#### Uniqueness Check

Each message is keyed by `data_id` (a unique identifier for the data object), and the value stored is `pubtime_epoch` — the publication timestamp of the message as a Unix epoch integer. The key expires after 6 hours (the `ttl_minutes = 360` setting).

Uniqueness is checked **twice**:

1. **Before** attempting to cache data:
```
last_cached = Redis.get(data_id)
if NOT is_unique(last_cached): skip
```

2. Download and upload to S3 (if `do_cache`)...

3. **After** caching data:
```
last_cached = Redis.get(data_id)   ← re-check
if NOT is_unique(last_cached): skip
else: Redis.set(data_id, pubtime_epoch, TTL=6hrs)
```

The second check is described as a "last minute dump" race condition guard — protecting against two Lambda instances processing the same message concurrently. Both might pass the first check before either has written to Redis, so the second check catches that collision.

---

#### `is_unique(last_cached)`

`last_cached` is a `pubtime_epoch` retrieved from Redis for a given `data_id`.

```
is_unique(last_cached):
    IF last_cached is None:
        return True  ← never seen this data_id before

    IF this message's pubtime_epoch > last_cached:
        return True  ← this is a newer version of the same data object (an update)

    ELSE:
        return False  ← already processed an equal or newer version, discard
```


---

## Use Prometheus to Query Global Cache Metrics

Use the Prometheus instance configured in your `{name}.champion.wis2dev.io` instance.

### Set Up a New Prometheus Scrape Job

1. Edit your `prometheus.yml` configuration to add a new target.

   Example scrape job for `data-metoffice-noaa-global-cache`:

   ```yaml
   - job_name: 'data-metoffice-noaa-global-cache'
     scheme: https
     metrics_path: '/metrics'
     static_configs:
       - targets: ['wis2cache.globaldata.nws.noaa.gov']
   ```

2. Restart your prometheus container:

   ```bash
   docker restart prometheus
   ```

3. In the Prometheus web app, check the health of the metrics scrape target:

   **Status → Target health**

4. Try out some queries.

   Example: increase in number of messages published by the GC, summed for all centre-ids:

   ```
   sum by (instance) (increase (wmo_wis2_gc_downloaded_total[1h]))
   ```
