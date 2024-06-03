---
layout:     post
title:      "스냅샷 플러그인 도입기(1)"
date:       2024-06-03
author:     이헌제 (lhj4125@gluesys.com)
categories: blog
tags:       [스냅샷, 스토리지, LVM, Thin Provisioning, 장애, Storage]
cover:      "/assets/Snapshot1intro_maincover.jpg"
main:       "/assets/Snapshot1intro_maincover.jpg"
---

장애 발생이나 유실된 데이터 복원을 위해서 특정 시점에 볼륨의 파일 시스템을 포착하여 별도로 보관하기 위해서 기존 AnyStorE 제품에서 스냅샷 기능을 사용하고 있었습니다. 스냅샷 생성 만을 위한 페이지로 구성되어 있어 확인은 가능했지만, 실제로 복원 등에 사용하기 위해서는 관리자를 통해서 데이터를 복원할 수 밖에 없었습니다. 실제 복원 시에는 고객 서비스에 영향이 없도록 진행되어야 했기에 도입하기 까다로웠던 점도 있었습니다. 


스냅샷 플러그인 구조로 이번에 스냅샷 기능을 다시 추가하게 되었는데, 스냅샷을 안정적으로 도입하기 위해서 어떤 테스트를 거치고 도입을 하게 되었는지 살펴보겠습니다. 


### 배경 설명 

본 제품에서는 LVM 볼륨을 주로 다룹니다. LVM 볼륨의 프로비저닝 정책에 따라서 스냅샷의 종류도 나뉘게 되는데, 우선 볼륨의 프로비저닝 종류에 대해서 간략히 설명해 보겠습니다.

* **씬 프로비저닝(Thin Provisioning)된 볼륨** : 사용자에게 필요한 만큼의 스토리지 공간을 할당하되, 실제로는 사용한 만큼의 공간만 실제 스토리지에서 차지하는 방식입니다. 볼륨 크기를 미리 정해두지 않으므로, 데이터가 증가하면 볼륨 크기를 쉽게 조정할 수 있습니다.

* **씩 프로비저닝(Thick Provisioning)된 볼륨** : 미리 정해진 크기의 스토리지 공간을 사용자에게 할당하는 방식입니다. 이 공간은 사용자가 실제로 사용하지 않더라도 할당된 크기만큼 스토리지에서 차지합니다. 스토리지 공간을 확보하는 데 있어서 보다 안정적이지만, 사용하지 않는 공간이 발생할 수 있어 스토리지 공간의 사용이 비효율적일 수 있습니다.

프로비저닝 된 볼륨에 따라서 스냅샷의 저장 방식도 바뀝니다. 

* **씬 프로비저닝된 볼륨의 스냅샷**: 원본 볼륨의 데이터를 공유하며, 자체 메타데이터 영역을 가지고 있습니다. 메타데이터 영역에는 스냅샷의 논리적 익스텐트와 물리적 익스텐트 간의 매핑이 저장되어 있습니다. 이를 통해 스냅샷은 원본 볼륨의 상태를 보존하고, 변경사항을 추적하게 됩니다.

* **씩 프로비저닝된 볼륨의 스냅샷** : 원본 볼륨의 데이터를 복사하여 새로운 공간에 저장합니다. 원본 데이터의 완전한 복사본을 제공하지만, 스토리지 공간을 더 많이 차지하게 됩니다. 

여기까지만 살펴봤을 때 차이점이 여럿 보입니다. 씬 프로비저닝된 볼륨은 저장방식이 다르기 때문에, 스냅샷 또한 저장하는 방법이 다르게 보입니다. 실제로 씬 프로비저닝된 볼륨은 스냅샷을 다음과 같은 구조로 저장합니다. 


&nbsp;


![씬 프로비저닝된 볼륨 구조](/assets/Snapshot1intro_maincover.png){: width="700"}
<center>&#60; 씬 프로비저닝된 볼륨 구조 &#62;</center>

&nbsp;


위 구조를 보게 되면 Thin 볼륨과 스냅샷 볼륨은 공통의 데이터 볼륨을 공유합니다. 그렇기 때문에 많은 스냅샷을 생성할 수 있지만, 스냅샷을 삭제하더라도 여유 공간이 추가적으로 확보되지 않을 수도 있습니다. 데이터 볼륨 말고도 메타데이터 볼륨도 보이는데, 메타데이터 볼륨에서는 각 Thin 볼륨에 속한 데이터 볼륨을 추적합니다. 씬 풀 생성 시 볼륨 그룹 별로 예비 메타데이터(pmshapre) 도 생성되는데, 하나의 볼륨 그룹에서 모든 씬 풀에 관한 예비 메타데이터로 씬 풀 안에서 생성된 모든 씬 풀을 수리하기 위해서 사용됩니다.


