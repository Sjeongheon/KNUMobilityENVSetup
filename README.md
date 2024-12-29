# KNU Mobility Environment Setup

SCOUT MINI 프로젝트를 위한 Docker 기반 ROS 개발 환경입니다.

## Supported Environments

| Environment | Base |
| --- | --- |
| Melodic | ROS Melodic Desktop Full |
| Noetic | ROS Noetic Desktop Full |

## Included

- Gazebo
- Cartographer ROS
- `ugv_sdk`
- `scout_ros`
- SocketCAN / `can-utils`
- X11 / OpenGL
- NVIDIA Container Runtime 지원

## Structure

```text
KNUMobilityENVSetup/
├── melodicSetup/
│   └── Dockerfile
├── noeticSetup/
│   └── Dockerfile
├── scripts/
│   ├── build.sh
│   └── run.sh
├── .dockerignore
└── README.md
```

## Build

```bash
./scripts/build.sh melodic
./scripts/build.sh noetic
```

## Run

```bash
./scripts/run.sh melodic
./scripts/run.sh noetic
```

NVIDIA GPU 사용 시:

```bash
USE_GPU=1 ./scripts/run.sh melodic
```

## CAN

Host에서 CAN Interface를 활성화한 뒤 Container를 실행합니다.

```bash
sudo modprobe gs_usb
sudo ip link set can0 up type can bitrate 500000
```

CAN 통신 확인:

```bash
candump can0
```

## WSL2

WSL2 환경에서 SCOUT MINI의 CAN Interface를 사용하는 경우, 기본 WSL2 Kernel에서 CAN 및 `gs_usb` 모듈을 지원하는지 확인해야 합니다.

필요한 Kernel Module을 사용할 수 없는 경우 Custom Kernel 설정이 필요합니다.

> [Windows + WSL2 환경 설정 가이드](https://app.notion.com/p/knu-mobility/Windows-WSL2-1168d4193db68072b1e6f0b9cf7564b2?v=ccf5b3dbdb6c4e64be7e6f7ef12b586d)