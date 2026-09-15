# Lab 3: Connect and Monitor Indirect Water Pumps

## Introduction

Create two indirectly connected water-pump twins and publish their telemetry through Gateway 1. Water Pump 3 sends telemetry that already matches the WaterPump model. Water Pump 4 sends flat telemetry with pressure in PSI. Both payloads reach OCI through Gateway 1, but each target twin uses its own adapter to produce the same canonical WaterPump state.

Estimated Time: 65 minutes

### Objectives

In this lab, you will:

- Create indirect twins that share one gateway association.
- Reuse the default and Flat PSI WaterPump adapters.
- Publish two source payload shapes through one MQTTs connection.
- Verify normalized values and the gateway association.

### Prerequisites

- Complete [Lab 2: Create the Gateway and Routing Adapter](../create-gateway-and-routing/create-gateway-and-routing.md).
- Have the ElectricMotor and WaterPump models and the default and Flat PSI WaterPump adapters in the IoT domain.
- Retain `IOT_DOMAIN_OCID`, `GATEWAY_INSTANCE_ID`, `GATEWAY_EXTERNAL_KEY`, `GATEWAY_SECRET_VALUE`, and `IOT_DEVICE_HOST` from Lab 2.

If the WaterPump models or adapters are missing, create them with [Appendix A: Create Required Factory, Production Line, and WaterPump Assets](../appendix-water-pump-assets/appendix-water-pump-assets.md).

## Task 1: Locate the WaterPump assets

1. List active WaterPump adapters. Identify **Water Pump Default Adapter** and **Water Pump Flat PSI Adapter**, the same adapter definitions used in the Getting Started workshop. Set their existing OCIDs; do not create duplicate adapters with the same display names.

    ```bash
    oci iot digital-twin-adapter list \
      --iot-domain-id "$IOT_DOMAIN_OCID" \
      --lifecycle-state ACTIVE \
      --all --output table

    export WATER_PUMP_MODEL_ID='<water-pump-model-ocid>'
    export DEFAULT_WATER_PUMP_ADAPTER_ID='<default-water-pump-adapter-ocid>'
    export FLAT_PSI_WATER_PUMP_ADAPTER_ID='<flat-psi-water-pump-adapter-ocid>'
    ```

2. Set stable external keys. An external key identifies an indirect device to the gateway routing adapter. It is not an MQTT user name because these pumps do not authenticate to OCI.

    ```bash
    export PUMP_3_EXTERNAL_KEY='water-pump-3'
    export PUMP_4_EXTERNAL_KEY='water-pump-4'
    ```

## Task 2: Create the indirect pump twins

1. Create Water Pump 3 with the default adapter. It has no `--auth-id`; its `--gateways` array associates it with Gateway 1.

    ```bash
    export PUMP_3_INSTANCE_ID=$(oci iot digital-twin-instance create \
      --iot-domain-id "$IOT_DOMAIN_OCID" \
      --digital-twin-model-id "$WATER_PUMP_MODEL_ID" \
      --digital-twin-adapter-id "$DEFAULT_WATER_PUMP_ADAPTER_ID" \
      --connectivity-type INDIRECT \
      --gateways "[\"$GATEWAY_INSTANCE_ID\"]" \
      --external-key "$PUMP_3_EXTERNAL_KEY" \
      --display-name "Water Pump 3" \
      --wait-for-state ACTIVE \
      --query 'data.id' --raw-output)
    ```

2. Create Water Pump 4 with the Flat PSI adapter. It uses the same WaterPump model and the same gateway association as Water Pump 3.

    ```bash
    export PUMP_4_INSTANCE_ID=$(oci iot digital-twin-instance create \
      --iot-domain-id "$IOT_DOMAIN_OCID" \
      --digital-twin-model-id "$WATER_PUMP_MODEL_ID" \
      --digital-twin-adapter-id "$FLAT_PSI_WATER_PUMP_ADAPTER_ID" \
      --connectivity-type INDIRECT \
      --gateways "[\"$GATEWAY_INSTANCE_ID\"]" \
      --external-key "$PUMP_4_EXTERNAL_KEY" \
      --display-name "Water Pump 4" \
      --wait-for-state ACTIVE \
      --query 'data.id' --raw-output)
    ```

3. Confirm the connectivity and gateway association. The `auth-id` field is null for both indirect pumps.

    ```bash
    oci iot digital-twin-instance get \
      --digital-twin-instance-id "$PUMP_3_INSTANCE_ID" \
      --query 'data.{name:"display-name",type:"connectivity-type",auth:"auth-id",gateways:gateways,key:"external-key"}'

    oci iot digital-twin-instance get \
      --digital-twin-instance-id "$PUMP_4_INSTANCE_ID" \
      --query 'data.{name:"display-name",type:"connectivity-type",auth:"auth-id",gateways:gateways,key:"external-key"}'
    ```

