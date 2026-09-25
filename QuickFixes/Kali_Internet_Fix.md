# Kali Internet Fix (eth0)

Kali has two NICs:

- **eth0** = internet (VirtualBox NAT). Should be `10.0.2.x`
- **eth1** = lab network. Yours is `192.168.56.103`. Do not change it.

If `ping 8.8.8.8` says `Network is unreachable`, eth0 has no IP.

---

## 1. VirtualBox first (power Kali off)

Kali VM → Settings → Network → **Adapter 1**

- Enable Network = ON
- Attached to = **NAT**
- Advanced → **Cable connected** = ON

Adapter 2 stays **Host-Only** (lab). Start Kali.

---

## 2. See what you have

```bash
ip -4 addr
ip route
ping -c 2 8.8.8.8
```

- eth0 has `10.0.2.x` and ping works → you are done. Skip to step 5.
- eth0 has no `inet` line → keep going.

---

## 3. Permanent NetworkManager profile

```bash
nmcli -f NAME,DEVICE,AUTOCONNECT con show
```

If `nat-eth0` does **not** exist yet:

```bash
sudo nmcli con add type ethernet ifname eth0 con-name nat-eth0 ipv4.method auto
```

Then always run:

```bash
sudo nmcli con modify nat-eth0 \
  connection.interface-name eth0 \
  connection.autoconnect yes \
  connection.autoconnect-priority 100 \
  ipv4.method auto

sudo nmcli con up nat-eth0
```

---

## 4. Prove it

```bash
ip -4 addr show eth0
ip route
ping -c 2 8.8.8.8
```

Good result:

- `inet 10.0.2.15/24` on eth0
- `default via 10.0.2.2 dev eth0`
- ping replies from `8.8.8.8`

---

## 5. Reboot test (this is the permanent check)

```bash
sudo reboot
```

After login:

```bash
ip -4 addr show eth0
ping -c 2 8.8.8.8
```

If eth0 still has `10.0.2.x`, the fix stays after every boot.

---

## Do not do

- Do not run DHCP on `eth1`, `docker0`, or `br-*`
- Do not delete `Wired connection 1` if it is on **eth1**
- Do not set a static IP on eth0 unless NAT DHCP is broken on the host

---

## If it still fails after reboot

VirtualBox is not giving DHCP. On the Windows host: confirm Adapter 1 is NAT + cable connected, restart VirtualBox as Administrator, start Kali, then:

```bash
sudo nmcli con up nat-eth0
ip -4 addr show eth0
```
