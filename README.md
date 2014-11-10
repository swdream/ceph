Cài  đặt mô hình CEPH đơn giản
===============================
Mục đích của bài viết là note lại các bước và các vấn đề cần chú ý khi cài một mô hình ceph đơn giản.
Mô hình bao gồm một ceph mon và hai ceph OSD.
===
### 1. Thành phần của mô hình.
<img class="image__pic js-image-pic" src="http://prntscr.com/54vcwt" alt="" id="screenshot-image">
##### 1.1. **Ceph-mon.**:
- Replicate Network: 10.10.28.18
- Disk:
    - /dev/vdb
    - /dev/vdc

##### 1.2. **Ceph-osd1.**:
- Replicate Network: 10.10.28.19
- Disk:
    - /dev/vdb
    - /dev/vdc

##### 1.3. **Ceph-osd2.**:
- Replicate Network: 10.10.28.20
- Disk:
    - /dev/vdb
    - /dev/vdc

### 2. Vai trò của từng node trong mô hình.
##### 2.1. Ceph-mon.
Ceph-mon là viết tắt của Ceph monitor.
Monitor (MON) là một lightweight Daemon xử lý tất cả các mối liên quan
giữa các external app và các clients.
Ceph mon quản lý các maps khác nhau trong hệ thống như CRUSH map, MON map...
CRUSH được các client sử dụng trong việc select các OSDs để lưu trữ các bản
coppy của data một cách. Việc select các OSDs này là bất kì, không xác định
trước. Ceph mon quản lý vị trí lưu trữ nơi các dữ liệu cần được lưu trữ và
duy trì đồng bộ chúng với các ceph OSDs nodes chứa chúng.

##### 2.2. Ceph-osd:
The storage deamon, là nơi lưu trữ các data, xử lý data replication recovery,
backfilling, rebalacing, cug cấp dữ liệu monitor....
Bạn sẽ phải chạy deamon này trên tất cả các server trong hệ thống của bạn.
Với mỗi OSD, bạn có thể có các hard drive disks tương ứng. Với mục đích
perfomance, tốt nhất là poll các hard drive disks  thành raid arrays,
LVM hoặc btrfs polling.
Chính vì thế, mỗi server của bạn sẽ có một daemond running. Theo mặc định,
có 3 pools sẽ được tạo ra: Data, metadata và RBD.
- pool(n): chứa các object, giống container trong swift.
- pool(v): group các disks.
Một Ceph Storage Cluster yêu cầu ít nhất hai OSD để hình thành
Active+clean state khi cluster tạo nên hai bản data copy.
Theo default, ceph OSD sẽ coppy data thành 2 bản, đặt trên 2 osd khác
nhau => hình thành một Active + clean state.

##### 2.3. Ceph-mds:
Meta-data server (MDS) là nơi chứa metadata. Các MDSs build các POSIX file
system trên top của các object cho các ceph clients. Tuy nhiên, nếu bạn không
sử dụng CEPH File System thì bạn không cần một MDS.

### 3. Install:
Cài đặt Ceph có hai cách chính:
- cài qua một Ceph-deploy.
- Cài manual.
Trong bài viết này, tôi cài manual hệ thống. Ceph-mon được cài đặt trên Ceph-mon,
hai ceph osd caì trên ceph-osd1, ceph-osd2
===
Trên tất cả các nodes:
```
wget -q -O- 'https://ceph.com/git/?p=ceph.git;a=blob_plain;f=keys/release.asc' | apt-key add -
apt-add-repository 'deb http://ceph.com/debian-firefly/ trusty main'
apt-get update && apt-get install ceph ceph-mds -y
```
Network:
Thêm vào file /etc/network/hosts:
```
10.10.28.18 ceph-mon
10.10.28.19 ceph-osd1
10.10.28.20 ceph-osd2
```
##### 1. Trên ceph-mon:
Fsid là UUID, được xác định như sau:
```
uuidgen
```
example:
```
uuidgen
30ca40b1-028d-47f4-8b1e-aa9844880c13
```
Tạo configuration file /etc/ceph/ceph.conf.
```
[global]
fsid = 30ca40b1-028d-47f4-8b1e-aa9844880c13
public_network = 10.10.28.18
mon initial members = node1, node2, node3
mon host = 10.10.28.18, 10.2.28.19, 10.2.28.20
auth cluster required = cephx
auth service required = cephx
auth client required = cephx
osd journal size = 1024
filestore xattr use omap = true
; Sets the number of replicas for objects in the pool
osd pool default size = 2
osd pool default min size = 1
;The default number of placement groups for a pool
osd pool default pg num = 128
;The default number of placement groups for placement for a pool
osd pool default pgp num = 128
osd crush chooseleaf type = 1
```
Create a keyring for your cluster and generate a monitor secret key.
```
ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
```
Generate an administrator keyring, generate a client.admin user and add the user to the keyring.
```
ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --set-uid=0 --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow'
ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
```
Generate a monitor map.
```
monmaptool --create --add node1 10.10.28.18 --add node2 10.2.28.19 --add node3 10.2.28.20 --fsid 30ca40b1-028d-47f4-8b1e-aa9844880c13 /tmp/monmap
```
Distributed config & keyring file to other nodes.
```
for file in /etc/ceph/ceph.conf /tmp/ceph.mon.keyring /etc/ceph/ceph.client.admin.keyring /tmp/monmap; do \
for node in ceph-osd1 ceph-osd2; do scp $file root@$node:$file; done; \
done;
```
##### 2. Thực hiện trên tất cả các nodes.
Create a default data directory and populate monitor data.
##### 2.1. Trên ceph-mon.
```
mkdir /var/lib/ceph/mon/ceph-mon
ceph-mon --mkfs -i ceph-mon --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
```
Start monitor on ceph-mon.
```
touch /var/lib/ceph/mon/ceph-mon/sysvinit
/etc/init.d/ceph start mon.ceph-mon
```
Sau đó chúng ta làm tương tự trên các nodes khác.
**Notes:**Cần điền đúng tên của các nodes trong các dòng lệnh.

##### 2. Verify.
```
ceph osd lspools
    0 data,1 metadata,2 rbd,
ceph -s
```

