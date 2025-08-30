


1️⃣ ฝั่ง Proxmox

สร้าง bridge สำหรับ LAN ใหม่

ไปที่ Datacenter → Node → Network → Create → Linux Bridge

ตั้งชื่อ เช่น vmbr1

Bridge ports: ว่าง (none) → จะต่อ VM/CT โดยตรง

VLAN aware: เลือก No (ถ้าไม่ใช้ VLAN แยก)

Apply และ reboot network ถ้าจำเป็น

ผูก VM/CT เข้ากับ bridge

VM/CT → Hardware → Network Device

Bridge → เลือก vmbr1

Model → VirtIO (แนะนำ)

2️⃣ ฝั่ง MikroTik CHR

ตั้ง IP LAN ใหม่

/ip address add address=192.168.10.1/24 interface=ether2


ether2 → LAN ของ VM/CT

สร้าง DHCP Pool

/ip pool add name=dhcp_pool1 ranges=192.168.10.2-192.168.10.254


สร้าง DHCP Server

/ip dhcp-server add name=dhcp1 interface=ether2 address-pool=dhcp_pool1 lease-time=30m


ตั้ง DHCP Network

/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=8.8.8.8,1.1.1.1


เปิด DHCP Server

/ip dhcp-server enable dhcp1


ตั้ง NAT ออก WAN

/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade

3️⃣ ตรวจสอบ

เช็ก DHCP Server

/ip dhcp-server print


ต้องเห็น running=yes

เช็ก IP ของ VM/CT

ip a


ต้องได้ IP ในช่วง 192.168.10.2–254

ทดสอบ Internet

ping 192.168.10.1  # gateway
ping 8.8.8.8       # Internet
ping google.com    # DNS

4️⃣ (ถ้ามี) เช็ก VLAN
/interface vlan print


ถ้าไม่ได้ใช้งาน VLAN บน ether1/ether2 → สามารถลบได้

/interface vlan remove vlan10

✅ ผลลัพธ์สุดท้าย

VM/CT ได้ IP จาก MikroTik DHCP

ออก Internet ผ่าน WAN ของ Tenda

วง LAN ใหม่ (192.168.10.0/24) แยกจาก WAN 192.168.0.0/24

DHCP, NAT, DNS พร้อมใช้งาน

ถ้าคุณต้องการ ผมสามารถทำ version diagram + table ให้ดู ภาพรวมทั้งหมด จะเห็น Proxmox ↔ MikroTik ↔ Tenda ชัดเจนครับ.

คุณอยากให้ผมวาดให้ไหม?
