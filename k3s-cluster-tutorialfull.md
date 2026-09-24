# Building a 2-Node K3s Cluster on Raspberry Pi 4 (Debian Trixie)

A lab manual for going from two fresh Raspberry Pi OS Lite 64-bit installs to a working, expandable K3s cluster — no Docker involved.

**Your two Pis, throughout this guide:**

```text
                    LAN
                     │
          ┌──────────┴──────────┐
          │                     │
   Raspberry Pi 4 #1     Raspberry Pi 4 #2
   k3s-master             k3s-worker
   K3s Server             K3s Agent
   <MASTER_IP>             <WORKER_IP>
          │                     │
          └──────── K3s ────────┘
```

Throughout this guide, replace:

- `<MASTER_IP>` — Pi #1's real LAN IP address (e.g. `192.168.1.50`)
- `<WORKER_IP>` — Pi #2's real LAN IP address (e.g. `192.168.1.51`)
- `<TOKEN>` — the exact node token you copy off the master in Step 10

Every command block is labeled with exactly which Pi it runs on. If a block has no label, it's a command you run on your own laptop/desktop.

## What's actually different on Trixie (read this first)

You asked not to get a stale tutorial, so here's what changed and what didn't, verified against current K3s and Raspberry Pi OS documentation and issue trackers:

- **Still true, don't skip it:** Raspberry Pi's kernel still ships with the memory cgroup controller *compiled in but switched off by default* — this is unchanged on Trixie. You still need the `cgroup_enable=memory cgroup_memory=1` boot parameters (Step 5). Some people assume newer kernels fixed this; they haven't.
- **Still true:** Kubernetes (and therefore K3s) still refuses to start if swap is active, *by default*. Swap support was added as an alpha feature in Kubernetes 1.22 and reached beta in 1.28, but it's opt-in configuration, not the default behavior — so disabling swap (Step 6) is still the simplest correct path.
- **Changed / no longer needed:** Older guides tell you to force `iptables-legacy` on Debian because of nftables bugs. That advice dates to Debian Buster's rocky transition to nftables and to a specific buggy iptables release range (1.8.0–1.8.4). Trixie ships a current nftables-based iptables, well past that range — you almost certainly don't need to touch this (Step 7 covers the one-line check and the fallback if you ever do).
- **Changed:** The boot config file moved from `/boot/cmdline.txt` to `/boot/firmware/cmdline.txt` back in the Bookworm era, and that's still where it lives in Trixie.
- **New consideration old tutorials never mention:** Since Bookworm, Raspberry Pi OS manages networking with NetworkManager instead of dhcpcd. NetworkManager can quietly "manage" (and break) the virtual network interfaces K3s's pod network creates, especially across reboots. There's a one-time, one-file fix in Step 4.
- **Simpler than you'd expect:** The install script already creates a `kubectl` command for you (it's a symlink to the `k3s` binary) — you don't need the manual-symlink trick some older tutorials show. The only reason you still need `sudo` is file permissions on the config, not a missing command (Step 15).
- **Confirmed current:** `metrics-server`, Traefik, ServiceLB, local-path-provisioner, and CoreDNS are all still bundled and enabled by default in current K3s.

---

## Step 1 — Plan the cluster

**What we're doing:** Understanding the two roles before you type anything.

**The server (control-plane) node — Pi #1, `k3s-master`:**
Runs the Kubernetes API server, the scheduler, the controller-manager, and (for a single-server setup like this one) an embedded SQLite database that holds all cluster state. It also runs containerd and kubelet, which means — unlike a `kubeadm`-based cluster — a K3s server node is *not* tainted against running regular workloads by default. It can and will run ordinary pods alongside its control-plane duties unless you explicitly tell it not to. For a 2-Pi lab, that's exactly what you want: wasting an entire Pi on control-plane-only duty would be a waste of a node.

**The worker (agent) node — Pi #2, `k3s-worker`:**
Runs only kubelet, kube-proxy, and containerd. It has no control-plane components at all — it just executes pods the scheduler assigns to it and reports status back to the server.

**How they communicate:**
The worker makes an *outbound* connection to the server on port 6443 (the Kubernetes API). K3s then tunnels most kubelet traffic back through that same connection ("reverse tunneling"), which is part of why K3s needs so few open ports compared to a full kubeadm cluster. Pod-to-pod traffic across the two Pis travels over a VXLAN overlay network (Flannel, port 8472/UDP) — more on this in Step 7.

**What K3s actually installs:**
A single ~80MB binary (`k3s`) that embeds the entire Kubernetes control plane, kubelet, kube-proxy, and several bundled add-ons: containerd (container runtime), Flannel (pod networking/CNI), CoreDNS (cluster DNS), Traefik (ingress controller), ServiceLB (a simple bare-metal load balancer), local-path-provisioner (default storage), and metrics-server (powers `kubectl top`). One binary, one systemd service, no separate pieces to install.

**Do you need Docker?** No. K3s uses **containerd** as its container runtime, bundled inside the same binary. Nothing in this guide touches Docker, and nothing needs to.

**What's included vs. a "full" Kubernetes install:** Functionally, everything — K3s is a certified, conformant Kubernetes distribution. It just ships lighter-weight defaults for a few components (SQLite instead of etcd for a single server, Traefik instead of no ingress controller, ServiceLB instead of requiring a cloud load balancer) — all of which you can disable and replace later if you outgrow them.

---

## Step 2 — Verify the operating system

**What we're doing:** Confirming both Pis are actually 64-bit, have enough headroom, and match what the rest of this guide assumes.

**Which Pi:** Run this whole block on **both** Pi #1 and Pi #2, one at a time over SSH.

