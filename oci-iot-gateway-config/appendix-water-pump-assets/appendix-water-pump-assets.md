# Appendix A: Create Required WaterPump Models and Adapters

## Introduction

Use this appendix only when the IoT domain does not already contain the ElectricMotor and WaterPump models and the two WaterPump adapters used in Lab 3. These assets are intentionally separated from the gateway-specific work in Lab 2. The same WaterPump model supports model-shaped telemetry and flat telemetry after adapter normalization.

Estimated Time: 45 minutes

### Objectives

In this appendix, you will:

- Create the ElectricMotor and WaterPump models.
- Create the default WaterPump adapter.
- Create the Flat PSI WaterPump adapter.
- Record and verify the resulting OCIDs.

### Prerequisites

- An active IoT domain and OCI CLI profile.
- A writable working directory.
- Permission to create digital twin models and adapters in the IoT domain.

## Task 1: Create the ElectricMotor and WaterPump models

1. Set your working directory and IoT domain OCID. The following commands save the model specifications as JSON files in the working directory.

    ```bash
    export WORKSHOP_DIR="$PWD"
    export IOT_DOMAIN_OCID='<iot-domain-ocid>'
    ```

2. Save the ElectricMotor model specification as `$WORKSHOP_DIR/electric-motor-model.json`.

    ```bash
    cat > "$WORKSHOP_DIR/electric-motor-model.json" <<'EOF'
    {
      "@context": [
        "dtmi:dtdl:context;3",
        "dtmi:dtdl:extension:historization;1"
      ],
      "@id": "dtmi:com:oracle:iot:example:ElectricMotor;1",
      "@type": "Interface",
      "displayName": "Electric Motor",
      "contents": [
        {
          "@type": ["Telemetry", "Historized"],
          "name": "motorTemperature",
          "schema": "double"
        },
        {
          "@type": ["Telemetry", "Historized"],
          "name": "vibrationLevel",
          "schema": "double"
        },
        {
          "@type": "Telemetry",
          "name": "powerConsumption",
          "schema": "double"
        }
      ]
    }
    EOF
    ```

3. Save the WaterPump model specification as `$WORKSHOP_DIR/water-pump-model.json`. Its `motor` component references the ElectricMotor DTMI created in the preceding file.

    ```bash
    cat > "$WORKSHOP_DIR/water-pump-model.json" <<'EOF'
    {
      "@context": [
        "dtmi:dtdl:context;3",
        "dtmi:dtdl:extension:quantitativeTypes;1",
        "dtmi:com:oracle:dtdl:extension:validation;1"
      ],
      "@id": "dtmi:com:oracle:iot:example:WaterPump;1",
      "@type": "Interface",
      "displayName": "Water Pump",
      "contents": [
        {
          "@type": "Component",
          "name": "motor",
          "schema": "dtmi:com:oracle:iot:example:ElectricMotor;1"
        },
        {
          "@type": ["Telemetry", "Validated"],
          "name": "flowRate",
          "schema": "double",
          "minimum": 0,
          "maximum": 1000
        },
        {
          "@type": ["Telemetry", "Pressure"],
          "name": "dischargePressure",
          "schema": "double",
          "unit": "bar"
        }
      ]
    }
    EOF
    ```

4. Create the ElectricMotor model before the WaterPump model because WaterPump references its DTMI as a component.

    ```bash
    export ELECTRIC_MOTOR_MODEL_ID=$(oci iot digital-twin-model create \
      --iot-domain-id "$IOT_DOMAIN_OCID" \
      --display-name "Electric Motor Model" \
      --spec "file://$WORKSHOP_DIR/electric-motor-model.json" \
      --wait-for-state ACTIVE \
      --query 'data.id' --raw-output)

    export WATER_PUMP_MODEL_ID=$(oci iot digital-twin-model create \
      --iot-domain-id "$IOT_DOMAIN_OCID" \
      --display-name "Water Pump Model" \
      --spec "file://$WORKSHOP_DIR/water-pump-model.json" \
      --wait-for-state ACTIVE \
      --query 'data.id' --raw-output)
    ```

5. Verify both stored specifications.

    ```bash
    oci iot digital-twin-model get-spec --digital-twin-model-id "$ELECTRIC_MOTOR_MODEL_ID"
    oci iot digital-twin-model get-spec --digital-twin-model-id "$WATER_PUMP_MODEL_ID"
    ```

## Task 2: Create the default WaterPump adapter

