---
keywords :
 - "vxlan"
 - "virtualization"
 - "가상화"
date : "2016-10-17T14:05:21+09:00"
title : "Network Overlay"
categories :
  - "cloud"
tags:
  - "vlan"
  - "network"
  - "network overay"
image: ""
---

# VXLAN
VXLAN의 'X'의 의미는 **eXtensible**을 뜻하며, L2 Network의 Scale(확장성)을 의미한다. VLAN기반 네트워크보다 더욱 많은 Layer 2 Segment를 구성할수 있으며, VLAN숫자의 제약(*4000개*)을 극복할 수 있음을 의미함

## Network Overlay
Network Overlay 이야기를 여기저기에서 많이 들리고는 있지만, 실제로 이것이 무엇인지에 대해선 잘 알수가 없어서 정리해 본다. 간단하게 추상화 레이어라고 하면 될까? 기존 데이터센터 네트워크망이 어떻게 구성이 되어 있던지 Network를 넘어 구성하겠다는 것이다. 요즘 나오는 SDN(Software Defined Network)와 같은 개념이지 않나 싶다.

- Network Overlay 기술
	- MAC Over IP/UDP 기술 기반

	MAC Over IP/UDP : 기존 Ethernet Frame의 한계를 극복할 수 있는 것은 VXLAN이라는 MAC Over ID/UDP Header가 추가로 생성되어 Encapsulation을 할 수 있기 때문이다.

Cisco/VMWare에 의해 IEFT에 VXLAN으로 표준 Draft됨
현재 CISCO, VMWare, Citrix, Redhat, Broadcom, Arista 등이 지지

## VXLAN
VXLAN Header + UDP + IP 기반으로 전송,

VXLAN Header는 24bit VXLAN Network ID로 구성됨

학습 하지 못한 목적지 VM MAC은 VXLAN기술 기반으로 IP Multicast를 이용하여 목적지 VM MAC주소가 있는 스위치에 전송 (Broadcast, Multicast, Unknown Destination)

VM MAC은 VXLAN기반의 Encapsulation된 패킷을 받아서 목적지 VM ARP Table을 학습하는 방식
학습한 목적지 VM MAC은 P2P Tunnel로 직접해당 스위치와 통신


### VXLAN의 등장배경
- 가상화 환경에서의 네트워크 스위치들의 MAC Address Table의 한계
	- 기본적으로 서버는 과거와 달리 가상화 기반으로 구성
	- 가상화 기반의 서버는 다량의 MAC Address Table을 보유
	- ToR(Top Of Rack), Access Switch 등에서 최대 수용할 수 있는 MAC의 한계

	일반서버들의 NIC & MAC address 숫자 (NIC 1개, Mac Address 2개)

	가상화 환경에서의 NIC & MAC address 숫자
		- VMWare ESXi 서버 기준으로 vNIC위한 AMC은 3개 정도 필요 이중화를 고려할때 6개의 nic가 필요

	단일 STP도메인당 MAC Address Table이 100k를 넘을 수 있음.
	네트워크 장비의 한계, MAC Table관리의 문제 발생

	--> 따라서 네트워크스위치에서 MAC Address Table을 관리하지 않고, 하단의 가상화 vSwitch에서만 소유하고, 해당 테이블을 통해서 Fowarding 하는 방식이다.


- 가상화 환경에서의 VLAN 숫자의 한계
	- Multi-Tenancy 환경에서 각 Tenancy 또는 각 VDC(Virtual DC)별로 별도의 VLAN 할당
	- 16bit VLAN ID에서 최대 수용 가능한 VLAN숫자의 한계 (약4000개)

	일반적인 VLAN은 12bit ID = 4,000 Vlan : Ethernet Frame Format이 12bit VLAN ID를 제공하기 때문
	VDC가 많아 질수록 Vlan숫자 부족하게 됨

	16bit VLAN = 4096개만을 제공하고, 여기에 Reserve되 VLAN을 제외하면 4000개 정도로 보면 됨\

	VXLAN은 VLAN ID가 26bit로 무려 16,000,000 개의 VLAN을 생성할 수 있다.


- 가상화 환경에서의 이동성에 대한 고민
	- Static한 VLAN Trunk환경에서는 결국 VM이동성의 한계가 있음.

	VXLAN은 VLAN Trnk가 필요없다.
	Multicast기반으로 알아서 Tree를 구성한다.
	훨씬 더 Dynamic하고 능동적인 구성이 가능하다.

### VXLAN의 등장배경 - VXLAN이 바꿔놓은 가상화 환경
- 가상화 환경에서의 MAC Address Table의 한계 극복
	- 일반적인 가상화 스위치의 Bridging 기술 적용
	- 학습하지 못한 ARP요청에 대해서만 Multicast 기반의 Broadcast/Multicast등을 처리
	- STP단일 도메인에서 필요한 MAC Address는 vmkernel port숫자만 필요

- 가상화 환경에서의 VLAN숫자의 한계
	- VXLAN의 VLAN IDbit는 24bit로 최대 16,000,000개의 VLAN생성 가능
	- 다양한 VDC, Multi-Tenant환경 구축 가능

- 가상화 환경에서의 이동성에 대한 고민
	- IGMP기반의 자동화된 prune traffic을 통해, ARP요청 시에 자동으로 갱신하는 방식


### VXLAN Network 구조

#### VTEP (VXLAN Tunnel End Point)





참고 : [청년정신](http://youngmind.tistory.com/entry/Network-Overlay-VXLAN-%EB%B6%84%EC%84%9D-1)
