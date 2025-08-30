
### **วิธีตั้งค่า MikroTik CHR และ Proxmox เพื่อใช้งาน DHCP และ NAT**

บทความนี้จะแนะนำขั้นตอนการตั้งค่า **MikroTik CHR** และ **Proxmox** เพื่อให้เครื่อง Virtual Machine (VM) หรือ Container (CT) สามารถรับ IP อัตโนมัติและเข้าถึงอินเทอร์เน็ตผ่าน MikroTik ได้

----------

### **1️⃣ ฝั่ง Proxmox**

ขั้นตอนนี้เป็นการสร้าง Network Bridge ใหม่เพื่อใช้เป็นวง LAN สำหรับ VM/CT โดยเฉพาะ

1.  **สร้าง Bridge สำหรับ LAN ใหม่**
    
    -   ไปที่ `Datacenter` → `Node` → `Network` → `Create` → `Linux Bridge`
        
    -   **ตั้งชื่อ:** เช่น `vmbr1`
        
    -   **Bridge ports:** ปล่อยว่าง (none) ซึ่งหมายถึงจะต่อ VM/CT โดยตรง
        
    -   **VLAN aware:** เลือก `No` (ถ้าไม่ได้ใช้ VLAN แยก)
        
    -   กด `Apply` และทำการ `reboot network` ถ้าจำเป็น
        
2.  **ผูก VM/CT เข้ากับ Bridge**
    
    -   เลือก VM หรือ CT ของคุณ จากนั้นไปที่แท็บ `Hardware` → `Network Device`
        
    -   **Bridge:** เลือก `vmbr1` ที่เพิ่งสร้างขึ้น
        
    -   **Model:** เลือก `VirtIO` (แนะนำ)
        

----------

### **2️⃣ ฝั่ง MikroTik CHR**

ในส่วนนี้เป็นการตั้งค่าภายใน MikroTik CHR เพื่อทำหน้าที่เป็น DHCP Server และ Firewall NAT

1.  **ตั้งค่า IP LAN ใหม่**
    
    รันคำสั่งนี้เพื่อกำหนด IP Address ของฝั่ง LAN ให้กับ `ether2` ซึ่งเป็น Interface ที่เชื่อมต่อกับ VM/CT
    
    ```
    /ip address add address=192.168.10.1/24 interface=ether2
    
    ```
    
2.  **สร้าง DHCP Pool**
    
    สร้างช่วง IP Address ที่จะแจกจ่ายให้กับอุปกรณ์ในวง LAN
    
    ```
    /ip pool add name=dhcp_pool1 ranges=192.168.10.2-192.168.10.254
    
    ```
    
3.  **สร้าง DHCP Server**
    
    สร้าง DHCP Server โดยระบุ Interface และ Pool ที่ต้องการใช้งาน
    
    ```
    /ip dhcp-server add name=dhcp1 interface=ether2 address-pool=dhcp_pool1 lease-time=30m
    
    ```
    
4.  **ตั้งค่า DHCP Network**
    
    กำหนด Gateway, DNS และ Network Address สำหรับ DHCP Server
    
    ```
    /ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=8.8.8.8,1.1.1.1
    
    ```
    
5.  **เปิดใช้งาน DHCP Server**
    
    เปิดการทำงานของ DHCP Server ที่เพิ่งสร้าง
    
    ```
    /ip dhcp-server enable dhcp1
    
    ```
    
6.  **ตั้งค่า NAT ออก WAN**
    
    กำหนดกฎ NAT เพื่อให้อุปกรณ์ในวง LAN (eth2) สามารถเข้าถึงอินเทอร์เน็ตผ่าน WAN (eth1) ได้
    
    ```
    /ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
    
    ```
    

----------

### **3️⃣ การตรวจสอบ**

เมื่อตั้งค่าเสร็จสิ้น คุณสามารถตรวจสอบสถานะและผลลัพธ์ได้ดังนี้

-   **เช็ก DHCP Server:** รันคำสั่ง `/ip dhcp-server print` เพื่อดูว่า Server ทำงานอยู่หรือไม่ (running=yes)
    
-   **เช็ก IP ของ VM/CT:** ในเครื่อง VM/CT ให้รันคำสั่ง `ip a` เพื่อดูว่าได้รับ IP ในช่วง **192.168.10.2–254** หรือไม่
    
-   **ทดสอบการเชื่อมต่ออินเทอร์เน็ต:** ทดสอบการ Ping ไปยัง Gateway, Internet และ DNS
    
    Bash
    
    ```
    ping 192.168.10.1   # gateway
    ping 8.8.8.8        # Internet
    ping google.com     # DNS
    
    ```
    

----------

### **4️⃣ (ถ้ามี) การตรวจสอบ VLAN**

หากไม่ได้ใช้งาน VLAN บน `ether1` หรือ `ether2` คุณสามารถตรวจสอบและลบได้โดยใช้คำสั่ง

-   **เช็ก VLAN:** `/interface vlan print`
    
-   **ลบ VLAN (ถ้าไม่ใช้งาน):** `/interface vlan remove vlan10`
    

----------

### **✅ ผลลัพธ์สุดท้าย**

-   VM/CT ได้รับ IP จาก **MikroTik DHCP**
    
-   สามารถออกอินเทอร์เน็ตผ่าน WAN ของ **Tenda** (หรืออุปกรณ์อื่นที่เชื่อมต่ออยู่)
    
-   วง LAN ใหม่ **(192.168.10.0/24)** ถูกแยกออกจากวง WAN **(192.168.0.0/24)**
    
-   DHCP, NAT, และ DNS พร้อมใช้งานอย่างสมบูรณ์
