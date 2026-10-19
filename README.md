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

## Upgrading from MattsTechInfo/Meshnet (NordVPN 3.x)
The last upstream image (`ghcr.io/mattstechinfo/meshnet:latest`, NordVPN 3.18.4 on Ubuntu Jammy) was built in September 2024 and receives no more updates. Upgrading is recommended mainly for **security**: a Meshnet node is an entry point into your network, and the old image misses two years of fixes.

- **Security fixes in the NordVPN client:** 3.20.0 ("significant security improvements, updating strongly encouraged"), an OpenSSL vulnerability fix in 4.5.0, OpenVPN security and OpenSSL 3.0.17 in 4.1.0, hardened token handling in 4.6.0.
- **Patched base image:** Ubuntu Noble, rebuilt weekly, instead of an unpatched 2024 Jammy image.
- **Meshnet fixes:** Meshnet firewall rules are cleaned up properly when Meshnet is turned off (3.20.3), the local network permission can no longer be bypassed through LAN discovery (3.20.1), the file sharing library was updated (3.19.1), and unexpected logouts during token renewal are fixed (4.6.0).
- **Old clients get cut off:** NordVPN has previously required clients to update before they can connect. Meshnet itself was announced for shutdown in 4.1.0 and kept in 4.2.0, only current clients receive such changes.

Things that changed and may affect you:

- **nftables instead of iptables (5.0):** the host kernel needs `nf_tables` (`lsmod | grep nf_tables`). NordVPN uses its own `inet nordvpn` table and only switches to a default-deny policy while a VPN connection or Kill Switch is active, Meshnet alone does not block other traffic. With `network_mode: host`, never run `nordvpn connect` or enable Kill Switch in this container, that would apply to the whole host.
- **Logout revokes the token (4.x):** inside the container only use `nordvpn logout --persist-token`.
- **IPv6 support was removed (4.0)**, Meshnet addresses are IPv4 so this does not affect Meshnet.
- **Privacy consent prompt (4.0)** is answered automatically by this image (analytics off).

The full NordVPN changelog is in the [releases](https://github.com/Sternpaul/meshnet/releases).

### Keeping the same Meshnet device
The device identity lives in `/var/lib/nordvpn/data/settings.dat` and `install.dat` inside your old container (the old image had no volume for it). Copy them into a volume before switching images, otherwise the new container registers as a new device. As long as the old container still exists (stopped is fine):

```bash
docker stop meshnet
docker commit meshnet meshnet-backup:3.18.4
mkdir -p meshnet-migrate
docker cp meshnet:/var/lib/nordvpn/data/settings.dat meshnet-migrate/
docker cp meshnet:/var/lib/nordvpn/data/install.dat meshnet-migrate/
docker volume create meshnet-nordvpn-data
docker run --rm -v meshnet-nordvpn-data:/var/lib/nordvpn -v "$PWD/meshnet-migrate:/migrate:ro" \
  --entrypoint cp ghcr.io/sternpaul/meshnet:latest /migrate/settings.dat /migrate/install.dat /var/lib/nordvpn/data/
```

Then point your compose file at `ghcr.io/sternpaul/meshnet:latest`, mount the volume at `/var/lib/nordvpn`, keep the same `hostname`, and run `docker compose up -d`:

```yaml
services:
  meshnet:
    image: ghcr.io/sternpaul/meshnet:latest
    # ... your existing settings, same hostname ...
    volumes:
      - nordvpn-data:/var/lib/nordvpn

volumes:
  nordvpn-data:
    name: meshnet-nordvpn-data
    external: true
```

The log should show `Already logged in.` and no new device appears in your Meshnet. To roll back, switch the image to `meshnet-backup:3.18.4` and remove the volume. Delete `meshnet-migrate` afterwards, it contains your login.

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
The example `docker-compose.yml` lists the most important variables with explanations; the access token goes into the `.env` file next to it. A `configMap` is supplied for Kubernetes deployments.

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
An example [`docker-compose.yml`](docker-compose/docker-compose.yml) with explanations for every setting has been supplied to easily deploy the Meshnet node. Put your token in the `.env` file next to it, set `NORDVPN_NICKNAME` and peer permissions, and start it.

Two pieces of configuration keep the node's identity stable on your Meshnet:

- **Set a fixed `hostname` that differs from `NORDVPN_NICKNAME`**. NordVPN derives the device's hardware ID from the host's hardware IDs and the hostname, so the same host and hostname come back as the same device. The client rejects a nickname equal to its own hostname ("a domain with this name already exists").
- **Persist `/var/lib/nordvpn`** (the example uses a named volume). It keeps the login, WireGuard key and stored hardware ID, so updates don't log in again. It is required to keep a device migrated from the old 3.x image: NordVPN 5.x calculates a different hardware ID than 3.x, and only the stored one keeps the old device.

NordVPN and Meshnet functionality require permissions to create a tunnel interface within the container. The container will require both capabilities `NET_ADMIN` and `NET_RAW`.

Make sure you have the `.env` file with your token next to the `docker-compose.yml` and run `docker compose up -d` to start the container. The node/peer should show up in your Meshnet within a few seconds.

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
