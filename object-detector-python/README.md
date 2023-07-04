*Copyright (C) 2021, Axis Communications AB, Lund, Sweden. All Rights Reserved.*

# An inference example application on an edge device

This example is written in Python and implements and tests the following object detection scenarios from the models trained using tensorflow ssd models:

* Run a video streaming inference on camera from our trained model
* Run a still image inference on camera from our trained model

This is an update for [object-detector-python example](https://github.com/AxisCommunications/acap-computer-vision-sdk-examples/tree/main/object-detector-python) with full description about how to run. Please refer to the original git address for more information.

## Example structure

Following are the list of files and a brief description of each file in the example

```text
object-detector-python
├── app
│   ├── detector.py
│   └── sample.jpg
├── docker-compose.yml
├── static-image.yml
├── Dockerfile
├── Dockerfile.model
└── README.md
```

* **detector.py** - The inference client main program
* **sample.jpg** - Static image used with static-image.yml
* **docker-compose.yml** - Docker compose file for streaming camera video example using larod inference service
* **static-image.yml** - Docker compose file for static image debug example using larod inference service
* **Dockerfile** - Build Docker image with inference client for camera
* **Dockerfile.model** - Build Docker image with inference model

## Requirements

Meet the following requirements to ensure compatibility with the example:

* Axis device
  * Chip: ARTPEC-{7-8} DLPU devices (e.g., Q1615 MkIII)
  * Firmware: 11.0 or higher
  * [Docker ACAP](https://github.com/AxisCommunications/docker-acap) installed and started, using TLS and SD card as storage: [available at here] (#ACAP-TLS-SD)
* Computer
  * Either [Docker Desktop](https://docs.docker.com/desktop/) version 4.11.1 or higher, or [Docker Engine](https://docs.docker.com/engine/) version 20.10.17 or higher with BuildKit enabled using Docker Compose version 1.29.2 or higher

## ACAP TLS SD
### INSTALL Docker ACAP
Check Docker ACAP compatibility:

```bash
DEVICE_IP=<device ip>
DEVICE_PASSWORD='<password>'

curl -s --anyauth -u "root:$DEVICE_PASSWORD" "http://$DEVICE_IP/axis-cgi/param.cgi?action=update&root.Network.SSH.Enabled=yes"

ssh root@$DEVICE_IP 'command -v containerd >/dev/null 2>&1 && echo Compatible with Docker ACAP || echo Not compatible with Docker ACAP'
```

If that's ok, install it
```bash
ARCH=<armv7hf or aarch64>
CHIP=<tpu or artpec8>
docker run --rm axisecp/docker-acap:latest-$ARCH $DEVICE_IP $DEVICE_PASSWORD install
```

Generate TLS and copy to Device dockerd
```bash
rm *.pem *.csr *.srl *.cnf
openssl genrsa -aes256 -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
openssl genrsa -out server-key.pem 4096
openssl req -subj "/CN=$DEVICE_IP" -sha256 -new -key server-key.pem -out server.csr
echo subjectAltName = IP:$DEVICE_IP,IP:127.0.0.1 >> extfile.cnf
echo extendedKeyUsage = serverAuth >> extfile.cnf
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile extfile.cnf
openssl genrsa -out key.pem 4096
openssl req -subj '/CN=client' -new -key key.pem -out client.csr
echo extendedKeyUsage = clientAuth > extfile-client.cnf
openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile-client.cnf
scp ca.pem server-cert.pem server-key.pem root@$DEVICE_IP:/usr/local/packages/dockerdwrapper/
```

Check the date of certificates:
```bash
openssl x509 -in ca.pem -text
```
Then Update the datetime of Camera Device according to it.

Enable TLS and SD card support:
```bash
DOCKER_PORT=2376
curl -s --anyauth -u "root:$DEVICE_PASSWORD" "http://$DEVICE_IP/axis-cgi/param.cgi?action=update&root.dockerdwrapper.UseTLS=yes"
docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem -H=$DEVICE_IP:$DOCKER_PORT version
export CHIP=<device chip>
export ARCH=<device arch>
export APP_NAME=acap4-object-detector-python
export MODEL_NAME=acap-dl-models

docker pull axisecp/acap-runtime:1.1.2-$ARCH-containerized

docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem  --host tcp://$DEVICE_IP:$DOCKER_PORT system prune --all --force

curl -s --anyauth -u "root:$DEVICE_PASSWORD"   "http://$DEVICE_IP/axis-cgi/param.cgi?action=update&root.dockerdwrapper.SDCardSupport=yes"
```

## Load docker Images to camera
Restart the APP, then load docker images
```bash
docker save $APP_NAME | docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem --host tcp://$DEVICE_IP:$DOCKER_PORT load
docker save $MODEL_NAME | docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem --host tcp://$DEVICE_IP:$DOCKER_PORT load
docker save axisecp/acap-runtime:1.1.2-$ARCH-containerized | docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem --host tcp://$DEVICE_IP:$DOCKER_PORT load
```

## Copy files to camera
copy app and models dir to camera using:
```bash
scp -r app $DEVICE_IP:/root/
scp -r models $DEVICE_IP:/root/
```

## Run dockers using docker-compose file
Run one of the docker-compose files

1- Inference on Video Stream
```bash
docker-compose --tlsverify --tlscacert ca.pem --tlscert cert.pem --tlskey key.pem --host tcp://$DEVICE_IP:$DOCKER_PORT --file docker-compose.yml --env-file ./config/env.$ARCH.$CHIP up
```

2- Inference Static Image
```bash
docker-compose --tlsverify --tlscacert ca.pem --tlscert cert.pem --tlskey key.pem --host tcp://$DEVICE_IP:$DOCKER_PORT --file static-image.yml --env-file ./config/env.$ARCH.$CHIP up
```


## License

**A more clear version from the combination of [ACAP](https://github.com/AxisCommunications/docker-acap) and object-detector-python example from [this](https://github.com/AxisCommunications/acap-computer-vision-sdk-examples) Repo**

**[Apache License 2.0](../LICENSE)**
