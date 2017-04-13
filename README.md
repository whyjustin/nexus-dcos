# Running Nexus Repository Manager in DC/OS on AWS

This is a tutorial explaining how to install and configure Sonatype [Nexus Repository Manager](https://www.sonatype.com/nexus-repository-oss) in Mesosphere [DC/OS](https://dcos.io/). It will detail how to configure Nexus Repository Manager as a private Docker registry and host a developer's Docker image on this registry. Finally, it will explain how to deploy this Docker image to DC/OS from the Docker registry on Nexus Repository Manager.

## Installing DC/OS on AWS

DC/OS can we installed via the [Cloud Formation Template](https://dcos.io/docs/1.9/administration/installing/cloud/aws/).

## Configuring DC/OS CLI tool

Find the Mesos Master hostname from the AWS CloudFormation page under the Outputs tab. More details available in the [Launch DC/OS section](https://dcos.io/docs/1.9/administration/installing/cloud/aws/basic/#-a-name-launchdcos-a-launch-dc-os) of the DC/OS documentation. Using this URL, configure and login to the DC/OS CLI tool, downloadable from your [DC/OS web interface](https://dcos.io/docs/1.9/usage/cli/install/).

```bash
./dcos config set core.dcos_url $masterUrl
./dcos auth login
```

## Install Marathon-lb

Marathon-lb may be installed from the DC/OS Universe either through the web UI or the command line. Below we will install via the CLI.

```bash
./dcos package install marathon-lb
```

## Install Nexus Repository Manager

Nexus Repository Manager may be installed from the DC/OS Universe either through the web UI or the command line. It should be configured for persistent storage using an external volume. On AWS this will create an Elastic Block Store volume named 'nexus-data'. Follow the screencast to configure via the DC/OS UI, or create a json file named options.json as follows

```json
{
  "service": {
    "name": "nexus",
    "cpus": 2,
    "mem": 2048
  },
  "networking": {
    "host-port": 0
  },
  "storage": {
    "persist": true,
    "local-volume": {
      "host-volume": "/opt/sonatype/nexus-data"
    },
    "external-volume": {
      "enable": true,
      "volume-name": "nexus-data"
    }
  }
}
```

and install Nexus Repository Manager with these options via the CLI.

```bash
./dcos package install --options=options.json nexus
```

## Configure Nexus Repository Manager

### Nexus Repository Manager template

After installing marathon-lb, HAProxy should be exposed via the AWS public slave. The public IP of this slave can be queried using the DC/OS CLI.

```bash
for id in $(./dcos node --json | jq --raw-output '.[] | select(.attributes.public_ip == "true") | .id'); do ./dcos node ssh --mesos-id=$id --master-proxy "curl -s ifconfig.co"; done
```

Use this IP to lookup the A record for the public slave from the AWS console. We will use this A record as the name for our Docker registry and will be referenced as $publicAName going forward in this tutorial. Generally these A records follow the format of `ec2-${publicIP.replace('.', '-')}-${awsRegion}.compute.amazonaws.com`.

Using the DC/OS web UI, adjust the Nexus template by adding the following port to the Docker portMappings

```json
{
  "containerPort": 10001,
  "hostPort": 0,
  "servicePort": 10001,"
  "protocol": "tcp",
  "name": "docker"
}
```

Adjust the service port for the default port mappings to 10000 to ensure that it will be exposed by marathon-lb. Finally, add the following labels to the Nexus template

```
"HAPROXY_GROUP": "external",
"HAPROXY_0_VHOST": "$publicAName"
```

The screencast details how to add these to the Nexus Repository Manager template.

### Nexus Repository Manager Docker registry

Once Nexus is running, navigate to the Nexus instance at http://$publicAName:10000. Login as an admin and create a new Docker (hosted) repository and configure the http port to 10001. More details about installing a Docker registry on Nexus Repository Manager are available in the [Private Registry for Docker](https://books.sonatype.com/nexus-book/3.0/reference/docker.html) chapter of the Nexus Documentation.

## Configure DC/OS Agents

Sonatype highly suggests using SSL for all Docker registries. We commonly see Nexus configured with SSL endpoint termination on a reverse proxy with communication between the reverse proxy and Nexus using HTTP. marathon-lb with HAProxy should be configured for SSL endpoint termination although outside the scope of this tutorial. We suggest that the insecure-registry configuration detailed below only be used within private systems. |
 :---- |

We will deploy a tarball with authentication information on every agent to allow the Docker daemons to log into our private registry. Another option would be to utilize shared storage available to all DC/OS agents preventing the need to maintain this file on multiple machines. First use the DC/OS CLI to retrieve the mesos-id of all the agents on the DC/OS cluster.

```bash
./dcos node --json | jq --raw-output '.[] | .id'
```

We will tunnel into each of these machines and configure their Docker daemons. For each ID returned above, run use the following command to tunnel into the box.

```bash
./dcos node ssh --mesos-id=$id --master-proxy
```

On each agent we will create a tarball with the default Nexus Repository Manager authentication. We suggest creating a system account to support the Docker registry access rather than admin. The auth field contains a Base64 encoded version of `username:password`.

```bash
echo {\"auths\": {\"$publicAName:10001\": {\"auth\":\"YWRtaW46YWRtaW4xMjM=\"}}} | tee .docker/config.json
tar cf docker.tar.gz .docker/
sudo mv docker.tar.gz /etc/
```

If we have not configured SSL, on each agent we will configure the Docker daemon to support our registry as insecure.

```bash
echo {\"insecure-registries\":[\"$publicAName:10001\"]} | sudo tee /etc/docker/daemon.json
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## Push a Docker image

Now that Nexus Repository Manager is configured, we can build and push a Docker image. First login to the new Docker registry, then tag an existing image or a new image with $publicAName:10001 and finally push this image. More details on [Docker deployments](https://docs.docker.com/registry/deploying/) are available in the Docker documentation.

```bash
docker login -u admin -p admin123 $publicAName:10001
docker build --tag=$publicAName:10001/sonatype-say .
docker push $publicAName:10001/sonatype-say
docker rmi $publicAName:10001/sonatype-say
```

## Create a DC/OS Service with Docker image

Finally we can create a new service with the Docker image we just pushed. Follow the screencast to configure via the DC/OS UI, or create a json file named sonatype-say.json as follows

```json
{
  "id": "/sonatype-say",
  "instances": 1,
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "$publicAName:10001/sonatype-say"
    },
  },
  "cpus": 0.1,
  "mem": 128,
  "uris": [
    "file:///etc/docker.tar.gz"
  ]
}
```

and install it on DC/OS

```bash
./dcos marathon app add sonatype-say.json
```

You can verify the service is now running using the following command.

```bash
./dcos marathon app list
```