**RUN ON BOTH (Pi #1 and Pi #2, separately)**
```bash
cat /etc/os-release
uname -m
uname -r
hostname
hostname -I
free -h
df -h /
```

**What to look for in the output:**

| Command | What it tells you | What you want to see |
|---|---|---|
| `cat /etc/os-release` | Distro and version | `PRETTY_NAME="Debian GNU/Linux trixie/sid"` or similar, confirming Trixie |
| `uname -m` | CPU architecture | `aarch64` — this confirms 64-bit. If you see `armv7l` instead, you're actually on a 32-bit OS and this whole guide's install commands still work (K3s ships ARMv7 builds too), but your kernel cgroup situation and available RAM addressing will differ from what's described here |
| `uname -r` | Kernel version | Something in the `6.12.x` line or newer (Trixie ships the 6.12 LTS series) |
| `hostname -I` | Current LAN IP(s) | Note this down — it's your `<MASTER_IP>` or `<WORKER_IP>` for the rest of the guide |
| `free -h` | Total RAM | K3s's own minimum is ~512MB, but for comfortable headroom running the full control plane plus workloads, 4GB+ is a much nicer experience than 2GB, especially on the server node |
| `df -h /` | Free disk space | A few GB free is plenty for a lab cluster; K3s itself is small, container images are what add up over time |

**If something looks wrong:** If `uname -m` doesn't say `aarch64` and you specifically flashed the 64-bit image, re-flash with Raspberry Pi Imager and double-check you picked "Raspberry Pi OS Lite (64-bit)," not the 32-bit Lite image — they look nearly identical in the picker.
## Step 3 — Set hostnames

**What we're doing:** Giving each Pi a name that matches its role, so `kubectl get nodes` output is readable and you never have to guess which SSH session is which.

**Does Kubernetes actually require this?** No. K3s registers each node under whatever hostname the OS reports at install time (or a name you override with `--node-name`). The cluster itself doesn't care what you call the boxes — this step is entirely for *your* sanity, not K3s's requirements.

**RUN ON MASTER (Pi #1)**
```bash
sudo hostnamectl set-hostname k3s-master
```

**RUN ON WORKER (Pi #2)**
```bash
sudo hostnamectl set-hostname k3s-worker
```

**Verify on each:**
```bash
hostname
```
Should print `k3s-master` or `k3s-worker` respectively. Your existing SSH session's shell prompt may still show the old name until you reconnect — that's cosmetic, the change already took effect.

**Optional but convenient — add both to `/etc/hosts` on both Pis**, so you can SSH by name instead of IP:

**RUN ON BOTH**
```bash
sudo tee -a /etc/hosts > /dev/null <<EOF
<MASTER_IP> k3s-master
<WORKER_IP> k3s-worker
EOF
```
Replace `<MASTER_IP>` and `<WORKER_IP>` with the real addresses from Step 2's `hostname -I` output before running this. Again — this is purely for your convenience SSHing around; the actual cluster join in Steps 8–11 uses raw IP addresses, not these hostnames.

---

## Step 4 — Update the Pis

**What we're doing:** Getting current packages and a current kernel before installing anything that depends on kernel features (cgroups, in the next step).

**RUN ON BOTH**
```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove -y
```
`full-upgrade` (rather than plain `upgrade`) is the right choice here because it will also pull in a newer kernel package if one is available, and correctly handles any package removals that come with it — plain `upgrade` refuses to do that.

**Reboot if a kernel or firmware package was updated** (the `apt` output will mention `linux-image` or `raspi-firmware` if so):
```bash
sudo reboot
```

### One-time fix: stop NetworkManager from fighting K3s's network interfaces

This is the one genuinely new gotcha that didn't exist on older, dhcpcd-based Raspberry Pi OS releases. Since Bookworm, Raspberry Pi OS manages networking with NetworkManager by default. NetworkManager doesn't know about the virtual interfaces K3s's pod network (Flannel) creates — `cni0` and `flannel.1` — and can try to "manage" them, most commonly causing pod-to-pod networking to misbehave after a reboot (which is exactly what Step 19 tests for). The fix is one small config file, and it's safe to add now even though those interfaces don't exist yet — NetworkManager will simply ignore them the moment K3s creates them later.

**RUN ON BOTH**
```bash
sudo tee /etc/NetworkManager/conf.d/90-flannel-unmanaged.conf > /dev/null <<'EOF'
[keyfile]
unmanaged-devices+=interface-name:cni0;interface-name:flannel.1
EOF
sudo systemctl reload NetworkManager
```

**Verify it took:**
```bash
NetworkManager --print-config | grep -A2 unmanaged-devices
```
You should see `cni0` and `flannel.1` listed. It's fine if this looks like a no-op right now — the interfaces don't exist until after Step 8/11 — you're just making sure the rule is in place before they show up.

---

## Step 5 — Check and configure cgroups

**What we're doing, and why it matters:** cgroups (control groups) are a Linux kernel feature that lets the container runtime track and enforce per-container CPU, memory, and process limits. Kubernetes' kubelet component refuses to run correctly without the `memory` and `cpuset` cgroup controllers active — this isn't optional plumbing, it's a hard requirement.

**Why this needs a manual step on a Pi and not on, say, a Debian VM:** Standard Debian on x86 ships with every relevant cgroup controller already active. Raspberry Pi's own kernel build has historically shipped with the memory cgroup controller compiled in but *disabled at boot* to save a small amount of RAM overhead — and this has **not changed** on Trixie's current kernel. This is the single most common thing that trips people up on a fresh Pi K3s install, and it's exactly as true today as it was three years ago.

**Check the current state first — don't blindly edit boot files:**
```bash
cat /proc/cgroups
```
Look at the `memory` line's last column:
```text
#subsys_name    hierarchy    num_cgroups    enabled
cpuset          0            10             1
cpu             0            10             1
memory          0            10             0     <-- 0 means disabled
```
If `memory` shows `1`, you're already fine — **skip straight to Step 6**, you don't need to touch `cmdline.txt`. If it shows `0` (which is what a fresh Raspberry Pi OS Trixie install will show), continue below.

**RUN ON BOTH**

Edit the boot command line — note the path, which changed from `/boot/cmdline.txt` in the Bookworm era and remains at this location in Trixie:
```bash
sudo cp /boot/firmware/cmdline.txt /boot/firmware/cmdline.txt.bak
sudo sed -i '1 s/$/ cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory/' /boot/firmware/cmdline.txt
```
This appends the three parameters to the *end of the existing single line* in the file — cmdline.txt must remain exactly one line, with no line breaks, or the Pi won't boot. The command above does this safely; if you ever edit the file by hand, be careful not to press Enter anywhere in it.

**Reboot, then re-check:**
```bash
sudo reboot
```
```bash
cat /proc/cgroups
```
`memory` should now show `1`.

**If it still shows `0` after a correct edit and reboot:** this has occasionally regressed on some Raspberry Pi kernel builds even with the right parameters in place (there are open Raspberry Pi kernel issues tracking exactly this on certain recent kernel versions). If you hit this: confirm the parameters actually made it into the running kernel command line with `cat /proc/cmdline` (not `cmdline.txt` — this shows what the kernel actually booted with, which can differ if there's a typo or a stray line break in the file), and try a `sudo apt full-upgrade` to pick up any pending kernel/firmware fix before digging further.
## Step 6 — Swap

**Should it be disabled? Yes — here's why, precisely.** Kubernetes' kubelet has, since its earliest versions, refused to start on a node with active swap, and **that default has not changed**, even though this is genuinely more nuanced than it used to be. Kubernetes added experimental swap support as an alpha feature in v1.22 and promoted it to beta in v1.28, continuing to mature since — but it's strictly opt-in: you have to explicitly configure `failSwapOn: false` and a swap behavior in the kubelet config. K3s doesn't do this for you out of the box. So unless you go out of your way to enable it, the plain, current, correct answer for a straightforward cluster is still: disable swap.

**Check whether you have any:**
```bash
free -h
```
Raspberry Pi OS activates a small (100MB) swap file by default via `dphys-swapfile`, so you'll very likely see a nonzero `Swap` line even on a fresh install.

**RUN ON BOTH**
```bash
sudo dphys-swapfile swapoff
sudo systemctl disable dphys-swapfile
```

** Edit by me**

```
sudo dphys-swapfile swapoff
sudo systemctl disable dphys-swapfile
```
Resulted in an an error:

```
Failed to disable unit: Unit dphys-swapfile.service does not exist
```
But I was able to fix it with

```
sudo swapoff -a
```

**Verify:**
```bash
swapon --show
free -h
```
`swapon --show` should print nothing, and `free -h`'s `Swap` row should read all zeros.

---

## Step 7 — Networking

**What we're doing:** Confirming the two Pis can actually reach each other on the ports K3s needs, and understanding why (per K3s's own current networking requirements docs).

| Port | Protocol | Purpose | Direction | Needed for this 2-node lab? |
|---|---|---|---|---|
| 6443 | TCP | Kubernetes API server | Worker → Server, and your laptop → Server for `kubectl` | **Yes, always** |
| 8472 | UDP | Flannel VXLAN (pod-to-pod overlay network) | All nodes ↔ all nodes | **Yes** (unless you swap out Flannel for a different CNI, which this guide doesn't) |
| 10250 | TCP | Kubelet API — used by `metrics-server` to pull stats directly from each node | All nodes ↔ all nodes | **Yes**, since metrics-server is on by default |
| 2379–2380 | TCP | Embedded etcd client/peer traffic | Server ↔ server | **No** — only relevant if you later add a second *server* node for HA (see Step 20); a single-server setup like this one uses embedded SQLite, not etcd, and never opens these |
| 80, 443 | TCP | Traefik's LoadBalancer service (Ingress) | Anything reaching your Ingress-exposed apps | Only once you actually expose something via Ingress — not required for the cluster to function |

**Do you need to touch your router or firewall?** No, as long as both Pis sit on the same LAN/switch with no VLAN segmentation between them — everything above is plain LAN traffic, nothing needs to reach the internet or come back in from it. K3s's own networking guidance is, in fact, to remove OS-level firewalls entirely if you're not otherwise relying on one; Raspberry Pi OS doesn't enable `ufw` or any packet filtering by default, so unless you've turned one on yourself, there's nothing to configure here.

**A note on iptables vs. nftables (an old tutorial trap):** Older guides insist on forcing legacy iptables mode on Debian because of nftables transition bugs from the Debian Buster era, plus a specific buggy iptables release range (1.8.0–1.8.4). Check your version:
```bash
iptables --version
```
Trixie ships a current nftables-backed iptables, well outside that buggy range — you do not need to switch to legacy mode. If you ever do hit an obscure iptables-related crash loop (rare on Trixie), K3s ships its own known-good iptables binary that you can force it to use instead of the OS's, by adding `--prefer-bundled-bin` to the install command in Step 8 — but don't add this preemptively; it's a fix for a specific symptom, not a preventative measure.

**Test basic connectivity now** (before K3s is even installed, this just confirms the two Pis can see each other at all):

**RUN ON MASTER**
```bash
ping -c 3 <WORKER_IP>
```

**RUN ON WORKER**
```bash
ping -c 3 <MASTER_IP>
```
Both should get replies with low latency (well under a millisecond on a wired LAN). If either times out, fix basic network connectivity (check the Ethernet cable, switch port, and that both Pis actually got DHCP leases) before going any further — nothing past this point will work without this.

You can't usefully test port 6443 yet since nothing is listening on it until Step 8 — we'll test it for real right after installing the server.
## Step 8 — Install K3s on the server

**What we're doing:** Installing K3s in its default "server" mode on `k3s-master`. This one command stands up a fully functional, single-node Kubernetes cluster — the worker in Step 11 joins *this*.

**RUN ON MASTER**
```bash
curl -sfL https://get.k3s.io | sh -
```

This is still the current, official install method — straight from K3s's own quick-start documentation, unchanged in substance for years because it doesn't need to change.

**What this one line actually does:**
- Detects your architecture (arm64) and downloads the matching K3s binary to `/usr/local/bin/k3s`
- Creates convenience symlinks: `kubectl`, `crictl`, and `ctr` all become the `k3s` binary invoked with a subcommand baked in — this is why `kubectl` will already work as its own command later, with no manual symlinking needed
- Writes helper scripts `k3s-killall.sh` and `k3s-uninstall.sh` to `/usr/local/bin` (handy if you ever need to wipe and start over)
- Generates a random node token (since none was supplied) and writes it to `/var/lib/rancher/k3s/server/node-token`
- Sets up and starts a systemd service named **`k3s`** (note this name — the worker's service will be named differently, `k3s-agent`, which matters for troubleshooting later), enabled to start on boot
- Writes the cluster's kubeconfig to `/etc/rancher/k3s/k3s.yaml`, readable only by root by default
- Starts running immediately as a single-node cluster, with containerd as its embedded container runtime and an embedded SQLite database for cluster state (since this is a single server, not an HA setup)

No separate container runtime, no Docker, no additional packages — this one binary is the whole thing.

---

## Step 9 — Verify the server

**RUN ON MASTER**
```bash
systemctl status k3s
```
**Expect:** `Active: active (running)` in green, with no recent restart loops in the log lines shown below it.

```bash
sudo k3s kubectl get nodes
```
**Expect:**
```text
NAME         STATUS   ROLES                  AGE   VERSION
k3s-master   Ready    control-plane,master   45s   v1.3x.y+k3s1
```
(Your exact version number will be whatever the current stable K3s release is — that's expected and fine.)

**What `Ready` actually means here:** the kubelet on that node has registered with the API server, reported healthy disk/memory/PID conditions, and confirmed its network plugin (Flannel) initialized successfully. If you see `NotReady` instead, jump to the troubleshooting section (Step 21) — don't proceed to the worker install until this says `Ready`.

**If `systemctl status k3s` shows a crash loop:** this is almost always the cgroups issue from Step 5 (K3s will refuse to start cleanly without the memory cgroup active) — double check `cat /proc/cgroups` before anything else.

---

## Step 10 — Securely get the node token

**What it is:** A shared secret that lets any machine possessing it — and with network access to port 6443 — join your cluster as a server or an agent. Anyone with this token effectively has root-equivalent control over your entire cluster: they could deploy pods, read secrets stored in it, or join a rogue node. Treat it exactly like a root password, not like a casual API key.

**RUN ON MASTER**
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
This prints a long string starting with `K10`. Copy it (including the whole string) — you'll paste it into the worker install command in the next step.

**Handling it safely:**
- Don't paste it into a chat app, a GitHub issue, a forum post, or anywhere else outside your own terminal — this applies even (especially) if you're asking someone for help debugging your cluster.
- It doesn't expire and doesn't rotate on its own, so you only need to fetch it again in the future if you're setting up another new node (see Step 20) and didn't save it anywhere.
- If you ever suspect it's been exposed, K3s can generate a fresh one: `sudo k3s token rotate` (this requires re-joining any existing agents with the new token — not something to do casually on a running cluster).

**Where it's used:** as the `K3S_TOKEN` value in the worker's install command, next.
## Step 11 — Install the worker

**RUN ON WORKER**
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN='<TOKEN>' sh -
```

**Replace before running:**
- `<MASTER_IP>` → Pi #1's real LAN IP (from Step 2)
- `<TOKEN>` → the exact string from Step 10, kept inside the single quotes exactly as shown — the quotes matter because the token can contain characters a shell would otherwise interpret

This is still the current, documented method for joining an agent — confirmed directly against K3s's own quick-start docs.

**What happens during this install:** the script detects that `K3S_URL` is set and installs K3s in **agent mode** instead of server mode this time — no API server, no scheduler, no datastore, just kubelet, kube-proxy, and containerd. It uses the token to authenticate to `https://<MASTER_IP>:6443`, registers itself as a new node, and starts a systemd service named **`k3s-agent`** (different from the master's `k3s` service name — you'll need this exact name for `systemctl`/`journalctl` commands on this Pi from now on).

**If the install hangs or fails immediately:** this is almost always one of: a typo in `<MASTER_IP>` or `<TOKEN>`, port 6443 not reachable (re-test with `nc -zv <MASTER_IP> 6443` from the worker — this will actually work now that the server is running), or swap/cgroups not sorted on the worker (Steps 5–6 apply to *both* Pis, not just the master).

---

## Step 12 — Verify the worker joined

**RUN ON MASTER**
```bash
sudo k3s kubectl get nodes -o wide
```
**Expect:**
```text
NAME         STATUS   ROLES                  AGE   VERSION       INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                       KERNEL-VERSION   CONTAINER-RUNTIME
k3s-master   Ready    control-plane,master   5m    v1.3x.y+k3s1  <MASTER_IP>     <none>        Debian GNU/Linux 13 (trixie)   6.12.x-v8-16k+   containerd://1.7.x-k3s1
k3s-worker   Ready    <none>                 30s   v1.3x.y+k3s1  <WORKER_IP>     <none>        Debian GNU/Linux 13 (trixie)   6.12.x-v8-16k+   containerd://1.7.x-k3s1
```

**Column by column:** `NAME` is the node's hostname; `STATUS` should read `Ready` for both; `ROLES` shows `control-plane,master` only for the server — the worker legitimately shows `<none>`, that's not an error, it just means "no control-plane role," which is exactly what an agent is; `AGE` is time since the node registered; `VERSION` is the K3s (and therefore Kubernetes) version; `INTERNAL-IP` should match the LAN IPs you've been using; `EXTERNAL-IP` is `<none>` because this is a bare-metal LAN setup with no cloud provider integration; `OS-IMAGE`/`KERNEL-VERSION` confirm Trixie and the 6.12 kernel line; `CONTAINER-RUNTIME` confirms containerd, never Docker.

```bash
sudo k3s kubectl get pods -A
```
**Expect** something like this (exact pod-name suffixes will differ):
```text
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   coredns-xxxxxxxxxx-xxxxx                  1/1     Running   0          5m
kube-system   local-path-provisioner-xxxxxxxxxx-xxxxx   1/1     Running   0          5m
kube-system   metrics-server-xxxxxxxxxx-xxxxx           1/1     Running   0          5m
kube-system   traefik-xxxxxxxxxx-xxxxx                  1/1     Running   0          5m
kube-system   svclb-traefik-xxxxxxxxxxxx-xxxxx          2/2     Running   0          5m
```
These are the bundled add-ons from Step 1: CoreDNS (cluster DNS), local-path-provisioner (default storage), metrics-server (powers `kubectl top`), Traefik (ingress controller), and an `svclb-*` pod per node backing Traefik's LoadBalancer service (this is ServiceLB at work — it runs one small proxy pod per node so that hitting *either* Pi's IP on ports 80/443 reaches Traefik). All of these should show `Running` with `0` or a low restart count. If any is stuck `Pending` or `CrashLoopBackOff` at this stage, see Step 21.
## Step 13 — Test Kubernetes with a real deployment

**What we're doing:** Proving pods actually run and are reachable, end to end, using nginx as a disposable test app.

**RUN ON MASTER** (all of Step 13 runs here — `kubectl` talks to the API server over the network, it doesn't need to run *on* the node hosting a given pod)

**1. Create the deployment:**
```bash
sudo k3s kubectl create deployment nginx-test --image=nginx
```
**Expect:** `deployment.apps/nginx-test created`

**2. Expose it** (NodePort makes it reachable from outside the cluster, on a random high port, via *any* node's IP — that's the point of NodePort):
```bash
sudo k3s kubectl expose deployment nginx-test --port=80 --type=NodePort
```
**Expect:** `service/nginx-test exposed`

**3. Check the pod:**
```bash
sudo k3s kubectl get pods -o wide
```
**Expect** a `nginx-test-xxxxxxxxxx-xxxxx` pod, `STATUS: Running`, with a `NODE` column showing either `k3s-master` or `k3s-worker` — whichever the scheduler happened to pick (more on why in Step 14).

**4. Check the service and find your port:**
```bash
sudo k3s kubectl get svc nginx-test
```
**Expect:**
```text
NAME         TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-test   NodePort   10.43.x.x      <none>        80:3XXXX/TCP   30s
```
The number after the colon in `PORT(S)` (something in the 30000–32767 range) is your NodePort.

**5. Access it from your laptop** (not from either Pi — the whole point is testing external reachability):
```bash
curl http://<MASTER_IP>:<NODEPORT>
```
Or open `http://<MASTER_IP>:<NODEPORT>` in a browser. You should get nginx's default "Welcome to nginx!" page.

**Try the *worker's* IP with the same port too** — `curl http://<WORKER_IP>:<NODEPORT>` — it should also work, even if the pod is physically running on the master. That's NodePort's job: kube-proxy on every node forwards traffic on that port to wherever the pod actually lives.

**6. Clean up:**
```bash
sudo k3s kubectl delete service nginx-test
sudo k3s kubectl delete deployment nginx-test
```

---

## Step 14 — How scheduling works

You already saw the scheduler in action in Step 13 — here's what actually happened.

**The kube-scheduler** (bundled into the server's control-plane components) decides which node runs each pod, based on requested CPU/memory (none were set for our nginx test, so it had free rein), current node capacity, and any taints/tolerations or affinity rules (none apply here either). With no constraints and two healthy nodes, placement for a single test pod is essentially "whichever node the scheduler judged had more room" — don't read anything meaningful into which Pi it landed on.

**The core building blocks, in plain terms:**
- **Pod** — the smallest deployable unit; one or more containers that share networking and storage. You almost never create bare pods directly (we didn't either — `create deployment` made one for us).
- **Deployment** — manages a desired number of identical pod replicas, handles rolling updates, and automatically recreates a pod if it crashes or its node fails (this is what "self-healing" means in Kubernetes — it's a control-plane feature, not something the pod does for itself).
- **Service** — a stable network name/IP in front of a *changing* set of pods (pods are ephemeral and get new IPs when recreated; Services give you something constant to point at instead).
- **Namespace** — a way to logically partition resources within one cluster. System components live in `kube-system`; anything you deploy without specifying a namespace lands in `default`.
- **Replicas** — how many identical copies of a pod a Deployment should keep running at once (we used the default of 1 for the nginx test; a real app would typically use 2+ for redundancy).

**See placement directly any time:**
```bash
sudo k3s kubectl get pods -A -o wide
```
The `NODE` column is your answer for "where is this actually running."
## Step 15 — Make kubectl easier to use

**A correction worth knowing up front:** you might expect you need to manually create a `kubectl` command, since so far you've been typing `sudo k3s kubectl`. You don't — the install script in Step 8 already created a `kubectl` symlink pointing at the `k3s` binary. Try it right now:

**RUN ON MASTER**
```bash
sudo kubectl get nodes
```
This already works. The *only* reason you still need `sudo` is that `/etc/rancher/k3s/k3s.yaml` (the kubeconfig `kubectl` reads to know how to connect) is owned by root and readable only by root by default — that's a file-permissions problem, not a missing-command problem, and it's what the rest of this step actually fixes.

### Drop the `sudo` requirement, on the Pi itself

**RUN ON MASTER**
```bash
mkdir -p ~/.kube
sudo k3s kubectl config view --raw | tee ~/.kube/config > /dev/null
chmod 600 ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```
Now plain `kubectl get nodes` (no `sudo`, no `k3s` prefix) works directly.

### Using it remotely from your laptop

**On your laptop**, install a standalone `kubectl` (via your OS's package manager, or Kubernetes' own docs — this is a separate, small binary from K3s's bundled one).

**RUN ON MASTER** — copy the same config back out:
```bash
sudo k3s kubectl config view --raw
```
Copy this output (or `scp` the `~/.kube/config` file you created above) to your laptop, e.g. as `~/.kube/config`.

**Edit one line in the copied file:** it will contain `server: https://127.0.0.1:6443` — this is correct for running `kubectl` *on* the server itself, but useless from anywhere else. Change it to:
```yaml
server: https://<MASTER_IP>:6443
```
Now `kubectl get nodes` from your laptop should reach the cluster directly over the network.

**The security implications, stated plainly:** this file contains an embedded client certificate and private key that grant full administrator access to your entire cluster — equivalent to a root password. Anyone who has it can read every Secret, deploy anything, or delete anything in your cluster. Treat it like an SSH private key: `chmod 600` it, never commit it to a git repo (public or private), never email it or paste it into chat. It does not expire on its own.

**An alternative some guides mention** — installing K3s with `--write-kubeconfig-mode 644` makes the file world-readable *on the Pi itself*, slightly simplifying the copy-out step above. It doesn't change any of the security guidance above — the file is exactly as sensitive either way; this flag only affects who can *read the file from the Pi's own disk*, not who it grants access to once copied elsewhere.

---

## Step 16 — OPTIONAL: Install Helm

**What Helm is:** a package manager for Kubernetes — think of it as `apt` or `brew`, but for Kubernetes applications. Instead of hand-writing and applying a dozen YAML files to install something like a monitoring stack or a database, you run one `helm install` command against a pre-packaged "chart" with configurable values. You don't need it for anything in this guide — it becomes useful the moment you want to install a more complex third-party application later.

**RUN ON MASTER** (or on your laptop, once Step 15's remote `kubectl` access is set up — Helm just needs a working kubeconfig, it doesn't care where it runs from)
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
This is still the current, official install method; the script auto-detects your architecture, so it works the same way on the Pi's arm64 as it would on an amd64 machine.

**Verify:**
```bash
helm version
```
Should print a `version.BuildInfo{Version:"v3.x.x", ...}` line.
## Step 17 — What K3s already includes by default

You saw all of these running back in Step 12's `get pods -A` output. Confirmed against current K3s documentation, all of the following ship **enabled by default** — don't install any of them separately, and don't let an older guide talk you into it:

| Component | What it does | Disable flag (if you ever needed to) |
|---|---|---|
| containerd | Container runtime — not optional, this is fundamental to how K3s runs anything | n/a |
| CoreDNS | Cluster-internal DNS (lets pods find Services by name) | `--disable=coredns` |
| Traefik | Ingress controller — handles `Ingress` resources, exposes itself on 80/443 | `--disable=traefik` |
| ServiceLB | K3s's built-in bare-metal `LoadBalancer` implementation (this is what backs those `svclb-*` pods) | `--disable=servicelb` |
| local-path-provisioner | Default dynamic storage provisioner, backed by node-local directories | `--disable=local-storage` |
| metrics-server | Powers `kubectl top nodes` / `kubectl top pods` | `--disable=metrics-server` |

You'd only ever pass one of these disable flags if you specifically wanted to replace a component with an alternative (e.g., swapping ServiceLB for MetalLB, or Traefik for ingress-nginx) — not something this lab needs.

**Try it now that metrics-server has had a few minutes to collect data:**
```bash
sudo kubectl top nodes
```

---

## Step 18 — Storage

**How local-path-provisioner actually works:** when something requests storage using the `local-path` StorageClass (the default — confirm with `kubectl get storageclass`), the provisioner creates a plain directory under `/opt/local-path-provisioner/` **on whichever node the consuming pod gets scheduled to**, and binds that directory as the PersistentVolume.

**The limitation that matters for a 2-Pi cluster:** this storage is *not* shared or replicated between your two Pis. If a pod using this kind of storage is scheduled to `k3s-worker`, its data lives only on `k3s-worker`'s SD card/SSD. Kubernetes remembers this and will keep trying to reschedule that pod back onto `k3s-worker` specifically — if that Pi is down, the pod simply won't run anywhere else until it's back, rather than "failing over" to the other node the way you might expect. There's no automatic replication like you'd get from Ceph, Longhorn, or an NFS-backed StorageClass — for a home lab this is a perfectly reasonable tradeoff, just know it's there.

**A tiny test PVC:**

**RUN ON MASTER**
```bash
cat <<EOF | sudo kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 100Mi
EOF
```

```bash
sudo kubectl get pvc test-pvc
```
**Expect `STATUS: Pending`** — and that's correct, not a bug. `local-path`'s binding mode is `WaitForFirstConsumer`, meaning it deliberately waits until an actual pod references the claim before creating the underlying storage — that way it can create it on whichever node the pod actually lands on, rather than guessing.

**Give it a consumer to bind to:**
```bash
cat <<EOF | sudo kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pvc-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - mountPath: /data
      name: test-vol
  volumes:
  - name: test-vol
    persistentVolumeClaim:
      claimName: test-pvc
EOF
```

```bash
sudo kubectl get pvc test-pvc
```
Now it should show `STATUS: Bound`.

**Clean up:**
```bash
sudo kubectl delete pod test-pvc-pod
sudo kubectl delete pvc test-pvc
```
## Step 19 — Reboot test

**What we're doing:** Confirming the cluster survives real-world power cycles, and understanding exactly what does and doesn't keep working when the single control-plane node is briefly unavailable — this matters a lot for a home lab where power blips happen.

**1. Reboot the worker first** (the lower-stakes test):

**RUN ON WORKER**
```bash
sudo reboot
```

**Watch from the master** while it's down and as it comes back:
```bash
sudo kubectl get nodes -w
```
(`-w` watches for changes live — press Ctrl+C to stop watching once you're satisfied.) Expect `k3s-worker` to flip to `NotReady` shortly after the reboot starts, then back to `Ready` once it boots and its `k3s-agent` service auto-starts — this should happen without you touching anything, since the install script enables the service to start on boot by default.

**2. Now reboot the master:**

**RUN ON MASTER**
```bash
sudo reboot
```

**What happens while it's down** (this is the important part your outline specifically asked about):
- The API server is completely unreachable — any `kubectl` command from anywhere will time out or refuse to connect.
- Pods that were **already running on the worker keep running** — kubelet manages its already-started containers independently of whether it can reach the API server; it doesn't need permission to keep something alive that it already started.
- **No new scheduling decisions happen**, and no automatic pod recreation happens at the cluster level — if a pod on the worker crashes during this window, kubelet's own local restart policy (e.g., a Deployment's default "Always" restart) still restarts *that specific container* locally, since that's a kubelet-level behavior, not something requiring the API server. But if the whole pod needs to be rescheduled elsewhere, that can't happen until the control plane is back.
- **DNS may break cluster-wide during this window** if the CoreDNS pod happened to be scheduled on the master — any workload doing a fresh DNS lookup (e.g., a new connection to another Service by name) will fail until the master returns, even though the worker itself is otherwise fine.

**After the master comes back:**
```bash
sudo kubectl get nodes
```
Both nodes should show `Ready` again once `k3s.service` finishes restarting (also enabled by default to start on boot).

**The single-control-plane limitation, stated directly:** with only one server node, that node is a single point of failure for the *control plane* — not for already-running workloads, but for anything requiring cluster-level decisions (scheduling, self-healing, `kubectl` access) while it's down. K3s does support true high availability via a second and third server node using an embedded etcd datastore instead of SQLite (etcd needs an odd number of members — 3 is the normal minimum — for quorum). That's a deliberate next step up in complexity beyond a 2-Pi lab, not something to bolt on casually, but it's worth knowing the growth path exists officially: see K3s's [High Availability with Embedded etcd](https://docs.k3s.io/datastore/ha-embedded) docs if you ever want to go there.

---

## Step 20 — Adding more Pis to the cluster later

This is exactly as straightforward as it should be — you built this cluster in a way that doesn't need any reconfiguration to grow. Adding a third (or fourth, fifth...) Pi as another **worker** is simply repeating Steps 2–7 and Step 11 on the new hardware:

1. Flash the new Pi with the same Raspberry Pi OS Lite 64-bit / Trixie image, and give it a unique hostname (e.g. `k3s-worker2`) — Step 3.
2. Run the update, NetworkManager fix, cgroups check/fix, and swap-disable steps — Steps 4–6. These are per-machine OS prep, not cluster configuration, so they're identical regardless of how many Pis you already have joined.
3. Confirm it can reach `<MASTER_IP>:6443` — Step 7.
4. Fetch the **same** node token again (it doesn't expire or change on its own):
   ```bash
   sudo cat /var/lib/rancher/k3s/server/node-token
   ```
5. Run the exact same agent install command from Step 11 on the new Pi, pointing at the same `<MASTER_IP>` and token.
6. Confirm it shows up:
   ```bash
   sudo kubectl get nodes
   ```
   No restart, no re-install, and no interruption to the existing two nodes — the running cluster just gains a third member.

**Is there a limit?** Nothing K3s imposes for agent nodes at this scale — a home cluster of a handful of Pis is well within normal territory. The practical ceiling is your master Pi's own CPU/RAM headroom to run the control plane for a larger fleet, and your switch's port count.

**A slightly more security-conscious option for future joins:** instead of reusing the same long-lived node token forever, K3s can generate short-lived, single-purpose bootstrap tokens:
```bash
sudo k3s token create --ttl 1h
```
This prints a new token that expires on its own after the time you specify, useful if you don't want to keep the master token itself lying around in your notes indefinitely. For a home lab, reusing the saved node token (as above) is simpler and perfectly fine — this is just worth knowing exists.

**If you ever want to add a second *server* (not just another worker)** for real HA instead of just more capacity: that's a different install flag (`--server` and `K3S_TOKEN` combined, rather than `K3S_URL`+`K3S_TOKEN`) and it automatically switches your datastore from embedded SQLite to embedded etcd the moment you do it. That's a meaningfully bigger step than adding a worker — see the HA docs linked in Step 19 before attempting it, rather than improvising.
## Step 21 — Troubleshooting

### Worker won't join

**RUN ON WORKER**
```bash
systemctl status k3s-agent
journalctl -u k3s-agent -e --no-pager
```
Look for the actual error near the end of the log. The usual suspects, roughly in order of likelihood: a typo in `<MASTER_IP>` or `<TOKEN>` in the install command (re-run it with the corrected values — it's safe to re-run), port 6443 unreachable (see the connectivity test below), or the worker itself failing cgroups/swap checks from Steps 5–6 (those apply to both Pis, not just the master).

### Master isn't Ready

**RUN ON MASTER**
```bash
systemctl status k3s
journalctl -u k3s -e --no-pager
```
If it's crash-looping, check `cat /proc/cgroups` first (Step 5) — an unenabled memory cgroup is the single most common cause of K3s refusing to start cleanly on a Pi. If it's running but the node still shows `NotReady`, use the node-condition check below to see the specific reason.

### Can't reach port 6443

**RUN ON WORKER**
```bash
nc -zv <MASTER_IP> 6443
```
- `Connection refused` → nothing is listening yet; check that `k3s` is actually running on the master (`systemctl status k3s`).
- Times out with no response → a network-layer problem: check the physical cable/switch port, confirm both Pis show the IPs you expect (`hostname -I`), and confirm there's no firewall in the way (`sudo ufw status` — should say inactive unless you specifically enabled it).

### Node shows NotReady

```bash
sudo kubectl describe node <node-name>
```
Scroll to the **Conditions** section near the bottom — it names the specific reason: disk pressure, memory pressure, or the network plugin not having initialized (`NetworkReady: false` most often points back to a Flannel/CNI issue, which in turn often traces back to the NetworkManager fix in Step 4 not having taken effect before the interfaces were created — re-check `NetworkManager --print-config | grep unmanaged-devices`).

### Pods stuck Pending

```bash
sudo kubectl describe pod <pod-name>
```
Read the **Events** section at the very bottom — it states the reason in plain language: insufficient CPU/memory on any node to satisfy a request, a node selector or taint that no node matches, or (if the pod uses a PVC) a claim that hasn't bound yet.

### Image won't pull

```bash
sudo kubectl describe pod <pod-name>
```
Look for `ImagePullBackOff` or `ErrImagePull` in the Events, with a reason underneath: a typo'd image name/tag, no working DNS/internet on the node (test with `curl -I https://registry-1.docker.io` from the node itself), registry rate-limiting, or — specific to ARM hardware — an image that's genuinely only published for `amd64`. Most popular images (nginx, busybox, alpine, postgres, etc.) publish proper multi-arch manifests and just work, but some older or niche images don't. If the pod actually starts but crashes instantly with `exec format error` in its logs, that's the architecture-mismatch signature specifically (see the last entry in this section).

### DNS doesn't work

**Confirm CoreDNS itself is healthy:**
```bash
sudo kubectl get pods -n kube-system -l k8s-app=kube-dns
```
Should show `Running`. If it's not, check its logs: `sudo kubectl logs -n kube-system -l k8s-app=kube-dns`.

**Test resolution from inside the cluster:**
```bash
sudo kubectl run dns-test --rm -it --image=busybox --restart=Never -- nslookup kubernetes.default
```
This should resolve to the cluster's internal API service IP. If it hangs or fails, and CoreDNS itself looks healthy, double check whether the master was recently rebooted (Step 19) — DNS can be briefly unreachable cluster-wide if CoreDNS's pod landed on the master and the master is the one currently down.

### Permission problems with kubectl

Error messages like `Unable to read /etc/rancher/k3s/k3s.yaml: permission denied` or `the connection to the server was refused` almost always mean one of: you dropped `sudo` before completing Step 15's kubeconfig copy-out, your `KUBECONFIG` environment variable isn't set the way Step 15 configured it (`echo $KUBECONFIG` to check), or — if you're on your laptop — the `server:` line in your copied kubeconfig still says `127.0.0.1` instead of `<MASTER_IP>`.

### Time/date problems

Kubernetes' TLS handshakes (and, in an HA setup, etcd itself) depend on reasonably synchronized clocks between nodes — a large clock drift can cause certificate validation failures or node-join errors that look unrelated to time at first glance.
```bash
timedatectl status
```
Confirm `System clock synchronized: yes` on both Pis. A Raspberry Pi has no battery-backed real-time clock, so on a cold boot with no network yet, the clock can briefly start from a stale saved time — this normally self-corrects within seconds once `systemd-timesyncd` reaches an NTP server, but if a Pi has been powered off for a long time and boots without network access, you may see transient certificate-adjacent errors that clear up on their own once it gets online.

### Raspberry Pi architecture problems

```bash
uname -m
```
`aarch64` = 64-bit ARM (what this whole guide assumes). `armv7l` = 32-bit ARM. `x86_64` = you're not actually on a Pi (helpful if you're troubleshooting a mixed-hardware cluster). Container images must either be genuinely multi-architecture or match your node's architecture exactly. The unambiguous symptom of an architecture mismatch is a pod that reaches `Running` briefly (or doesn't even do that) and then crashes with `exec format error` visible in `sudo kubectl logs <pod-name>` — that's not a configuration problem to debug further, it's a straightforward sign the image itself was built for the wrong CPU architecture, and the fix is finding/building an arm64 image instead.
## Step 22 — Cheat sheet

Run all of these on the master (or from your laptop once Step 15's remote access is set up); drop `sudo` if you completed the kubeconfig copy-out in Step 15.

```bash
sudo kubectl get nodes                    # list nodes and their Ready status
sudo kubectl get nodes -o wide             # same, plus IPs, OS, kernel, container runtime
sudo kubectl get pods -A                   # every pod in every namespace
sudo kubectl get pods -A -o wide           # same, plus which node each pod is on
sudo kubectl get svc -A                    # every Service in every namespace
sudo kubectl get deployments -A            # every Deployment in every namespace
sudo kubectl describe node <name>          # detailed status + Conditions for one node
sudo kubectl describe pod <name>           # detailed status + Events for one pod
sudo kubectl logs <pod-name>               # a pod's container logs
sudo kubectl top nodes                     # live CPU/memory usage per node (needs metrics-server)
sudo systemctl status k3s                  # server service status (master only)
sudo systemctl status k3s-agent            # agent service status (worker only)
sudo journalctl -u k3s -e --no-pager       # server logs, most recent first (master only)
sudo journalctl -u k3s-agent -e --no-pager # agent logs, most recent first (worker only)
```

---

## Final checklist

- [ ] `sudo kubectl get nodes` shows **both** `k3s-master` and `k3s-worker` as `Ready`
- [ ] `systemctl status k3s` is `active (running)` on the master; `systemctl status k3s-agent` is `active (running)` on the worker
- [ ] `sudo kubectl get pods -A` shows CoreDNS, Traefik, ServiceLB (`svclb-*`), local-path-provisioner, and metrics-server all `Running` in `kube-system`
- [ ] The Step 13 nginx test was reachable from your laptop via `curl` on the NodePort, through **both** Pis' IPs
- [ ] `cat /proc/cgroups` shows `memory` enabled (`1`) on both Pis
- [ ] `swapon --show` prints nothing on both Pis
- [ ] Both Pis survived a reboot and rejoined automatically (Step 19)
- [ ] The node token from Step 10 is saved somewhere safe (a password manager, not a plain text file) for when you add more Pis later
- [ ] (Optional) `kubectl` works without `sudo` and without the `k3s` prefix

**The one command that proves it all at a glance:**
```bash
sudo kubectl get nodes -o wide && sudo kubectl get pods -A
```
If that shows two `Ready` nodes and every system pod `Running`, your cluster is genuinely up and doing its job.

---

## Sources referenced

- [K3s Quick-Start Guide](https://docs.k3s.io/quick-start)
- [K3s Requirements](https://docs.k3s.io/installation/requirements)
- [K3s Networking Requirements](https://docs.k3s.io/installation/requirements#networking)
- [K3s Known Issues](https://docs.k3s.io/known-issues)
- [K3s Cluster Datastore](https://docs.k3s.io/datastore)
- [K3s High Availability with Embedded etcd](https://docs.k3s.io/datastore/ha-embedded)
- [K3s Token management (`k3s token`)](https://docs.k3s.io/cli/token)
- [Kubernetes: Swap memory management](https://kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/)
- [Raspberry Pi OS now based on Debian 13 "Trixie"](https://www.cnx-software.com/2025/10/06/raspberry-pi-os-debian-13-trixie/)
- [Helm: Installing Helm](https://helm.sh/docs/intro/install/)
