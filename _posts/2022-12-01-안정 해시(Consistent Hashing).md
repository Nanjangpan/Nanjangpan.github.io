---
title: 안정 해시(Consistent Hashing)
categories: [System]
tags: [consistent_hashing, 가상-면접-사례로-배우는-대규모-시스템-설계-기초]     # TAG names should always be lowercase
---


# 안정 해시(Consistent Hashing)

[가상 면접 사례로 배우는 대규모 시스템 설계 기초](https://product.kyobobook.co.kr/detail/S000001033116)  책에서 읽은 안정 해시 내용을 정리한 글입니다.


## 1. 해시 키 재배치(rehash) 문제

N개의 캐시 서버가 있을 때 부하를 균등하게 나누는 보편적인 방법은 아래와 같습니다. (N은 서버 개수)
$$
serverIndex = hash(key) % N
$$


하지만 해당 방법은 **서버의 개수가 달라질 때 대부분의 키를 재분배해야 합니다. 이 과정에서 대규모 캐시 miss가 발생하게 됩니다.**

앞선 문제를 해결하는 방법이 **안정 해시(Consistent Hashing)** 입니다.

## 2. 안정 해시의 구현

안정 해시는 해시 테이블 크기가 조정 될 때 평균적으로 오직 k/n개의 키만 재배치하는 해시 기술입니다.
(k: 키의 개수, n: 슬롯의 개수)

이와는 달리 대부분의 전통적 해시 테이블은 슬롯의 수가 바뀌면 거의 대부분 키를 재배치합니다.

### 2.1 기본 구현법
![Untitled](https://i.ibb.co/xsKWpdr/Untitled.png)
> 이미지 출처 :  [Consistent Hashing(일관된 해싱)](https://uiandwe.tistory.com/1325)

서버와 키를 균등 분포 해시 함수를 사용해 해시 링에 배치합니다. 
키의 위치에서 링을 시계 방향으로 탐색하다 만나는 최초의 서버가 키가 저장될 서버입니다.
    
#### 기본 구현법의 문제점
    
  1. 서버가 추가되거나 삭제되는 상항을 감안하면 파티션의 크기를 균등하게 유지하는게 불가능합니다.
  
   2. 키의 균등 분포를 달성하기가 어려움이 있습니다.
   
   3. 이를 해결하기 위해 **가상 노드**(virtual node) 또는 **복제**(replica)라 불리는 기법을 사용합니다.


### 2.2 가상노드
![](https://miro.medium.com/max/926/1*3sNlBRN-yTOWMdzDCjrQ4Q.png)

> 이미지 출처 :  [How to Use Consistent Hashing in a System Design Interview?](https://medium.com/interviewnoodle/how-to-use-consistent-hashing-in-a-system-design-interview-b738be3a1ae3)
- 가상 노드는 실제 노드 또는 서버를 가르키는 노드로서, 하나의 서버는 링 위에 여러개의 가상 노드를 가질 수 있습니다.

- **가상 노드를 늘릴면 키의 분포는 표준 편차가 작아져서 데이터가 고르게 분포 되어 점점 더 균등해집니다.** 그러나 개수를 많이 늘리면 공간을 많이 차지하는 문제점이 있으니 적절한 타협점을 찾아야 합니다.


## 3. 안정해시의 이점
1. 서버가 추가되거나 삭제 될 때 재배치되는 키의 수가 최소화 됩니다.

2. 데이터가 보다 균등하게 분포하게 되므로 수평적 규모 확장성을 달성하기 쉽습니다.

3. 핫스팟 키 문제를 줄일 수 있습니다. (안정해시는 데이터를 좀 더 균등하게 분배하기 때문입니다)