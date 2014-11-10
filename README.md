Cài đặt mô hình CEPH đơn giản
===============================
Mục đích của bài viết là note lại các bước và các vấn đề cần chú ý khi cài một mô hình ceph đơn giản.
Mô hình bao gồm một ceph mon và hai ceph OSD.
===
### 1. Thành phần của mô hình.
- ** Ceph-mon.**:
Replicate Network: 10.2.28.18
Disk:
* /dev/vdb
* /dev/vdc