1. Create the default adapter. It requires no JSON files because the source payload already matches the WaterPump model shape and uses bar for pressure.

    ```bash
    export DEFAULT_WATER_PUMP_ADAPTER_ID=$(oci iot digital-twin-adapter create \
      --iot-domain-id "$IOT_DOMAIN_OCID" \
      --digital-twin-model-id "$WATER_PUMP_MODEL_ID" \
      --display-name "Water Pump Default Adapter" \
      --description "Accepts model-shaped water-pump telemetry." \
      --wait-for-state ACTIVE \
      --query 'data.id' --raw-output)
    ```

## Task 3: Create the Flat PSI WaterPump adapter

1. Save the Flat PSI adapter inbound envelope as `$WORKSHOP_DIR/flat-psi-water-pump-envelope.json`.

    ```bash
    cat > "$WORKSHOP_DIR/flat-psi-water-pump-envelope.json" <<'EOF'
    {
      "referenceEndpoint": "/telemetry",
      "referencePayload": {
        "dataFormat": "JSON",
        "data": {
          "time": "2026-09-02T18:00:00.000000Z",
          "motorTemperature": 68.4,
          "vibrationLevel": 1.7,
          "powerConsumption": 12.6,
          "flowRate": 247.5,
          "dischPressPsi": 62.37
        }
      },
      "envelopeMapping": {
        "timeObserved": "$.time"
      }
    }
    EOF
    ```

2. Save the Flat PSI adapter routes as `$WORKSHOP_DIR/flat-psi-water-pump-routes.json`. The route builds the nested motor component and converts PSI to bar.

    ```bash
    cat > "$WORKSHOP_DIR/flat-psi-water-pump-routes.json" <<'EOF'
    [
      {
        "condition": "*",
        "description": "Build the motor component and convert PSI pressure to bar.",
        "payloadMapping": {
          "$.motor.motorTemperature": "$.motorTemperature",
          "$.motor.vibrationLevel": "$.vibrationLevel",
          "$.motor.powerConsumption": "$.powerConsumption",
          "$.flowRate": "$.flowRate",
          "$.dischargePressure": "${(.dischPressPsi * 0.0689475729)}"
        },
        "referencePayload": {
          "dataFormat": "JSON",
          "data": {
            "time": "2026-09-02T18:00:00.000000Z",
            "motorTemperature": 68.4,
            "vibrationLevel": 1.7,
            "powerConsumption": 12.6,
            "flowRate": 247.5,
            "dischPressPsi": 62.37
          }
        }
      }
    ]
    EOF
    ```

3. Create the Flat PSI adapter.

    ```bash
    export FLAT_PSI_WATER_PUMP_ADAPTER_ID=$(oci iot digital-twin-adapter create \
      --iot-domain-id "$IOT_DOMAIN_OCID" \
      --digital-twin-model-id "$WATER_PUMP_MODEL_ID" \
      --display-name "Water Pump Flat PSI Adapter" \
      --description "Maps flat water-pump telemetry and converts PSI pressure to bar." \
      --inbound-envelope "file://$WORKSHOP_DIR/flat-psi-water-pump-envelope.json" \
      --inbound-routes "file://$WORKSHOP_DIR/flat-psi-water-pump-routes.json" \
      --wait-for-state ACTIVE \
      --query 'data.id' --raw-output)
    ```

## Task 4: Verify the WaterPump assets

1. List active adapters for the WaterPump model. Retain all three exported IDs for Lab 3.

    ```bash
    oci iot digital-twin-adapter list \
      --iot-domain-id "$IOT_DOMAIN_OCID" \
      --digital-twin-model-id "$WATER_PUMP_MODEL_ID" \
      --lifecycle-state ACTIVE \
      --all --output table
    ```

## Documentation References

- [Creating digital twin models](https://docs.oracle.com/en-us/iaas/Content/internet-of-things/create-digital-twin-model.htm)
- [Creating digital twin adapters](https://docs.oracle.com/en-us/iaas/Content/internet-of-things/create-digital-twin-adapter.htm)
- [Digital twin model overview](https://docs.oracle.com/en-us/iaas/Content/internet-of-things/digital-twin-models.htm)
- [DTDL v3 specification](https://github.com/Azure/opendigitaltwins-dtdl/blob/master/DTDL/v3/DTDL.v3.md)

## Acknowledgements

* **Author** - Pete St. Pierre, Director, Product Management
* **Last Updated By/Date** - Pete St. Pierre, September 2026
