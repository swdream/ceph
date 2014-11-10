Cài  đặt mô hình CEPH đơn giản
===============================
Mục đích của bài viết là note lại các bước và các vấn đề cần chú ý khi cài một mô hình ceph đơn giản.
Mô hình bao gồm một ceph mon và hai ceph OSD.
===
### 1. Thành phần của mô hình.
<img class="image__pic js-image-pic" src="http://prntscr.com/54vcwt" alt="" id="screenshot-image">
##### 1.1. **Ceph-mon.**:
- Replicate Network: 10.2.28.18
- Disk:
    - /dev/vdb
    - /dev/vdc

##### 1.2. **Ceph-osd1.**:
- Replicate Network: 10.2.28.19
- Disk:
    - /dev/vdb
    - /dev/vdc

##### 1.3. **Ceph-osd2.**:
- Replicate Network: 10.2.28.20
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
Trong bài viết này, tôi cài manual hệ thống.
===
Trên tất cả các nodes:
```
wget -q -O- 'https://ceph.com/git/?p=ceph.git;a=blob_plain;f=keys/release.asc' | apt-key add -
apt-add-repository 'deb http://ceph.com/debian-firefly/ trusty main'
apt-get update && apt-get install ceph ceph-mds -y
```