4. **Optional:** List every indirectly connected device associated with Gateway 1. This read-only pipeline first lists only indirect twins in the IoT domain, then uses `jq` to filter each twin's `gateways` array for `$GATEWAY_INSTANCE_ID`. OCI IoT does not provide a server-side list filter for a particular gateway, so this command performs that last filter locally. This step is for informational purposes and is not required to continue with the lab.

    ```bash
    oci iot digital-twin-instance list \
      --iot-domain-id "$IOT_DOMAIN_OCID" \
      --connectivity-type INDIRECT \
      --all \
      --output json |
    jq --arg gateway "$GATEWAY_INSTANCE_ID" '
      .data.items[]
      | select((.gateways // []) | index($gateway) != null)
      | {
          id,
          name: ."display-name",
          externalKey: ."external-key",
          gateways
        }'
    ```

## Task 3: Connect Gateway 1 and publish telemetry

1. Configure MQTTX for Gateway 1. Use `mqtts://$IOT_DEVICE_HOST`, port `8883`, TLS, a clean session, `$GATEWAY_EXTERNAL_KEY` as the user name, and `$GATEWAY_SECRET_VALUE` as the password.

2. Publish gateway status telemetry to the `data` topic. The empty target keeps this message with Gateway 1.

    ```bash
    mqttx pub \
      -h "$IOT_DEVICE_HOST" -p 8883 -l mqtts \
      -t data \
      -m '{"time":"2026-09-02T18:00:00.000000Z","connectedDeviceCount":2,"cpuUtil":30,"memUtil":25,"firmware":"Oracle Linux 9.1"}' \
      -u "$GATEWAY_EXTERNAL_KEY" -P "$GATEWAY_SECRET_VALUE"
    ```

3. Publish model-shaped telemetry for Water Pump 3. The `water-pumps/water-pump-3` path resolves the target external key and delegates the payload to Pump 3's default adapter.

    ```bash
    mqttx pub \
      -h "$IOT_DEVICE_HOST" -p 8883 -l mqtts \
      -t "water-pumps/$PUMP_3_EXTERNAL_KEY" \
      -m '{"time":"2026-09-02T18:01:00.000000Z","motor":{"motorTemperature":68.4,"vibrationLevel":1.7,"powerConsumption":12.6},"flowRate":247.5,"dischargePressure":4.3}' \
      -u "$GATEWAY_EXTERNAL_KEY" -P "$GATEWAY_SECRET_VALUE"
    ```

4. Publish flat PSI telemetry for Water Pump 4. The gateway resolves Pump 4 from the path and supplies `timeObserved`; the target adapter inherits that timestamp. The target adapter receives the original endpoint, but its wildcard route matches it. It maps the flat pump fields, converts PSI to bar, and builds the nested motor component. No gateway-specific Flat PSI adapter is required.

    ```bash
    mqttx pub \
      -h "$IOT_DEVICE_HOST" -p 8883 -l mqtts \
      -t "water-pumps/$PUMP_4_EXTERNAL_KEY" \
      -m '{"time":"2026-09-02T18:02:00.000000Z","motorTemperature":68.4,"vibrationLevel":1.7,"powerConsumption":12.6,"flowRate":247.5,"dischPressPsi":62.37}' \
      -u "$GATEWAY_EXTERNAL_KEY" -P "$GATEWAY_SECRET_VALUE"
    ```

## Task 4: Verify normalized state and gateway association

1. Retrieve latest content and metadata for Gateway 1 and both pumps.

    ```bash
    oci iot digital-twin-instance get-content \
      --digital-twin-instance-id "$GATEWAY_INSTANCE_ID" \
      --should-include-metadata true

    oci iot digital-twin-instance get-content \
      --digital-twin-instance-id "$PUMP_3_INSTANCE_ID" \
      --should-include-metadata true

    oci iot digital-twin-instance get-content \
      --digital-twin-instance-id "$PUMP_4_INSTANCE_ID" \
      --should-include-metadata true
    ```

2. Confirm both pumps expose the canonical WaterPump paths. For Water Pump 4, `62.37` PSI is approximately `4.30` bar. Confirm recent `timeLastHeard` metadata for the gateway and both pumps.

## Learn More

- [Scenario: Create digital twins for indirectly connected devices using a gateway](https://docs.oracle.com/en-us/iaas/Content/internet-of-things/gateway-instance.htm)
- [Route indirect device data using target and contentRoot](https://docs.oracle.com/en-us/iaas/Content/internet-of-things/gateway-target-content-root.htm)
- [IoT domain database schema reference](https://docs.oracle.com/en-us/iaas/Content/internet-of-things/iot-domain-database-schema.htm)

## Acknowledgements

* **Author** - Pete St. Pierre, Director, Product Management
* **Last Updated By/Date** - Pete St. Pierre, September 2026
