# Global Replay (GRep) Service Tutorials

## Try Out the Canadian Instance of the GRep Service

Centre-id: `ca-eccc-msc-global-replay`

### Useful Links

- [API documentation (Swagger UI) — "Try it out"](https://wis2-grep.weather.gc.ca/openapi?f=html#/wis2-grep-subscriber/describeWis2-grep-subscriberProcess)
- [What can you query — "Queryables"](https://wis2-grep.weather.gc.ca/collections/wis2-notification-messages/queryables)

---

### HTTP Fetch (Synchronous Request)

1. [API endpoint — Collection: `wis2-notification-messages`](https://wis2-grep.weather.gc.ca/collections/wis2-notification-messages)
2. [API endpoint — Browse all notification messages](https://wis2-grep.weather.gc.ca/collections/wis2-notification-messages/items)
3. [API endpoint — Browse notification messages on specific topic (and sub-topics)](https://wis2-grep.weather.gc.ca/collections/wis2-notification-messages/items?topic=origin/a/wis2/us-noaa-nws)
4. [More queries](https://github.com/wmo-im/wis2-grep?tab=readme-ov-file#api-queries)

---

### MQTT Fetch (Asynchronous Request)

1. [API endpoint — Process: `wis2-grep-subscriber`](https://wis2-grep.weather.gc.ca/processes/wis2-grep-subscriber)

2. Use MQTT Explorer to connect to the GRep broker (GRep still in test mode — not yet connected to Global Brokers). Connection details:

   ```
   GLOBAL-BROKER=mqtts://globalbroker.meteo.fr:8883
   USERNAME=everyone
   PASSWORD=everyone
   ```

3. Generate a UUID for your subscriber (you'll use this to set the topic on which you receive the replayed messages you request):

   ```bash
   ~ % uuidgen
   34F7849F-2D3D-4E9F-B3D1-69487E7DA7DE
   ```

   > *** USE YOUR OWN UUID TO AVOID CONFLICT WITH OTHER CHAMPIONS! ***

4. Set up a subscription for your replayed messages — using your unique UUID. Topics will be formed as `replay/a/wis2/<centre-id>/<subscriber-id>/<topic>`. For everything published for your unique subscriber ID:

   ```
   replay/a/wis2/ca-eccc-msc-global-replay/34F7849F-2D3D-4E9F-B3D1-69487E7DA7DE/#
   ```

5. Use the [Swagger UI](https://wis2-grep.weather.gc.ca/openapi?f=html#/processes/wis2-grep-subscriber/execution) or curl to set up an MQTT Fetch.

   **Example:** Request 20 mins of all messages from `ca-eccc-msc` WIS2 Node (origin) from midday (UTC) 19-Mar-2026 
   
   > *** AADJUST DATE-TIMES AS NEEDED (only 72 hours of messages are cached) ***

   `[POST] https://wis2-grep.weather.gc.ca/processes/wis2-grep-subscriber/execution`

   Request payload:
   ```json
   {
     "inputs": {
       "datetime": "2026-03-19T12:00:00Z/2026-03-19T12:20:00Z",
       "subscriber-id": "34F7849F-2D3D-4E9F-B3D1-69487E7DA7DE",
       "topic": "origin/a/wis2/ca-eccc-msc"
     }
   }
   ```

   Using curl:
   ```bash
   curl -X 'POST' \
     'https://wis2-grep.weather.gc.ca/processes/wis2-grep-subscriber/execution' \
     -H 'accept: application/json' \
     -H 'Content-Type: application/json' \
     -d '{
     "inputs": {
       "datetime": "2026-03-19T12:00:00Z/2026-03-19T12:20:00Z",
       "subscriber-id": "34F7849F-2D3D-4E9F-B3D1-69487E7DA7DE",
       "topic": "origin/a/wis2/ca-eccc-msc"
     }
   }'
   ```

   Response (success) — plus 2456 messages published on the wis2-grep broker:
   ```json
   {"status":"successful","subscriptions":[{"rel":"items","type":"application/geo+json","href":"mqtts://everyone:everyone@globalbroker.inmet.gov.br:8883","title":"Instituto Nacional de Meteorologia (Brazil), Global Broker Service","channel":"replay/a/wis2/ca-eccc-msc-global-replay/34F7849F-2D3D-4E9F-B3D1-69487E7DA7DE/#"},{"rel":"items","type":"application/geo+json","href":"mqtts://everyone:everyone@wis2globalbroker.nws.noaa.gov:8883","title":"National Oceanic and Atmospheric Administration, National Weather Service, Global Broker Service","channel":"replay/a/wis2/ca-eccc-msc-global-replay/34F7849F-2D3D-4E9F-B3D1-69487E7DA7DE/#"},{"rel":"items","type":"application/geo+json","href":"mqtts://everyone:everyone@gb.wis.cma.cn:8883","title":"China Meteorological Administration, Global Broker Service","channel":"replay/a/wis2/ca-eccc-msc-global-replay/34F7849F-2D3D-4E9F-B3D1-69487E7DA7DE/#"},{"rel":"items","type":"application/geo+json","href":"mqtts://everyone:everyone@globalbroker.meteo.fr:8883","title":"M\u00e9t\u00e9o-France, Global Broker Service","channel":"replay/a/wis2/ca-eccc-msc-global-replay/34F7849F-2D3D-4E9F-B3D1-69487E7DA7DE/#"}]}
   ```

6. Repeat this query as an HTTP Fetch (synchronous request) to check the same number of messages are returned:

   ```bash
   curl "https://wis2-grep.weather.gc.ca/collections/wis2-notification-messages/items?datetime=2026-03-19T12:00:00Z/2026-03-19T12:20:00Z&topic=origin/a/wis2/ca-eccc-msc"
   ```

   Response (success), returning 2456 records.

---

## [ADVANCED] Try Out the GRep Test Client

Exploring further: using the [MQTT-SUBSCRIBER-WEB](https://github.com/wmo-im/wis2-champions/blob/main/Bali-workshop-MAR2026/GREP_TEST_CLIENT.md) application to test the GRep service.

> Note: there are still some issues with the GRep implementation — can you see what they are?

> Note: GRep server may be configured to discard messages from GTS-to-WIS2 Gateways
