# This document describes the necessary steps required to control Meross Smart Plugs and Surge Protectors from a local MQTT server

The following sources were instrumental in setting up local control:
- https://github.com/bytespider/Meross and https://github.com/bytespider/Meross/wiki/MQTT
- https://github.com/albertogeniola/MerossIot and https://github.com/albertogeniola/MerossIot/issues/1

## Configure a local MQTT server

Your local MQTT server needs to be accessible via a secure connection, which means setting up a certificate for it.

Using the instructions [here](https://github.com/albertogeniola/MerossIot/issues/1#issuecomment-648112885) create the certificates required for TLS

```
mkdir -p certs/{ca,broker}
cd certs/

# CA
openssl genrsa \
  -out ca/ca.key \
  2048
openssl req \
  -new \
  -x509 \
  -days 1826 \
  -key ca/ca.key \
  -out ca/ca.crt \
  -subj "/CN=MQTT CA"

# broker
openssl genrsa \
  -out broker/broker.key \
  2048
openssl req \
  -new \
  -out broker/broker.csr \
  -key broker/broker.key \
  -subj "/CN=broker"
openssl x509 \
  -req \
  -in broker/broker.csr \
  -CA ca/ca.crt \
  -CAkey ca/ca.key \
  -CAcreateserial \
  -out broker/broker.crt \
  -days 360
rm broker/broker.csr
```

Next, modify your `mosquitto.conf` file to register the certificates and enable the secure port. I've got both secure and non-secure as my home automation software uses the non-secure route.

```
port 1883
listener 8883

require_certificate false
use_identity_as_username false
allow_anonymous true

cafile /mosquitto/certs/ca/ca.crt
certfile /mosquitto/certs/broker/broker.crt
keyfile /mosquitto/certs/broker/broker.key
```

## First-time setup of Meross devices

Grab the code from here: https://github.com/bytespider/Meross and make sure you can run it.

For each device in turn:
1. Press and hold the power button for ~5s until it goes into factory reset mode (flashing orange light)
2. Connect to the Wifi-AP that the device is broadcasting
3. Run `node meross setup --gateway 10.10.10.1 --wifi-ssid {ssid} --wifi-pass {password} --mqtt {mosquitto_ip}:8883`
4. Make a note of the response as it contains the UUID that will be used when sending mqtt messages

## Testing that the devices are talking to your MQTT server properly

Download MQTT explorer or similar and connect it to your MQTT server.

Press the power button on your Meross device, you should see an MQTT topic being broadcast

The topics to subscribe to are `/appliance/<device uuid>/publish` and `/appliance/<device uuid>/subscribe`

## MQTT Messages

Unfortunately all messages are sent to the same topics, just with different headers and payloads.