---
title: "blkid를 활용하여 Mount시 Volume 고정하기"
description:
date: 2024-07-28
update: 2021-07-28
tags:
  - Linux
---
AWS에서 인스턴스에 EBS를 여러 개 연결하면 Mount 시 EBS의 파일 크기가 작은 순으로 할당되는 것을 막기 위해, blkid 명령어로 EBS의 UUID를 얻어 Mount 시 EBS 연결을 UUID를 통해 할당되게 만들 수 있습니다.

/etc/fstab 파일에 마운트할 디스크나 블록 디바이스의 파일 시스템 명( /dev/sda1 등)을 적는 것보다 UUID를 적는 것이 더 유리합니다.

# blkid란?
blkid는 디스크 블록 장치의 파일 시스템 종류와 함께 파일 시스템의 UUID 값을 출력합니다. 이 UUID 값은 이후 파일 시스템을 시스템에 자동 마운트하는 과정에서 사용됩니다.

## 파일 시스템 생성
아래 명령어를 사용하기 위해서는 파일 시스템을 디스크 블록에 먼저 생성해야 합니다.

```shell
mkfs.xfs -f /dev/sda1
```
## /etc/fstab 파일에 UUID 추가하기
예를 들어, /dev/sdc1을 /mnt/my에 마운트하려면 /etc/fstab 파일에 다음과 같이 UUID를 추가합니다.

```shell
UUID=b1c7c897-be87-4487-9e81-bc6d2d146a0e /mnt/my ext4 defaults 0 2
```
UUID=b1c7c897-be87-4487-9e81-bc6d2d146a0e: 마운트할 디스크의 UUID.

/mnt/my: 마운트할 디렉토리.

ext4: 파일 시스템 타입.

defaults: 마운트 옵션. 여기서는 기본 옵션을 사용.

0: 덤프(dump) 유틸리티로 파일 시스템을 백업할지를 결정. 0은 백업하지 않음을 의미.

2: 부팅 시 파일 시스템 검사의 순서. 0은 파일 시스템 검사를 하지 않음을 의미하며, 1은 루트 파일 시스템을 검사, 2는 루트 파일 시스템 외의 파일 시스템을 검사합니다.