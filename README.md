# Kube-Home: Advanced Kubernetes Home Server Setup

## Overview
This guide documents the set up of a single-node Kubernetes cluster on Ubuntu 24.04 LTS Server using MicroK8s or K3s. It includes configurations for persistent storage, networking and the deployment of essential services like Pi-hole, Jellyfin and Homepage. Services are exposed using both LoadBalancer, and Gateway API through Kong Gateway for ingress management.

This setup serves as a flexible starting point for building a Kubernetes-based home server. While based on personal preferences, it can be easily customized to suit individual needs.

Here’s what the homepage looks like once services are up and running:

![Homepage Screenshot](./images/homepagetest.png)

## Table of Contents

1. [Overview](#overview)  
2. [Tech Stack](#tech-stack)  
3. [Base System Installation](#base-system-installation)  
4. [Kubernetes Setup (K3s or MicroK8s)](#kubernetes-setup-k3s-or-microk8s) 
5. [MetalLB Setup](#metallb-load-balancer-setup)  
6. [Service Deployments](#service-deployments)  
7. [Kong Gateway Setup with Gateway API](#kong-gateway-setup-with-gateway-api)
8. [Manage Certificates with cert-manager and Let's Encrypt](#manage-certificates-with-cert-manager-and-lets-encrypt)  
9. [License](#license)

## Tech Stack

This section provides an overview of the core components used in the setup:

| Component                                          | Description                                       |
| ------------------------------------------------- | ------------------------------------------------- |
| **[Ubuntu 24.04 LTS](https://ubuntu.com/server)**       | Host operating system                             |
| **[K3s](https://k3s.io/) / [MicroK8s](https://microk8s.io/)** | Lightweight Kubernetes distributions              |
| **Local Path Provisioner** | HostPath-based CSI driver for local persistent storage |
| **[MetalLB](https://metallb.io/)**          | LoadBalancer implementation for bare-metal setups |
| **[Pi-hole](https://pi-hole.net/)**          | DNS-level ad blocking and local DNS resolver      |
| **[Jellyfin](https://jellyfin.org/)**         | Media streaming server                            |
| **[Omada Software Controller](https://www.omadanetworks.com/en/business-networking/omada/controller/)** | TP-Link SDN network controller                    |
| **[Homepage](https://gethomepage.dev/)**         | Homepage for linking various services    |
| **[Kong Ingress Controller (KIC)](https://konghq.com/products/kong-ingress-controller)** | Manages Gateway API for Kong Gateway |
| **[cert-manager](https://cert-manager.io/)** | Automates X.509 certificate management and ACME DNS-01 challenges |


## Base System Installation
### Hardware Requirements
This home server setup can run on both bare-metal and virtualized environments. A minimum of 2 CPU cores, 8GB of RAM and 20-30GB of disk space is recommended for smooth operation.

For reference, I use a Beelink Mini S12 Pro with an Intel N100 CPU (4 cores) and 16 GB RAM. It idles at 4–5% CPU and 23% memory usage with all current services running. Resource needs vary by service load — scale your hardware as needed.

### OS Installation

- Install **Ubuntu 24.04 LTS Server**
- Set hostname: `k8s-test-1`
- Enable `ssh`
- Create user: `ubuntu`

>Most Kubernetes manifests in this project assume the hostname `k8s-test-1`, particularly for PersistentVolume declarations. If you choose a different hostname, be sure to update the relevant manifests accordingly.

### Persistent Volume Configuration

Kubernetes manifests in this setup use `/data/k8s-local` as the centralized location for persistent container data.

>💡 **Optional**: You can create a dedicated LVM volume for `/data` to separate persistent data from the root filesystem.
```bash
# Display existing volume groups
sudo vgdisplay -v

# Create a new logical volume (replace <size> with the desired size in GB)
sudo lvcreate -L <size>G -n data-lv /dev/ubuntu-vg

# Format it with XFS
sudo mkfs.xfs /dev/ubuntu-vg/data-lv

# Check UUID for fstab entry
lsblk -f /dev/ubuntu-vg/data-lv
```
Edit `/etc/fstab` to mount on boot
```bash
UUID=<UUID_STRING>  /data  xfs  defaults,noatime,nodiratime  0  2
```
Then, mount the volume 
```bash
sudo mkdir /data
sudo mount /data
```
➡️ Create the directory for Kubernetes persistent data. This is required regardless of volume creation.
```bash
sudo mkdir -p -m 750 /data/k8s-local
```

## Kubernetes Setup (K3s or MicroK8s)

Install either **K3s** or **MicroK8s**. Both are lightweight Kubernetes distributions well-suited for a homelab use case. They are easy to install and require minimal maintenance.

### Install K3s

Install a single-node K3s cluster:
```bash
# Install K3s without Traefik and built-in service load balancer
sudo su
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik --disable=servicelb" sh -
```
Verify that the cluster is running

```bash
kubectl get nodes -o wide
systemctl status k3s
```

### Install MicroK8
Install single node MicroK8s cluster
```bash
sudo snap install microk8s --classic  
```
Verify microk8s is running
```bash
  sudo microk8s status --wait-ready 
  sudo microk8s kubectl get nodes -o wide
```
Enable essential MicroK8s add-ons:
```bash   
  sudo microk8s enable rbac
  sudo microk8s enable dns
  sudo microk8s enable hostpath-storage
  sudo microk8s enable helm3
```
### Remote Access with Kubectl
If you prefer to access the Kubernetes cluster from a remote server using kubectl, export the kubeconfig file and copy it to your remote host.
```bash
# MicroK8s
sudo microk8s config > kubeconfig-microk8s.yaml

# K3s
sudo cat /etc/rancher/k3s/k3s.yaml > kubeconfig-k3s.yaml
```
Copy the file to your remote host and edit it. In the copied YAML file, update the server: address to the real IP address of your Kubernetes node:
```bash
clusters:
- cluster:
    server: https://<NODE-IP>:6443
```

Check the current kubectl version on the node
```bash
# MicroK8s
microk8s kubectl version

# K3s
k3s kubectl version
```

Install matching kubectl version to your remote host. Replace v1.x.y with the version from above. 

```bash
K8S_VERSION=v1.x.y     
curl -LO https://dl.k8s.io/release/${K8S_VERSION}/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
💡 Version compatibility:
kubectl is officially supported if its minor version is within ±1 of the Kubernetes API server version. For example, if your cluster is running Kubernetes v1.32.4, you can safely use kubectl version v1.31, v1.32, or v1.33. Patch versions (e.g., v1.32.*) are fully compatible and can be updated freely.

Access the cluster using the edited kubectl config file
```bash
export KUBECONFIG=~/kubeconfig-<k3s|microk8s>.yaml
kubectl get nodes -o wide
```

For detailed installation and configuration instructions, refer to the official Kubernetes documentation: [Install `kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux)


## MetalLB Setup
MetalLB enables LoadBalancer services in bare-metal clusters by assigning external IPs from a local address pool. This is critical for homelabs where cloud load balancers are unavailable. 

⚠️ Install MetalLB after Kubernetes setup and before deploying services that require external access. Choose an IP range that does not overlap with your DHCP server.

### MetalLB for K3s

Install MetalLB with Helm
```bash
helm install metallb metallb/metallb --namespace metallb-system --create-namespace
```
Edit template file `metallb-addresspool.yaml` to define your range(s). Then apply the configuration.
```bash
cd ./metallb
kubectl apply -f metallb-L2Advertisement.yaml -f metallb-addresspool.yaml
```

### MetalLB for MicroK8s

  Decide the IP range MetalLB should use to automatically allocate IPs. Be sure to use an IP range that does not overlap with your local network DHCP range.

  Enable  MetalLB add-on. You will be prompted to enter the IP address range. Choose one outside your DHCP scope.
  ```bash  
  sudo microk8s enable metallb
  ```

To update the IP range later:
```bash
# Export current config
kubectl get -n metallb-system ipaddresspool default-addresspool -o yaml  > default-addresspool.yaml
# Edit the file and then re-apply
kubectl apply -f  default-addresspool.yaml  
```

## Service Deployments

Initially, each service is exposed using a `LoadBalancer` service to simplify connectivity and testing. Once verified, a centralized Gateway API configuration is introduced using Kong as the gateway controller.

> ⚠️ Ensure MetalLB is installed and configured before deploying any services. See the [MetalLB Load Balancer Setup](#metallb-load-balancer-setup) for details.

### Pi-hole DNS Server

Pi-hole provides DNS resolution and ad-blocking for the network.

Create the directory for persistent data:
```bash
sudo mkdir -p -m 750 /data/k8s-local/pihole/etc
```
Choose one of the following options to deploy Pi-hole:

Option 1 – Manual deployment using base manifests. 
Edit the templates under pihole/base, then deploy:
```bash
cd ./pihole/base
kubectl apply -f pihole-ns.yaml
kubectl apply -f pihole-pv.yaml
kubectl apply -f pihole-pvc.yaml
kubectl apply -f pihole-cm.yaml
kubectl apply -f pihole-secret.yaml
kubectl apply -f pihole-deployment.yaml
kubectl apply -f pihole-svc.yaml
```
Option 2 – Deploy using Kustomize overlay. Review and modify patch.yaml, configs.txt, and secrets.txt in the overlay directory. Then deploy:
```bash
cd ./pihole/overlays/template

# Preview final manifest
kubectl kustomize .

# Check syntax
kubectl kustomize . | kubectl apply -f - --dry-run=client

# Apply
kubectl kustomize . | kubectl apply -f -
```
Verify deployment and access the web UI:

```bash
kubectl get svc -n pihole -o wide  # Note the LoadBalancer EXTERNAL-IP
```
Visit:
http://\<external-ip\>/admin/login

### Jellyfin Media Server
Create local directories for persistent data volumes:
```bash
sudo mkdir -p -m 750   /data/k8s-local/jellyfin/config
sudo mkdir -p -m 750   /data/k8s-local/jellyfin/cache
sudo mkdir /data/media  # Directory for media files
```

Edit jellyfin manifest templates and then deploy
```bash
cd ./jellyfin
kubectl kustomize | kubectl apply --dry-run=client -f -  # Check syntax
kubectl kustomize | kubectl apply -f -
```
Verify deployment
```bash
kubectl get svc -n jellyfin  # Check LoadBalancer EXTERNAL-IP
http://<external-ip>  # Check access with browser
```

### Omada Software Controller
Omada Software controller is relevant only if you have TP-Link Omada network devices.

Create local directories for persistent data volumes
```bash
sudo mkdir -p -m 750 /data/k8s-local/omada/data
sudo mkdir -p -m 750 /data/k8s-local/omada/logs
```
Edit templates and deploy
```bash
cd ./omada
kubectl kustomize .   # Check final yaml output  before applying it
kubectl kustomize . | kubectl apply -f - 
```
Verify
```bash
kubectl get svc -n omada  # Check LoadBalancer EXTERNAL-IP
http://<external-ip>      # Check access with browser
```
### Homepage
A simple central homepage for accessing all deployed services.

Edit templates and deploy
```bash
cd ./homepage  
kubectl kustomize . | kubectl apply -f - --dry-run=client
kubectl kustomize . | kubectl apply -f -
```

Verify service has internal ClusterIP and deployment is OK
```bash
kubectl get deployment,svc -n homepage 
```
## Kong Gateway Setup with Gateway API
Kong Gateway is configured using Gateway API resources via the Kong Ingress Controller (KIC), which acts as the control plane.
### Choose Your DNS domain
This setup uses host-based routing (e.g. jellyfin.home.example.com) via Kong Gateway. For routing to work, a functioning local DNS (like Pi-hole) must resolve these domain names to the Kong proxy's external IP. Alternatively, you can define static entries in your system’s hosts file (e.g., /etc/hosts on Linux or macOS).

>The key requirement is that each fully qualified domain name resolves to the Kong LoadBalancer IP.

If you don't have a public domain, you can use a custom local-only domain for internal DNS resolution. Suitable examples include:

- `example.home`
- `example.lan` – commonly used in local networks (e.g., Pi-hole default)
- `example.internal`
- `example.com` – used throughout this project's manifests; [reserved for documentation and testing](https://datatracker.ietf.org/doc/html/rfc2606), safe to use in local environments

> **Note:** Ensure your DNS server (e.g., Pi-hole, dnsmasq, or local resolver) is properly configured to resolve these custom domains to the correct internal IPs.

Avoid .local, which is reserved for mDNS and may cause conflicts.

All provided HTTPRoute templates and the Kong gateway manifest in this repo usehome.example.com as a placeholder. Replace it with your actual domain and make sure your DNS server maps the relevant records to the Kong proxy IP as explained later in chapter [Configure Pi-hole DNS server](#configure-pi-hole-dns-server)

### Install Kong
Install Gateway API CRDS (standard channel)
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```
Install Kong Ingress Controller
```bash
helm repo add kong https://charts.konghq.com
helm repo update
helm install kong kong/ingress -n kong --create-namespace 
```
First edit templates and then create gatewayclass and gateway resources
```bash
cd ./gateway/kong
kubectl apply -f gateway-ns.yaml -f kong-gc.yaml -f  kong-gtw.yaml 
```

Verify connectivity to Kong via the external proxy address:
```bash
kubectl get svc -n kong kong-gateway-proxy  # Check LoadBalancer EXTERNAL-IP of the proxy
curl -i <external-ip>  # Check connectivity to Kong proxy, use the LB EXTERNAL-IP
```
If Kong is working correctly, the `curl` output should show below HTTP 404 error as no routes have been configured yet. By default, Kong returns a 404 error when it cannot match a request to any defined route. Once you configure HTTP routes, Kong will forward requests to the appropriate backend services based on the routing rules.

```
HTTP/1.1 404 Not Found
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 48
X-Kong-Response-Latency: 0
Server: kong/3.0.0
 
{"message":"no Route matched with those values"}
```


### Configure Pi-hole DNS Server
Login to the Pi-hole admin console and add DNS records that resolve the Kong proxy external IP to your service hostnames.

Pi-hole UI path: *Setting -> Local DNS records*

| Domain                    | IP Address                 | 
| -----------------------   | ------------------------   | 
| proxy.home.example.com    | \<kong-proxy-external-ip\> | 
| jellyfin.home.example.com | \<kong-proxy-external-ip\> |
| omada.home.example.com    | \<kong-proxy-external-ip\> |
| pihole.home.example.com   | \<kong-proxy-external-ip\> |

Ensure Pi-hole domain is set correctly (e.g., home.example.com). You can set this via:
- Environment variable:  `FTLCONF_dns_domain`
- Admin UI: *Settings -> All Settings -> dns.domain*

(Pi-hole default is *lan*)
Finally, configure your client system to use the Pi-hole Kubernetes service LoadBalancer IP as its DNS server (not the Kong proxy IP).

Verify DNS resolution:
```bash
curl http://proxy.home.example.com
```

### Attach HTTP routes to Kong Gateway
HTTP routes define how incoming requests are forwarded to backend services based on hostnames, paths, or other criteria. In this section, HTTP routes are configured to enable access to the services through the Kong Gateway.

Review httproute templates and attach them to Kong gateway to get access to the services via the proxy IP address. Most route templates are based on hostnames, with some also including path-prefix routing. Before applying routes, ensure the `hostnames` field in each HTTPRoute template matches your domain name as configured in the DNS. Here is snippet from `homepage-route.yaml` template:

```yaml
hostnames:
  - proxy.home.example.com
```

Review and edit templates, then apply:
```bash
cd ./gateway/kong
kubectl apply -f homepage-route.yaml  # Hostnames + PathPrefix route
kubectl apply -f jellyfin-route.yaml  # Hostnames + PathPrefix route
kubectl apply -f pihole-route.yaml    # Hostnames route
kubectl apply -f omada-route.yaml     # Hostnames route
```
Verify the state of gateway and routes
```bash
# Check state of gateway and all routes
kubectl get gc,gtw,httproute -A

# Describe a specific HTTP route for detailed information
kubectl describe -n <namespace> httproute <httproute_name>
```
If gateway, HTTP routes and DNS are working properly, you should be able to access the following URLs in a browser:

- http://proxy.home.example.com :  Access the Homepage service
- http://\<gateway-proxy-ip-address\> : Access the Homepage service
- http://jellyfin.home.example.com : Access the Jellyfin service
- http://\<gateway-proxy-ip-address\>/jellyfin/  # Access the Jellyfin service (keep trailing /)
- and so on...

## Manage Certificates with cert-manager and Let's Encrypt
This setup uses cert-manager to automatically provision and renew HTTPS certificates for services exposed through Kong Gateway. It requests a wildcard TLS certificate from Let's Encrypt using the DNS-01 ACME challenge, stores the resulting certificate in a Kubernetes secret, and uses it to terminate HTTPS traffic in Kong Gateway.

cert-manager is a Kubernetes controller that automates the management and renewal of TLS certificates.

### Prerequisites ###
This setup uses the ACME DNS-01 challenge, which requires control over your public domain’s DNS records. It works by creating temporary DNS TXT records to prove domain ownership to Let’s Encrypt.

You must have:
- A public domain name (e.g., yourdomain.net)
- DNS API access or valid credentials to allow cert-manager to create TXT records automatically under your domain

❗️Note: Placeholder domains like home.example.com cannot be used to issue real certificates. You must use a real, publicly resolvable domain for DNS-01 validation and certificate issuance.


### Install cert-manager

On microk8s enable cert-manager add-on with this command:
```bash
sudo microk8s enable cert-manager
```

On K3s you can deploy cert-manager with helm:
```bash
On K3s deploy cert-manager with helm:
helm repo add jetstack https://charts.jetstack.io --force-update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.2 \
  --set crds.enabled=true
```  

### Setup cert-manager and Request Certificate

Provided templates and instructions describe the steps for configuring cert-manager with Cloudflare DNS as an example. For other DNS hosting providers, please check cert-manager documentation. 

❗️Note: You need to replace the placeholder domain `example.com` with your own public domain in template files and command examples below.

1. Assuming you already have a public DNS domain. For the testing in local private network, choose/create some subdomain e.g. `home.` in your public DNS hosting records. The A record of public domain does not need to point to any live IP address when using DNS-01. However, you may set it to a placeholder IP for consistency (`0.0.0.0`). 

2. Create a Cloudflare API token for ACME DNS-01 challenge and place the key in template file `cloudflare-api-token-secret.yaml`. In the Cloudflare dashboard, create an API token with the following permissions.

Permissions:
  - Zone - Zone - Read
  - Zone - DNS - Edit
  Zone resources:
  - Include - Specific zone - <YOUR_DOMAIN> 

Then, update the template file `cloudflare-api-token-secret.yaml` with the created API token:
```bash
cd cert-manager
nano cloudflare-api-token-secret.yaml
```

3. Create certificate cluster issuer using Let's Encrypt staging environment. Review template file `letsencrypt-staging-clusterissuer.yaml` and set a valid email.

4. Create certificate definition using provided template file `proxy-certificate.yaml`. Edit the file and ensure that:
  - certificate `secretName:` name matches the secret `name:` in Kong gateway template file `gateway/kong/kong-gtw.yaml`. Otherwise it does not matter what name you choose to use.
  - certificate `dnsNames:` matches your chosen public dns domains.
  - certificate  `organizations:` Optionally set organization: field for informational purposes.

5. Create the cert-manager resources using the prepared manifest templates:
```bash
# Apply secret containing Cloudflare API token
kubectl apply -f cloudflare-api-token-secret.yaml

# Apply a ClusterIssuer using Let's Encrypt staging
kubectl apply -f letsencrypt-staging-clusterissuer.yaml

# Apply the certificate definition to request a wildcard cert
kubectl apply -f proxy-certificate.yaml
```


6. You can monitor the issuance of the certificate by commands:
  ```bash
  kubectl get cert,crs -n gateway
  kubectl describe cert -n gateway
  kubectl describe crs -n gateway
  ```
  
7. Once certificate is ready, Kong should now present it when querying the proxy address with https.
```bash
# You should now see a Let's Encrypt certificate instead of the default Kong self signed certificate
```bash
curl -ivko /dev/null https://proxy.home.example.com
```
8. Next, check the http routing still works. Try accessing the homepage using browser.
```bash
https://proxy.home.example.com :  Access the Homepage service with browser.
```
The homepage should open, but again with certificate warning. Check the presented certificate details. You should see the Let's Encrypt `Staging` certificate created for your own public domain.

This concludes the steps of testing the automatic creation and management of Let's Encrypt certificates using cert-manager. If you would like to get real trusted production certificates from Let's Encrypt and update all the remaining configs to match your domain, please proceed with changes explained in next chapter.

### Request Let's Encrypt Production Certificate and Finalize Setup 
If you initially used a placeholder domain like `home.example.com` (as in the template defaults), it must be replaced with your actual public domain. To issue a valid production certificate from Let's Encrypt and update all relevant configurations, follow these steps:

1. Create the cert-manager Clusterissuer for Let's-Encrypt production certificates. Edit the file `letsencrypt-prod-clusterissuer.yaml` and include a valid email address. Then apply the configuration.
```bash
kubectl apply -f letsencrypt-prod-clusterissuer.yaml
```
2. Replace the staging certificate with a production certificate. Edit file `proxy-certificate.yaml` and change the `issuerRef.name` value from `letsencrypt-staging` to `letsencrypt-prod`. Apply the configuration.
```bash
kubectl apply -f proxy-certificate.yaml
```
3. Update Kong Gateway and local DNS configuration. Revisit the chapter *Kong Gateway Setup with Gateway API config*. Update all references to `home.example.com` to your actual public domain. Re-apply all affected manifests. Also ensure your local DNS records (e.g. in Pi-hole) are updated to match your public domain setup. 

4. Update the Homepage deployment configuration:
- In `homepage/homepage-deployment.yaml`, update the `HOMEPAGE_ALLOWED_HOSTS` environment variable to include your public domain
- Also in `homepage-configmap.yaml, update domain names in `services.yaml: |` section
Then, re-apply the config:
```bash
kubectl apply -f ./homepage-configmap.yaml -f homepage-deployment.yaml

# Tip: After applying the updated ConfigMap, the Homepage app must be restarted to reflect the changes:
kubectl rollout restart deployment homepage -n homepage 
```

### Verify Setup
  ```bash
  # Check that Kong now serves a Let's Encrypt production certificate
  curl -ivko /dev/null https://proxy.home.<YOUR_DOMAIN>

  # Check if secure connection is established (no certificate warnings). If there are errors check that the DNS name matches your actual domain configured in local DNS server (pi-hole).   
  curl -ivo /dev/null https://proxy.home.<YOUR_DOMAIN>

  # Now, check Homepage loads when accessed using browser and curl. With curl you should see HTML output in the terminal. if not, check Homepage http route matches your domain. 
  curl -iv https://proxy.home.<YOUR_DOMAIN>

  # Check also all other services are accessible via Homepage using the browser.

  ```
## License

This project is licensed under the [MIT License](./LICENSE).

