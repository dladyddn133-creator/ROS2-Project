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

## Custom YOLO11n Target Detector

프로젝트의 분홍색 표적을 실시간으로 검출하기 위해  
직접 데이터를 수집하고 라벨링하여 단일 클래스 YOLO11n 모델을 학습했습니다.

### Training Pipeline

1. 실제 로봇 운용 환경에서 표적 영상을 촬영
2. 영상 프레임을 이미지 데이터로 추출
3. 표적 영역을 `target` 클래스로 직접 라벨링
4. Train / Validation 데이터셋 분리
5. YOLO11n 모델을 100 epochs 학습
6. 검증 성능 확인 후 Jetson Orin Nano의 실시간 검출 노드에 적용
7. 검출 박스 중심 좌표를 IBVS 제어 입력으로 사용

### Model Performance

| Metric | Validation Result |
|---|---:|
| Precision | ≈ 0.99 |
| Recall | ≈ 0.99 |
| mAP@0.5 | ≈ 0.99 |
| mAP@0.5:0.95 | ≈ 0.79 |
| Training Epochs | 100 |
| Model | YOLO11n |
| Classes | 1 (`target`) |

> 위 성능은 프로젝트 실험 환경에서 구성한 validation dataset 기준입니다.

## My Contribution

본 프로젝트에서 **Leader Waffle의 자동 조준 및 타격 서브시스템 개발과 전체 시스템 통합**을 담당했습니다.

### OMX Auto-Aim System

* OpenManipulator-X 기반 자동 조준 및 타격 파이프라인 설계
* 지도상의 위험 좌표를 로봇팔 조준 방향으로 변환하는 Point-at IK 적용
* YOLO 객체 검출 결과와 영상 중심 오차를 이용한 IBVS 제어 구현
* 표적 중심 정렬을 위한 Pan/Tilt PD 제어 및 Deadband 튜닝
* `AIMING → SCANNING → TRACKING → CONFIRMING → FIRING` 상태 머신 구현
* 조준 안정성 유지 시간과 Cooldown을 포함한 안전 타격 조건 설계

### ROS 2 Integration

* Scout가 생성한 지도와 위험 좌표를 Leader Waffle의 이동 및 조준 시스템과 연동
* Nav2 이동 명령과 OMX 조준 동작을 비동기 상태 머신으로 연결
* 긴급 TARGET 입력 시 진행 중인 순찰 작업을 중단하는 우선순위 큐 및 선점 처리
* 조준 가능한 위치가 아닐 경우 주변 후보 위치를 평가하여 재이동하는 View Pose 로직 적용
* ROS 2 토픽과 노드 간 인터페이스 설계 및 실제 로봇 통합 테스트

### Embedded and Hardware Integration

* Jetson Orin Nano에서 ROS 2, YOLO 및 OpenManipulator-X 구동 환경 구성
* GPIO 기반 타격 장치 제어 및 발사 펄스 출력 구현
* 부팅·종료 시 출력 LOW 유지, 타격 비활성화 및 Cooldown 안전 기능 적용
* 실물 로봇의 모터 방향, 관절 한계, 조준 오차 및 제어 파라미터 조정

### Monitoring and Demonstration

* YOLO 카메라 영상과 조준 상태를 확인하기 위한 Flask 기반 디버그 스트리밍 연동
* 통합 대시보드에서 로봇 상태, 실시간 지도, 경로 및 객체 검출 결과 검증
* 실물 로봇과 대시보드 영상을 동기화하여 전체 탐색·이동·조준·타격 과정 기록

> 전체 다중 로봇 시스템은 팀원들과 공동으로 개발했으며,
> 본 저장소에서는 제가 담당한 OMX 자동 조준·타격 및 Leader 연동 부분을 중심으로 정리했습니다.

## Tech Stack

ROS 2, Python, TurtleBot3, Jetson, SAC, Nav2, YOLO, IBVS

## Repository Structure

주요 패키지 설명

## Build and Run

실행 방법

## Related Repository

OMX Auto-Aim 서브시스템 링크