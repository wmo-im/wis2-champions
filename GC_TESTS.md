# Global Cache Test Specification

## Functional Tests

### MQTT Broker Connectivity

An MQTT client must be able to connect to the local broker of the Global Cache on port 8883 using the MQTT protocol version 5 with TLS (i.e., mqtts protocol) and username/password authentication.

**Evaluate:**
1. Check if the connection is successful (rc code). If the connection is successful, the test passes. If the connection is not successful, the test fails.

---

### GC MQTT Broker Subscription

A Global Cache must allow connected MQTT clients to subscribe to the `cache/a/wis2/#` topic using a provided connection string.

**Evaluate:**
1. Check if the subscription is successful. If the subscription is successful based on the returned rc code (SUBACK), the test passes. If the subscription is not successful, the test fails.
2. Close the connection to the broker after the test.

---

### WIS2 Notification Message (WNM) Processing

Test that the GC functions as expected under normal conditions. The Global Cache should process a valid incoming WNM, download the data at the provided canonical link, and publish a new WNM on the proper `cache/` topic using the proper message structure, and update the necessary GC metrics. This test also evaluates the client data download requirement: An HTTP client (i.e., a Web browser) must be able to connect to the HTTP server of the Global Cache on port 443 using HTTP 1.1 with TLS but without any authentication and be able to resolve the URL provided in a data download link (a link object's `href` property where `rel=canonical`) from a notification message published by the Global Cache within the previous 24 hours; i.e., download a cached data item.

**Pre-requisites:**

Prepared WIS2 Notification Messages and associated data objects:
- A known number of valid WNMs with:
  - `properties.cache` set to `true`
  - `properties.data_id` + `properties.pubtime` should be unique to each message. Ensuring a different `data_id` is best here.
- Accompanying data objects should be accessible via the canonical link provided in the WNM.
  - The canonical link should be accessible per the core requirements and the data object hash should match the hash provided in the WNM if integrity properties are provided.

**Evaluate:**

1. WNM Messages
   - The total number of cache notification messages published by the GC on the `cache/a/wis2/#` topic.
   - All messages should be the same as the source WNMs except for:
     - The canonical link (a link object's `href` property where `rel=canonical`), this should point to the GC's cached object.
     - The unique identifier of the message (`id`).
     - The topic, always on the cache channel. Note the incoming message may be unchanged if it was originally published on the `cache` channel.
2. Data Objects
   - The total number of data objects cached by the GC. This should match the number of cache notification messages published.
   - The data objects cached by the GC should be identical to the source data objects.
     - The diff or hashes of the data objects should be identical.
3. GC Metrics
   - `wmo_wis2_gc_download_total` (incremented by the total number of incoming messages, i.e., +=1 for each WNM)
   - `wmo_wis2_gc_last_download_timestamp_seconds` (set for each and within expected time range)

---

### Cache False Directive

Where a Global Cache receives a notification message with `properties.cache` set to `false`, the Global Cache should publish a notification message where the data download link (a link object's `href` property where `rel=canonical`) refers to the source data server (i.e., the origin WIS2 Node).

**Pre-requisites:**

Prepared WIS2 Notification Messages:
- A known number of valid WNMs with:
  - `properties.cache` set to `false`
  - `properties.data_id` + `properties.pubtime` should be unique to each message.
- Accompanying data objects are not required for this test.

**Evaluate:**

1. WNM Messages
   - The total number of cache notification messages published by the GC on the `cache/a/wis2/#` topic.
   - All messages should be the same as the source WNMs except for:
     - The unique identifier of the message (`id`).
     - The topic, always on the cache channel. Note the incoming message may be unchanged if it was originally published on the `cache` channel.
2. Data Objects
   - No data objects should be cached by the GC.
3. GC Metrics
   - `wmo_wis2_gc_download_total` (unchanged)
   - `wmo_wis2_gc_last_download_timestamp_seconds` (unchanged)

---

### Source Download Failure

Where a Global Cache receives a valid WNM, but is unable to download a data item from the location specified in a notification message (i.e., the source data server), the metric `wmo_wis2_gc_download_errors_total` is incremented.

**Pre-requisites:**

Prepared WIS2 Notification Messages:
- A known number of valid WNMs with:
  - Invalid data download links (a link object's `href` property where `rel=canonical`)
  - `properties.data_id` + `properties.pubtime` should be unique to each message.
- Accompanying data objects are not required for this test.

**Evaluate:**

1. WNM Messages
   - No messages should be published on the `cache/a/wis2/#` topic as received by the test MQTT client.
2. Data Objects
   - No data objects should be cached by the GC.
3. GC Metrics
   - `wmo_wis2_gc_download_total` (unchanged)
   - `wmo_wis2_gc_last_download_timestamp_seconds` (unchanged)
   - `wmo_wis2_gc_downloaded_errors_total` (+=1 for each WNM)

**WIS2-to-GTS Gateway variant:** Provide 10 WNMs with no inline data and a bad (unresolvable) URL for the canonical data. The Global Cache should throw errors and no data should be published.

---

### Data Integrity Check Failure

A Global Cache should validate the integrity of the resources it caches and only accept data which matches the integrity value from the WIS Notification Message. If the WIS Notification Message does not contain an integrity value, a Global Cache should accept the data as valid. In this case a Global Cache may add an integrity value to the message it republishes.

**Pre-requisites:**

Prepared WIS2 Notification Messages:
- A known number of valid WNMs with:
  - Invalid data integrity value (accessed via `properties.integrity.value` and the method specified in `properties.integrity.method`)
  - `properties.data_id` + `properties.pubtime` should be unique to each message.
- Accompanying data objects that are accessible via the canonical link provided in the WNM.

**Evaluate:**

1. WNM Messages
   - No messages should be published on the `cache/a/wis2/#` topic as received by the test MQTT client.
2. Data Objects
   - No data objects should be cached by the GC.
3. GC Metrics
   - `wmo_wis2_gc_download_total` (unchanged)
   - `wmo_wis2_gc_last_download_timestamp_seconds` (unchanged)
   - `wmo_wis2_gc_downloaded_errors_total` (unchanged)
   - `wmo_wis2_gc_integrity_failed_total` (+=1 for each WNM)

**WIS2-to-GTS Gateway variant (inline data):** Provide 10 WNMs with inline data, a resolvable URL for the canonical data, and a bad method and/or value for `properties.integrity`. The Global Cache should trigger an error for validation of both inline data and data retrieved from URL; no data should be published.

**WIS2-to-GTS Gateway variant (URL data):** Provide 10 WNMs with no inline data, a resolvable URL for the canonical data, and a bad method and/or value for `properties.integrity`. The Global Cache should trigger an error for validation of data retrieved from URL; no data should be published.

---

### WIS2 Notification Message Deduplication

A Global Cache must ensure that only one instance of a notification message with a given unique identifier (`id`) is successfully processed.

**Pre-requisites:**

Prepared WIS2 Notification Messages:
- A known number of valid WNMs with:
  - `properties.data_id` + `properties.pubtime` are NOT unique to each message, but shared by 2 or more messages.
- Accompanying data objects that are accessible via the canonical link provided in the WNM.

**Evaluate:**

1. WNM Messages
   - Only one message should be published by the GC on the `cache/a/wis2/#` topic per unique identifier, defined as `properties.data_id` + `properties.pubtime`.
2. Data Objects
   - Only one data object should be cached per unique identifier, defined as `properties.data_id` + `properties.pubtime`.
3. GC Metrics
   - `wmo_wis2_gc_download_total` (+=1 for each unique identifier)
   - `wmo_wis2_gc_last_download_timestamp_seconds` (set to current for each unique identifier)
   - `wmo_wis2_gc_downloaded_errors_total` (unchanged)
   - `wmo_wis2_gc_integrity_failed_total` (unchanged)

---

### WIS2 Notification Message Deduplication (Alternative 1)

Where a Global Cache fails to process a notification message relating to a given unique data object (`properties.data_id` + `properties.pubtime`), a Global Cache should successfully process a valid, subsequently received notification message with the same unique data identifier.

**Pre-requisites:**

Prepared WIS2 Notification Messages:
- A known number of valid WNMs with:
  - `properties.data_id` + `properties.pubtime` are NOT unique to each message, but shared by 2 or more messages. This defines a unique identifier message set.
  - For each unique identifier message set, the first published message should be invalid, or the data object inaccessible, and the second message/data object should be valid.
- Accompanying data objects that are accessible (or not) via the canonical link provided in the WNM.

**Evaluate:**

1. WNM Messages
   - Only one message should be published by the GC on the `cache/a/wis2/#` topic per unique identifier, defined as `properties.data_id` + `properties.pubtime`.
2. Data Objects
   - Only one data object should be cached per unique identifier, defined as `properties.data_id` + `properties.pubtime`.
3. GC Metrics
   - `wmo_wis2_gc_download_total` (+=1 for each unique identifier)
   - `wmo_wis2_gc_last_download_timestamp_seconds` (set to current for each unique identifier)
   - `wmo_wis2_gc_downloaded_errors_total` (unchanged)
   - `wmo_wis2_gc_integrity_failed_total` (unchanged)

**WIS2-to-GTS Gateway variant:** Provide 10 pairs of WNMs where each pair shares the same `data_id` and `pubtime`. The first WNM of each pair should have a bad method and/or value for `properties.integrity`; the second should be valid. The Global Cache should throw an error for the first WNM but successfully process the second. Pairs of WNMs may include inline data.

---

### WIS2 Notification Message Deduplication (Alternative 2)

Related to the two previous tests, a GC should not process and cache a data item if it has already processed and cached a data item with the same `properties.data_id` and a `properties.pubtime` that is equal to or less than the `properties.pubtime` of the new data item (i.e., the GC should ignore out-of-sequence updates).

**Pre-requisites:**

Prepared WIS2 Notification Messages:
- A known number of valid WNMs with:
  - `properties.data_id` + `properties.pubtime` are NOT unique to each message, but shared by 2 or more messages. This defines a unique identifier message set.
  - For each unique identifier message set, the first published message should be valid, and the second message/data object should have a `properties.pubtime` earlier than the first message.
- Accompanying data objects that are accessible via the canonical link provided in the WNM.

**Evaluate:**

1. WNM Messages
   - For each message set with a shared `data_id`, each message should be processed by the GC and received on the `cache/a/wis2/#` topic only if the `properties.pubtime` has been correctly set for each message sent in chronological order — out-of-sequence messages are ignored.
2. Data Objects
   - Data objects should be cached per unique identifier when they are received in chronological order (as defined by `properties.pubtime`).
3. GC Metrics
   - `wmo_wis2_gc_download_total` (+=1 for each unique identifier)
   - `wmo_wis2_gc_last_download_timestamp_seconds` (set to current for each unique identifier)
   - `wmo_wis2_gc_downloaded_errors_total` (unchanged)
   - `wmo_wis2_gc_integrity_failed_total` (unchanged)

**WIS2-to-GTS Gateway variant:** Provide 10 pairs of WNMs where each pair shares the same `data_id`. The first WNM of each pair has `pubtime = {time}` and should be processed successfully. The second WNM of each pair has `pubtime = {time − x seconds}` (x could be 1 to 60 seconds or longer) and should be ignored. Pairs of WNMs may include inline data.

---

### Data Update

A Global Cache should treat notification messages with the same data item identifier (`properties.data_id`), but different publication times (`properties.pubtime`) as unique data items. Data items with the same `properties.data_id` but a greater/later publication time AND an update link (`links['rel']='update'`) should be processed (see WNM Processing test). Data items with the same `properties.data_id` but earlier or identical publication times should be ignored (see Deduplication tests).

**Pre-requisites:**

Prepared WIS2 Notification Messages:
- A known number of valid WNMs with:
  - `properties.data_id` + `properties.pubtime` are NOT unique to each message, but shared by 2 or more messages. This defines a unique identifier message set.
  - For each unique identifier message set, the first published message should be valid, and the second message/data object should have a `properties.pubtime` later than the first message AND include an update link (rather than a canonical link).
- Accompanying data objects that are accessible via the canonical link provided in the WNM.

**Evaluate:**

1. WNM Messages
   - For each message set with a shared `data_id`, each message should be processed by the GC and received on the `cache/a/wis2/#` topic only if the `properties.pubtime` has been correctly set for each message sent in chronological order AND later messages use an update link — out-of-sequence messages and in-sequence messages without an update link are ignored.
2. Data Objects
   - Data objects should be cached per unique identifier when they are received in chronological order (as defined by `properties.pubtime`) with an update link.
3. GC Metrics
   - `wmo_wis2_gc_download_total` (+=1 for each identifier that is either new or arrives in sequential time-order with an update link)
   - `wmo_wis2_gc_last_download_timestamp_seconds` (set to current when a new or updated data object is cached)
   - `wmo_wis2_gc_downloaded_errors_total` (unchanged)
   - `wmo_wis2_gc_integrity_failed_total` (unchanged)

**WIS2-to-GTS Gateway variant (URL data):** Provide 5 pairs of WNMs with no inline data. Each pair shares the same `data_id`. The first WNM has `pubtime = {time}` and should be processed successfully. The second WNM has `pubtime = {time + x}` and a data URL link with `rel=update`, and should also be processed successfully. Note: pairs may be interleaved — i.e., send all 5 first messages, wait, then send all 5 second messages.

**WIS2-to-GTS Gateway variant (inline data):** Provide 5 pairs of WNMs with inline data. Each pair shares the same `data_id`. The first WNM has `pubtime = {time}` and should be processed successfully. The second WNM has `pubtime = {time + x}` and a data URL link with `rel=update` (the `links.href` is invalid so only inline data can be used), and should also be processed successfully. Note: pairs may be interleaved — i.e., send all 5 first messages, wait, then send all 5 second messages.

---

## Performance Tests

### WIS2 Notification Processing Rate

A Global Cache shall be able to successfully process, on average, 2000 unique WNMs per minute with an average message size of 85kb. This test represents the average message size of the current WNMs. The noted WNMs/minute rate can be used as a performance indicator for the GC being tested.

**Pre-requisites:**

Prepared WIS2 Notification Messages and associated data objects:
- A known number of valid WNMs with:
  - `properties.cache` set to `true`
  - `properties.data_id` + `properties.pubtime` should be unique to each message. To ensure consistency, `data_id` should be used to determine uniqueness.
- Accompanying data objects should be accessible via the canonical link provided in the WNM.
  - The canonical link should be accessible per the core requirements and the data object hash should match the hash provided in the WNM if integrity properties are provided.
  - Average message size should be 85kb.

**Evaluate:**

1. WNM Messages
   - The total number of cache notification messages published by the GC on the `cache/a/wis2/#` topic should match what was published (2000).
2. Data Objects
   - 2000 data objects should be cached.
3. GC Metrics
   - `wmo_wis2_gc_download_total` (incremented for each of the 2000 messages)
4. Time
   - The time taken to process the messages should not exceed 60 seconds (plus time taken to publish the WNMs) in order to pass the test.

---

### Concurrent Client Downloads

A Global Cache should support a minimum of 1000 simultaneous downloads.

**Pre-requisites:**

Prepared WIS2 Notification Messages and associated data objects:
- A single valid WNM with:
  - `properties.cache` set to `true`
- Accompanying data object should be accessible via the canonical link provided in the WNM.
  - A larger than average data object should be generated/used in order to ensure that the clients downloading the data object concurrently do not finish before the test is complete. A 200MB data object will be used.
- ApacheBench (ab) to manage the concurrent downloads.

---

## Additional Tests (not in original test-script) — Inline Data

### Inline Data Retrieved OK

The Global Cache should successfully process WNMs that contain inline data, extracting the data from the WNM and caching it.

**Pre-requisites:**
- Provide 10 WNMs with inline data.

**Evaluate:**
- Global Cache should successfully process the 10 WNMs, extracting the data from the WNM and caching it.

---

### Failure to Retrieve Inline Data, Success to Retrieve Data from URL

Where inline data cannot be retrieved, the Global Cache should fall back to retrieving data via the remote URL.

**Pre-requisites:**

Provide 10 WNMs with:
- Inline data, but a bad `size` attribute — this should force the Gateway to throw an error and then try to retrieve data via the remote URL.
- No `properties.integrity` in the WNM. So, `size` is the only option to verify the embedded data is not correct.
- A resolvable URL for the canonical data — this data should be processed.

**Evaluate:**
- Global Cache should trigger an error (for inline retrieval); data published (for URL retrieval).

---

### Failure to Retrieve Inline Data, Failure to Retrieve Data from URL

Where both inline data retrieval and remote URL retrieval fail, no data should be published.

**Pre-requisites:**

Provide 10 WNMs with:
- Inline data, but a bad `size` attribute — this should force the Gateway to throw an error and then try to retrieve data via the remote URL.
- A bad (unresolvable) URL for the canonical data — this should force the Gateway to throw an error; this data cannot be retrieved.

**Evaluate:**
- Global Cache should trigger an error (for inline retrieval and for URL retrieval); no data published.

