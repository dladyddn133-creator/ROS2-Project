# ROS 2 Multi-Robot Autonomous Exploration & Response System

> 강화학습 기반 자율 탐색, 실시간 위험 지도 생성,
> 다중 로봇 임무 인계와 OMX 자동 조준을 통합한 협업 로봇 시스템

## Demo

전체 탐색 및 타격 영상

## Project Overview

본 프로젝트는 세 대의 TurtleBot3가 역할을 분담하여 자율 탐색부터 위험 표적 대응까지 수행하는 ROS 2 기반 다중 로봇 시스템입니다.

Active Scout가 SAC 강화학습 정책으로 환경을 탐색하면서 SLAM 지도와 위험 지도를 생성하고, Leader Waffle은 공유된 지도와 위험 좌표를 이용해 목표 지점으로 이동합니다. 이후 OpenManipulator-X가 YOLO 객체 검출과 IBVS 제어를 이용해 표적을 자동으로 조준하고 타격합니다.

Active Scout에 장애가 발생하면 대기 중인 Follower Scout가 마지막 정찰 위치로 이동하여 탐색 임무를 이어받도록 설계했습니다.

### Operation Flow

1. Active Scout가 SAC 정책을 이용해 미지 환경을 자율 탐색
2. Cartographer SLAM으로 실시간 지도 생성
3. 카메라 객체 검출 결과를 Bayesian Risk Map에 반영
4. 지도·위험 좌표·로봇 상태를 Leader에게 공유
5. Leader Waffle이 Nav2를 이용해 대응 위치로 이동
6. OpenManipulator-X가 Point-at IK로 표적 방향을 개략 조준
7. YOLO와 IBVS 제어로 표적을 정밀 추적
8. 안전 조건을 만족하면 타격 동작 수행
9. Active Scout 장애 발생 시 Follower Scout가 임무 인계

## Robot Roles

| Robot              | Domain | Role       | Main Functions                                     |
| ------------------ | -----: | ---------- | -------------------------------------------------- |
| **Active Scout**   |     22 | 자율 정찰 로봇   | SAC 기반 탐색, Cartographer SLAM, 위험 지도 생성, 카메라 데이터 송신 |
| **Follower Scout** |     21 | 예비 정찰 로봇   | Leader 추종, Active Scout 상태 감시, 장애 발생 시 탐색 임무 인계    |
| **Leader Waffle**  |     20 | 대응 및 통합 로봇 | 공유 지도 수신, AMCL/Nav2 주행, 통합 대시보드, OMX 자동 조준 및 타격    |

## System Architecture

```text
[Active Scout · Domain 22]
 SAC RL Exploration
 Cartographer SLAM
 Bayesian Risk Map
 Camera Streaming
          │
          │ Map / Risk / Pose / Status
          ▼
[ROS 2 Domain Bridge]
          │
          ▼
[Leader Waffle · Domain 20]
 Unified Dashboard
 Shared Map + AMCL + Nav2
 Target Coordinate Processing
 OMX Auto-Aim
 YOLO + IBVS + Firing
          ▲
          │ Failover / Role / Pose
          │
[Follower Scout · Domain 21]
 Leader Following
 Scout Failure Detection
 Mission Takeover
```

각 로봇은 별도의 `ROS_DOMAIN_ID`에서 동작하며, Domain Bridge를 통해 지도·위험 정보·위치·상태처럼 협업에 필요한 데이터만 전달합니다. 하드웨어 속도 명령은 각 로봇의 도메인 내부에서만 처리하여 `/cmd_vel` 충돌을 방지했습니다.

## Key Features

* **SAC 기반 실물 로봇 자율 탐색**

  * 학습된 정책을 이용해 TurtleBot3의 선속도와 각속도 생성
* **실시간 SLAM 및 Bayesian Risk Map**

  * 탐색 중 생성된 지도에 객체 검출 결과와 위험도를 누적
* **Multi-Robot Failover**

  * Active Scout 장애 발생 시 Follower가 마지막 위치로 이동해 정찰 임무 인계
* **ROS 2 Multi-Domain Architecture**

  * Leader, Active Scout, Follower를 Domain 20·21·22로 분리
* **Nav2 기반 자율 대응**

  * 공유 지도와 위험 좌표를 기반으로 Leader Waffle의 이동 목표 생성
* **YOLO 및 IBVS 자동 조준**

  * 객체 검출 결과의 영상 중심 오차를 이용해 OpenManipulator-X 정밀 제어
* **안전 타격 상태 머신**

  * 조준 안정성, Deadband 유지 시간, Cooldown 및 비활성화 상태 확인
* **통합 웹 대시보드**

  * 로봇 카메라, YOLO 결과, 지도, 위험도, 경로 및 시스템 상태 시각화


## Demo Scenarios

영상 3개와 각 영상 설명

## My Contribution

본인이 직접 담당한 구현

## Tech Stack

ROS 2, Python, TurtleBot3, Jetson, SAC, Nav2, YOLO, IBVS

## Repository Structure

주요 패키지 설명

## Build and Run

실행 방법

## Related Repository

OMX Auto-Aim 서브시스템 링크