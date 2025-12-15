# ChirpStack Docker example

This repository contains a skeleton to setup the [ChirpStack](https://www.chirpstack.io)
open-source LoRaWAN Network Server (v4) using [Docker Compose](https://docs.docker.com/compose/).

**Note:** Please use this `docker-compose.yml` file as a starting point for testing
but keep in mind that for production usage it might need modifications. 

## Directory layout

* `docker-compose.yml`: the docker-compose file containing the services
* `configuration/chirpstack`: directory containing the ChirpStack configuration files
* `configuration/chirpstack-gateway-bridge`: directory containing the ChirpStack Gateway Bridge configuration
* `configuration/mosquitto`: directory containing the Mosquitto (MQTT broker) configuration
* `configuration/postgresql/initdb/`: directory containing PostgreSQL initialization scripts

## Configuration

This setup is pre-configured for all regions. You can either connect a ChirpStack Gateway Bridge
instance (v3.14.0+) to the MQTT broker (port 1883) or connect a Semtech UDP Packet Forwarder.
Please note that:

* You must prefix the MQTT topic with the region.
  Please see the region configuration files in the `configuration/chirpstack` for a list
  of topic prefixes (e.g. eu868, us915_0, au915_0, as923_2, ...).
* The protobuf marshaler is configured.

This setup also comes with two instances of the ChirpStack Gateway Bridge. One
is configured to handle the Semtech UDP Packet Forwarder data (port 1700), the
other is configured to handle the Basics Station protocol (port 3001). Both
instances are by default configured for EU868 (using the `eu868` MQTT topic
prefix).

### Reconfigure regions

ChirpStack has at least one configuration of each region enabled. You will find
the list of `enabled_regions` in `configuration/chirpstack/chirpstack.toml`.
Each entry in `enabled_regions` refers to the `id` that can be found in the
`region_XXX.toml` file. This `region_XXX.toml` also contains a `topic_prefix`
configuration which you need to configure the ChirpStack Gateway Bridge
UDP instance (see below).

#### ChirpStack Gateway Bridge (UDP)

Within the `docker-compose.yml` file, you must replace the `eu868` prefix in the
`INTEGRATION__..._TOPIC_TEMPLATE` configuration with the MQTT `topic_prefix` of
the region you would like to use (e.g. `us915_0`, `au915_0`, `in865`, ...).

#### ChirpStack Gateway Bridge (Basics Station)

Within the `docker-compose.yml` file, you must update the configuration file
that the ChirpStack Gateway Bridge instance must used. The default is
`chirpstack-gateway-bridge-basicstation-eu868.toml`. For available
configuration files, please see the `configuration/chirpstack-gateway-bridge`
directory.

# Data persistence

PostgreSQL and Redis data is persisted in Docker volumes, see the `docker-compose.yml`
`volumes` definition.

## Requirements

Before using this `docker-compose.yml` file, make sure you have [Docker](https://www.docker.com/community-edition)
installed.

## Importing device repository

To import the [lorawan-devices](https://github.com/TheThingsNetwork/lorawan-devices)
repository (optional step), run the following command:

```bash
make import-lorawan-devices
```

This will clone the `lorawan-devices` repository and execute the import command of ChirpStack.
Please note that for this step you need to have the `make` command installed.

**Note:** an older snapshot of the `lorawan-devices` repository is cloned as the
latest revision no longer contains a `LICENSE` file.

## Usage

To start the ChirpStack simply run:

```bash
$ docker-compose up
```

After all the components have been initialized and started, you should be able
to open http://localhost:8080/ in your browser.

##

The example includes the [ChirpStack REST API](https://github.com/chirpstack/chirpstack-rest-api).
You should be able to access the UI by opening http://localhost:8090 in your browser.

**Note:** It is recommended to use the [gRPC](https://www.chirpstack.io/docs/chirpstack/api/grpc.html)
interface over the [REST](https://www.chirpstack.io/docs/chirpstack/api/rest.html) interface.

## TLS Setup for Basic Station

This branch includes TLS server authentication for secure Basic Station connections.

### Prerequisites

Generate certificates using [chirpstack-certificates](https://github.com/chirpstack/chirpstack-certificates):

```bash
git clone https://github.com/chirpstack/chirpstack-certificates.git
cd chirpstack-certificates

# Set your hostname(s) - comma-separated for multiple
export CHIRPSTACK_GATEWAY_BRIDGE_HOSTS="yourdomain.com,localhost"

# Generate certificates
make
```

### Installation Steps

1. **Create the certs directory:**
   ```bash
   mkdir -p config/certs
   ```

2. **Copy certificates from chirpstack-certificates:**
   ```bash
   cp /path/to/chirpstack-certificates/certs/ca/ca.pem config/certs/
   cp /path/to/chirpstack-certificates/certs/chirpstack-gateway-bridge/basicstation/basicstation.pem config/certs/
   cp /path/to/chirpstack-certificates/certs/chirpstack-gateway-bridge/basicstation/basicstation-key.pem config/certs/
   ```

3. **Set file permissions:**
   ```bash
   chmod 644 config/certs/*.pem
   ```

4. **Start the services:**
   ```bash
   docker compose up -d
   ```

5. **Verify TLS is working:**
   ```bash
   openssl s_client -connect localhost:13008 -CAfile config/certs/ca.pem </dev/null
   ```

### Gateway Configuration

Configure your LoRaWAN gateway to connect via TLS:

| Setting | Value |
|---------|-------|
| Protocol | `wss://` (WebSocket Secure) |
| Server URI | `wss://yourdomain.com:13008` |
| CA Certificate | Upload contents of `config/certs/ca.pem` |

### TLS Modes

This setup uses **Server TLS Only** (gateway verifies server certificate).

To enable **Mutual TLS** (both server and gateway verify each other), edit `config/gateway-bridge/chirpstack-gateway-bridge.toml`:

```toml
[backend.basic_station]
tls_cert="/etc/chirpstack-gateway-bridge/certs/basicstation.pem"
tls_key="/etc/chirpstack-gateway-bridge/certs/basicstation-key.pem"
ca_cert="/etc/chirpstack-gateway-bridge/certs/ca.pem"  # Enables client cert verification
```

With mTLS, each gateway needs its own certificate signed by the CA.

### Security Notes

- The `config/certs/` directory is excluded from git via `.gitignore`
- Never commit private keys to version control
- Certificates are valid for 1 year by default (regenerate before expiry)
- For production, use certificates from a trusted CA or ensure gateways have your CA certificate