### 스냅샷 도입 전 테스트


스냅샷을 도입하기 전에 몇 가지 스냅샷 기능에 관한 테스트를 진행하였습니다. 진행하면서 어떤 점을 고민하면서 넘어갔는지 살펴보겠습니다. 


#### 1. Thin pool 데이터 볼륨 저장 공간 확보 

Thin 볼륨의 파일 시스템에서 파일을 삭제하면 일반적으로 씬 풀에 Free 블록을 다시 추가하지 않습니다. 파일 시스템에서는 삭제되었지만, 씬 풀에 적용되지 않은 경우, `fsrtim` 을 통해서 물리적 공간을 씬 풀에 복원할 수 있습니다. 하지만 매번 파일을 삭제할 때마다 `fstrim` 을 통해서 씬 풀의 데이터 공간에 대한 자동 해제를 일일이 복원해줄 수 없습니다. 볼륨 마운트 시에 `discard` 옵션은 씬 풀의 데이터 공간에 대한 자동 해제 옵션을 다뤄 이러한 점을 해결할 수 있었습니다.


```bash
# Thin Vol 데이터 쌓기
dd if=/dev/zero of=/{mount_path}/fileN bs=4096 count=1000000;

# 용량 확인 
lvs
  LV           VG        Attr       LSize   Pool         Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root         anystor-e -wi-ao---- <44.00g                      
  swap         anystor-e -wi-ao----   5.00g                          
  ThinLV       thinvpool Vwi-aotz--  60.00g tp_thinvpool        58.33  
  tp_thinvpool thinvpool twi---tz--  40.00g                     87.51  3.28 

# 데이터 삭제 후 재 마운트
# mount 시에 `-o discard` 추가
mount -o discard /dev/mapper/thinvpool-ThinLV /{mount_path}

# Thin Vol 데이터 쌓기
dd if=/dev/zero of=/{mount_path}/fileN bs=4096 count=1000000;

# 용량 확인
lvs
  LV           VG        Attr       LSize   Pool         Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root         anystor-e -wi-ao---- <44.00g                                
  swap         anystor-e -wi-ao----   5.00g                                      
  ThinLV       thinvpool Vwi-aotz--  60.00g tp_thinvpool        1.78             
  tp_thinvpool thinvpool twi---tz--  40.00g                     2.67   1.64 
```


#### 2. Thin 데이터 볼륨 공간 부족 

자동 확장을 설정한 경우 데이터 공간이 부족하기 전에 설정 값에 따라서 씬 풀의 데이터 공간이 확장됩니다. 하지만 확장할 공간이 없고 씬 풀의 데이터 공간이 이미 100% 인 경우에도 추가로 디스크를 증설해준다면 이후에 수동으로 확장할 수 있습니다. 만약 자동 확장 시 볼륨의 물리적인 용량이 부족한 경우 확장에 실패하여 데이터 볼륨이 풀 나는 경우가 있는데, 그 경우의 쓰기 동작을 설정할 수 있습니다. 

데이터 볼륨에 여유 공간이 없는 경우에 큐 버퍼에 쓰기 명령을 넣었다가 확장이 완료되면 버퍼를 사용합니다. 기본 타임아웃(60초)이 지난 경우에는, 모든 쓰기 명령이 `error`을 반환하게 됩니다. 씬 풀에서는 `--errorwhenfull` 옵션으로 씬 풀이 가득 찼을 때, 큐 버퍼 사용 유무를 지정할 수 있습니다.  

```bash
# 씬 풀의 데이터 볼륨이 가득찬 경우
lvs
  LV           VG        Attr       LSize   Pool         Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root         anystor-e -wi-ao---- <44.00g          
  swap         anystor-e -wi-ao----   5.00g           
  ThinLV       thinvpool Vwi-aotz--  60.00g tp_thinvpool        66.67   
  tp_thinvpool thinvpool twi---tzD-  40.00g                     100.00 3.53 

# 씬 풀의 errorwhenfull 옵션으로 파일 시스템의 동작 결정
lvchange --errorwhenfull (y/n) thinvpool/tp_thinvpool
```

씬 풀이 가득 찼을 때 파일 시스템의 동작을 결정했다면, 다음으로는 씬 풀의 수동 확장으로 용량을 확장할 수 있습니다. 씬 풀만 확장해준다면, 씬 볼륨은 자동으로 확장됩니다. 

