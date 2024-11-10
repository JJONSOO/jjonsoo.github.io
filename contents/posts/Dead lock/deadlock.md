---
title: "Deadlock in Oracle with Insert query"
description:
date: 2024-11-10
update: 2021-11-10
tags:
  - Oracle
  - Deadlock
---
이번글은 Oracle Insert 시 발생하는 데드락 문제 분석과 해결 방안에 대해 작성해보았습니다.

## 데드락이 발생한 상황 가정
상품(Product)과 사은품(Gift) 두 개의 테이블이 있다고 가정해 보겠습니다.

여기서 상품 테이블의 기본 키(PK)는 사은품 테이블의 기본 키(PK)이자 외래 키(FK)로 설정되어 있으며, 상품 테이블의 PK는 시퀀스(sequence)를 통해 생성됩니다.

상품 및 사은품 Insert 쿼리는 다음과 같은 흐름으로 이루어집니다.

아래 흐름은 하나의 트랜잭션에서 동작합니다.

1. 상품 sequence의 LAST NUMBER로 return_id를 지정
2. 상품 insert. PK는 sequence의 NEXT_VAL로 지정한다.
3. 1)에서 return_id를 가져와 사은품의 PK로 지정하여 insert.

## 문제 발견

위와 같은 흐름에서는 데이터의 일관성 문제는 있을 수 있지만, 데드락이 발생하지는 않습니다.
데드락은 두 개의 트랜잭션이 서로가 보유한 자원을 기다리며 발생하는 상황을 의미합니다.
위 상황에서 데드락이 발생하지 않는 이유에 대해 알아보겠습니다.

트랜잭션 1에서 return id를 x, 트랜잭션 2에서 return id를 y라고 가정해보겠습니다.
트랜잭션 1에서 상품 insert시 실제 저장되는 PK는 x+A, 트랜잭션 2에서 상품 insert시 실제 저장되는 PK는 y+B입니다. (x>0, y>0, A>=0, B>=0)
트랜잭션 1은 상품 pk가 x+A인 상품을 insert하여 x+A row X lock을 가지고 있고, 트랜잭션 2은 상품 pk가 y+B인 상품을 insert하여 y+B row X lock을 지닌 상태에서 데드락이 발생할 수 있는 상황은 다음과 같습니다.

트랜잭션 1에서 사은품 insert시 pk가 x일 경우 참조성 확인을 위해 트랜잭션2의 상품의 pk인 y+B row를 READ, 트랜잭션 2에서 사은품 insert시 pk가 y일 경우 참조성 확인을 위해 트랜잭션1의 상품의 pk인 x+A row READ하는 경우입니다.

하지만 x = y+B 이며 y = x+A 수식이 성립할 수 없습니다.

따라서, 상품 한개가 아닌 여러개를 저장하고 3단계에서 return id를 for문을 돌려 사은품의 PK를 (return_id+idx)로 저장하는 상황에서는 데드락이 발생합니다.

## 문제 해결
데드락을 방지하려면, 상품 테이블에 저장된 ID 목록을 한 번에 가져와 사은품 테이블에 Insert하는 방법을 사용해야 합니다.

하나의 세션에서만 동작하거나 낮은 동시성에서는 문제가 발생하지 않지만, 여러 세션이 동시에 상품을 Insert하게 되면 데드락 발생 가능성이 커집니다.

## 결론
이번 분석을 통해 Oracle에서도 Insert 과정에서 특정 조건 하에 데드락이 발생할 수 있음을 알게 되었으며, Sequence를 기반으로 PK가 생성되고 여러 종속적 Insert 작업이 있을 경우 데드락 위험이 존재합니다.

이러한 문제를 인지하고, 관련 레코드를 일괄 Insert하는 방식으로 해결할 수 있습니다.