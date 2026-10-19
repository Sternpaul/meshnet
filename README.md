# Meshnet
[![GitHub](https://img.shields.io/github/license/Sternpaul/meshnet?style=for-the-badge)](https://github.com/Sternpaul/meshnet/blob/master/LICENSE)
![GitHub Workflow Status (with event)](https://img.shields.io/github/actions/workflow/status/Sternpaul/meshnet/docker-image.yml?style=for-the-badge)

This (Docker) container provides the official NordVPN client configured for Meshnet VPN usage. Easily deploy fully configurable Meshnet nodes that automatically join your Meshnet network. This project is not a supported/official container image by NordVPN and is in no way endorsed by the company NordVPN.

This is a maintained continuation of [MattsTechInfo/Meshnet](https://github.com/MattsTechInfo/Meshnet), which stopped receiving updates. The image rebuilds itself:

- **Daily check**: CI checks NordVPN's apt repository every day and publishes a new image as soon as NordVPN ships a new Linux client.
- **Weekly rebuild**: the image is rebuilt every week regardless, to pick up base image security patches.
- Images are tagged with the NordVPN client version they contain (e.g. `5.4.1`, `5.4`, `5`) plus `latest`, for `linux/amd64` and `linux/arm64`.
- Every new NordVPN client version gets a [GitHub release](https://github.com/Sternpaul/meshnet/releases) with NordVPN's changelog, so you can watch the repository for releases to get notified.

```
docker pull ghcr.io/sternpaul/meshnet:latest
```

## General Meshnet information
Meshnet is a free self hosted VPN network connecting multiple nodes together. Its functionality, provided with the NordVPN application, is available on most platforms, including Android/Google TV. The usecases are extensive, the most popular ones are to access local data/network through Meshnet while having remote clients or setting up an exit node to route all client traffic through a single location.
Read more about Meshnet over here: https://meshnet.nordvpn.com/

## Installation and configuration
Deploying this container is quite easy as it does not require specific ports to function as a gateway node, initial traffic is outgoing. Other uses may require different configurations.

### Preparations
A (free) NordVPN account is required to enable Meshnet and use the services.
This container requires an `Access token` to log on the NordVPN client. Follow the following steps to generate a new `Access token`:
- Login to https://my.nordaccount.com/
- Go to Services → NordVPN
- Scroll down to "Access token"
- Press the "Generate new token" button
- Choose "Doesn't expire" unless you want to rotate the token yourself

### nftables requirement
Since NordVPN 5.0 the client manages its firewall with nftables instead of iptables, with a default-deny policy. The host kernel must support nftables (`nf_tables` module), which all current distributions do. Some older NAS kernels (e.g. certain Synology models) do not; Meshnet will fail to come up there.

### Environment variables
A `.env` file is supplied with the `docker-compose.yml` file for configuration purposes, this file already contains quite some commentary. A `configMap` is supplied for Kubernetes deployments.

#### General config
- `NORDVPN_TOKEN` - Supply your `Access token` to be able to login. If you want to use a file or secret instead, please leave this ENV blank or comment it out.
- `NORDVPN_TOKENFILE` - Load the `Access token` from a file mounted in the container. Make sure nothing else but the token is inside. Please leave this blank if you are using `NORDVPN_TOKEN` or comment it out.
- `NORDVPN_MESHNET_DEBUG` - Enable debug mode, anything non-empty will ENABLE. Use this if you need more verbose error logging for troubleshooting.
- `NORDVPN_HEALTHCHECK_INTERVAL` - Set the interval to verify connectivity to the set URL, defaults to 300 (seconds).
- `NORDVPN_HEALTHCHECK_URL` - An address to verify if connectivity is available. Choose something depending on what connectivity you want to verify, defaults to www.google.com. Please keep in mind, if the healthcheck fails the container will be killed.
- `NORDVPN_NICKNAME`- Set a nickname for this device/instance on Meshnet, every peer will see and can use this nickname.

#### Meshnet Permissions
Permissions are configured directly on the client. NordVPN currently ALLOWS all peers connected to Meshnet by default for Fileshare and Remote access services and DENIES Routing and Local network services.

This container will run DENY configuration first, followed by ALLOW. ALLOW will overwrite the DENY! Entering a peer in both DENY and ALLOW will first DENY the peer and then overwrite it with an ALLOW.

Peers must be entered with their FQDN/Name assigned by Meshnet, comma separated, example: `peer-atlas.nord,peer-fuji.nord`

- `NORDVPN_DENY_PEER_ROUTING` - Block peers from using this node as a router.
- `NORDVPN_DENY_PEER_LOCAL` - Block peers from accessing the local network of this node.
- `NORDVPN_DENY_PEER_FILESHARE` - Block peers from sharing files with this node.
- `NORDVPN_DENY_PEER_REMOTE` - Block peers from remote access to this node.


- `NORDVPN_ALLOW_PEER_ROUTING` - Allow peers to use this node as a router.
- `NORDVPN_ALLOW_PEER_LOCAL` - Allow peers to access the local network of this node (ROUTING permissions required!).
- `NORDVPN_ALLOW_PEER_FILESHARE` - Allow peers to  sharing files with this node.
- `NORDVPN_ALLOW_PEER_REMOTE` = Allow peers to use remote access on this node.

### Deployment - docker-compose
An example `docker-compose.yml` has been supplied to easily deploy the Meshnet node. Two pieces of configuration keep the node's identity stable on your Meshnet:

- **Persist `/var/lib/nordvpn`** (the example uses a named volume). The NordVPN client stores its login and device identity there; without it, every container recreation registers a new peer on your Meshnet.
- **Set a `hostname` that differs from `NORDVPN_NICKNAME`**. The client rejects a nickname equal to the machine's own hostname ("a domain with this name already exists").

NordVPN and Meshnet functionality require permissions to create a tunnel interface within the container. The container will require both capabilities `NET_ADMIN` and `NET_RAW`.

Make sure you have the `.env` file next to the `docker-compose.yml` and run `docker compose up -d` to start the container. The node/peer should show up in your Meshnet within a few seconds.

### Deployment - Kubernetes
Kubernetes examples are supplied to easily deploy the Meshnet node on a Kubernetes cluster. There are two files, `meshnet-deployment.yaml` and `meshnet-env.yaml`. In the `meshnet-deployment.yaml`, please make sure the `hostname` is not empty. Without configuring a `hostname`, every restart of the container will show as a new node within the Meshnet. To also survive pod recreation, mount a persistent volume at `/var/lib/nordvpn`.

NordVPN and Meshnet functionality require permissions to create a tunnel interface within the container. The container will require both capabilities `NET_ADMIN` and `NET_RAW`.

Make sure you have the `meshnet-env.yaml` file configured and run `kubectl apply -f <meshnet_folder> -n <namespace>` to start the container. The node/peer should show up in your Meshnet within a few seconds.

## ARM64
Next to the default AMD64 platform this container is also built for ARM64. This will allow for easy deployment on Ampere based K8s nodes or VM's in, for example, the Free-Tier Oracle Cloud Infrastructure. If Meshnet fails to start on a node, make sure the `nf_tables` kernel module is loaded (`sudo modprobe nf_tables`).

## Updating
Every published image is tagged `latest`, so a container using `ghcr.io/sternpaul/meshnet:latest` picks up new NordVPN clients with `docker compose pull && docker compose up -d`, or automatically with a tool such as [Watchtower](https://containrrr.dev/watchtower/). Pin a version tag (e.g. `5.4.1`) instead if you prefer to update by hand.

## Synology
A few users have had some problems with certain Synology devices as these do not come with the required netfilter (nftables) kernel modules.

## Credits
Originally created by [Matts Bos / MattsTechInfo](https://github.com/MattsTechInfo/Meshnet). The NordVPN 5.x fixes (repository URL, script permissions, analytics prompt, persisted device identity) follow those found by [nmbradley/nordvpn-meshnet](https://github.com/nmbradley/nordvpn-meshnet).

Starting this image has been based on the excellent work of https://github.com/bubuntux/nordvpn with their NordVPN client implementation in Docker.