```bash
lvextend -L+{size} thinvpool/tp_thinvpool
```


#### 3. Thin 메타데이터 볼륨 공간 부족 


Thin 메타데이터의 볼륨이 작고, 스냅샷이 많은 경우 메타데이터 볼륨이 부족할 수 있습니다. 만약 Thin 메타데이터의 볼륨이 가득 찬 경우 메타데이터의 볼륨이 손상될 수 있습니다. 

```bash
lvcreate -s -n snap1 vg1/thinvol
...
lvcreate -s -n snap600 vg1/thinvol

lvs                                         
  thinvol       vg1 Vwi-aotz--  10.00g tp_thinpool        82.70       
  tp_thinpool   vg1 twi-aotzM-  10.00g                    84.81  100.00                          
```

메타데이터를 확인하고 복구하는 것은 파일 시스템에서 `fsck/repair` 를 실행하는 것과 유사합니다.  씬 풀의 메타데이터를 복구하는 방법 중 하나는 `lvconvert --repair {vg}/{thinpool}`  를 실행하는 것입니다, 이 동작에서는 pmspare 로 복구 메타데이터를 생성하고, 메타데이터를 교체할 수 있도록 유도합니다. 

단순히 공간이 더 필요한 경우는 메타데이터 확장을 시도해볼 수 있습니다. 

```bash
lvs -a
  thinlv             vg Vwi-aotz--  10.00g thinpool     82.70                
  thinpool           vg twi-aotzM-  10.00g              84.81  100.00        
  [thinpool_tdata]   vg Twi-ao----  10.00g                                   
  [thinpool_tmeta]   vg ewi-ao----  12.00m  

lvextend --poolmetadatasize +{size} vg/thinpool 

lvs -a 
  thinlv             vg Vwi-aotz--  10.00g thinpool     82.70                
  thinpool           vg twi-aotz--  10.00g              84.84  28.04         
  [thinpool_tdata]   vg Twi-ao----  10.00g                                   
  [thinpool_tmeta]   vg ewi-ao----  60.00m   
```


#### 4. 메타데이터 장애 복구

실제 이용 시에는 메타데이터의 용량 풀이나, 디스크 장애 등의 사례에서 메타데이터가 손실되는 경우도 있었습니다. LVM 에서는 각각의 디스크가 동일한 볼륨 풀을 사용하고 있다면, 동일한 메타데이터를 디스크에 복사하여 사용하기 때문에 디스크에 할당된 정보(PV 정보 등)을 제외하고 복사한다면 손쉽게 복구할 수 있습니다. 메타데이터 영역 중 디스크 정보에 해당하는 656 byte 를 제외하고 1024 KB 까지 복사하여 손상된 디스크의 메타데이터를 복구합니다. 


```bash
# 메타데이터 장애 로그
  Metadata location on /dev/sde at 109056 begins with invalid VG name.
  Failed to read metadata summary from /dev/sde
  Failed to scan VG from /dev/sde
  Couldn't find device with uuid 3C36Fc-B9Fc-7LGH-EoAO-0r2c-ECrl-GuWorB.
  Cannot restore Volume Group VGPB with 1 PVs marked as missing.
  Restore failed.

# 임시 파일 복사 (백업용)
dd if=/dev/sde of=/root/temp bs=1020k count=1

# LVM 메타데이터 복사
dd if=/dev/sdd of=/dev/sde bs=1020k count=1 conv=notrunc

# LVM PV Header 만 다시 복사
dd if=/root/temp of=/dev/sde bs=656 count=1 conv=notrunc
```


### 마치며

이 외에도 스냅샷 서비스의 스펙을 결정하기 위해서 많은 고민이 있었고, 또 고도화를 위해서 많은 의견들이 있었습니다. 예를 들어 Samba 의 Shadow copy 를 통한 스냅샷의 Windows 클라이언트 노출, Gluster 볼륨의 스냅샷 제공, PV 의 메타데이터가 너무 작은 경우 스냅샷 생성 실패 사례, 여러 가지 장애 케이스(씬 프로비저닝 볼륨, 파일 시스템, PV, VG, Data LV 등의 각각의 장애 사례)에 대한 위험 관리의 다양한 측면으로 서비스를 더욱더 안정적으로 확장시켜가도록 진행하고 있습니다. 



### 참고
---
* [lvmthin linux manual page](https://man7.org/linux/man-pages/man7/lvmthin.7.html)



