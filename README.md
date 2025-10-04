# FPGA_Image_Processing_Project

---

## 요약
- **OV7670 카메라 모듈**과 **VGA** 출력의 동작 원리를 심층적으로 이해한 뒤, 수집된 영상 처리 기법을 적용하여 실시간 교통관리 시스템을 구현하는 것을 목표 하였습니다.
- 최종적으로는 교통 흐름 감시, 이상 상황 탐지, 신호 제어 연계 등 실무에 적용 가능한 시스템을 개발하는 것을 목표로 하였습니다.

---
## 목차
- [팀원소개](#팀원소개)
- [개요](#개요)
- [SCCB Protocol](#SCCB-Protocol)
- [알고리즘](#알고리즘)
- [Architecture](#Architecture)
- [SystemVerilog 검증](#SystemVerilog-검증)
- [Trouble Shooting](#Trouble-Shooting)
- [결론 및 고찰](#결론-및-고찰)

---

## 팀원소개

<img width="1139" height="497" alt="image" src="https://github.com/user-attachments/assets/28b046a8-7ecb-4692-ae51-3bb6f5fc1eea" />

<img width="1140" height="493" alt="image" src="https://github.com/user-attachments/assets/cd2ab2c1-67fa-48f0-b178-bf66bc6a9f89" />

---

## 개요
### VGA 동작원리

<img width="931" height="701" alt="image" src="https://github.com/user-attachments/assets/40459663-e43d-4eca-a8bf-d7e3cc2b15e7" />

- 원하는 프레임보다 넘어서 Display되는 현상이 발생. **(porch 구간)**
- Display 시간 뿐만 아니라 **Front Porch, Sync, Back Porch**의 시간까지 고려하여 한 프레임이 만들어지는 시간 계산이 필요.

### 640x480 VGA Frame

<img width="875" height="654" alt="image" src="https://github.com/user-attachments/assets/f5322cd5-1086-44ce-80b7-f0b8d47aa928" />

- **Active Video (화면 표시 구간)**
    - 실제로 픽셀이 화면에 그려지는 구간. (사용자가 보는 부분)
- **Front Porch (전방 여유 구간)**
    - 한 줄/프레임을 다 그린 뒤 다음 sync 신호가 나오기 전까지의 짧은 쉬는 시간.
    - 전자총(브라운관 시대)이나 디스플레이 회로가 다음 줄을 준비하는 시간.
- **Sync Pulse (동기 신호 구간)**
    - 수평(Hsync), 수직(Vsync) 신호가 나오는 구간.
    - 다음 줄의 시작 또는 다음 프레임의 시작을 알려줌.
- **Back Porch (후방 여유 구간)**
    - sync가 끝난 뒤 실제 그림을 그리기 전까지의 준비 시간.
    - 이 구간이 지나야 다시 Active Video로 들어감.
  
<img width="329" height="159" alt="image" src="https://github.com/user-attachments/assets/e3e19b0e-d499-4439-89fc-f85004ed253c" />

- 60hz frame = 1초에 60 frame을 출력  = 800pixs x 525 lines x 60 = **약 25Mhz**

### VGA 하드웨어 연결

<img width="1128" height="592" alt="image" src="https://github.com/user-attachments/assets/c5117dcc-e33f-405d-adaa-8c538c45ea3a" />


- RGB가 각각 4bit씩 할당 되어 출력.
- 출력으로 나가기 전 저항 값은 Digital 신호를 Analog 신호로 변환하기 위함.
- Horizontal / Vertical Sync를 맞춰 동작.

![VGA_Display (1)](https://github.com/user-attachments/assets/718aa0f8-bc07-4425-bc63-83a80bd545e4)

- Pixel Counter : 픽셀을 표현할 위치를 지정.
- VGA Decoder :

  Porch 구간이 아닌 Display Pixel 구간을 판단하여 DE(Display Enable) 신호 전달 및 현재 픽셀 위치 값 전달.

   v_sync, h_sync 신호를 VGA로 전달.
- Display Data : RGB의 값을 지정하고 어떤 위치에서 어떤 값을 내보낼지를 판단하고 VGA로 정보 전달.

### OV7670 (30 Frame) ↔ FPGA (60 Frame) Connect

<img width="887" height="724" alt="image" src="https://github.com/user-attachments/assets/b1658cbf-d3ba-444b-ae26-1d59f75710b2" />

- HREF 값이 High(1)가 되면 Data가 즉시 나가는 구조.
- 첫 CLK에 1Byte (8bit) 픽셀의 첫 정보가 들어오고 바로 다음 CLK에 1Byte (8bit) 픽셀의 두 번째 정보가 들어와 한 픽셀을 처리하는 데에 2CLK이 소모된다.
- 이 때 RGB565 Format의 16bit 정보가 들어오는데 VGA는 RGB444 호환이기 때문에 하위 비트를 잘라내 RGB444 Format으로 내보내게 됨 (하위 비트가 표현하는데 있어 더 낮은 영향을 줌)

<img width="1539" height="891" alt="image" src="https://github.com/user-attachments/assets/d06d5f9d-1b9d-44c8-a231-8347ade5ff40" />

- VRAM에서 OV7670 카메라 모듈로부터 들어오는 데이터를 저장하고 그 값을 VGA MemoryController를 통해 외부 VGA로 출력.
- 이 때 OV7670을 저장하는데 사용되는 wclk과 RAM의 값을 출력하는데 사용되는 rclk이 다른 clk 주기를 가지기 때문에 **CDC문제**가 발생.
- 이를 해결하기 위해 제어신호를 rclk 도메인에서 **2-FF Syncronizer**로 동기화함으로써 CDC문제를 해결

---

## SCCB-Protocol
### Protocol 분석

<img width="1047" height="502" alt="image" src="https://github.com/user-attachments/assets/3555f4c1-e5f8-4fcb-bd7f-eb44cebb1fe0" />

- I2C 구조와 매우 유사한 프로토콜 방식으로 본래엔 3-wire까지 지원하지만 OV7670 카메라 모듈 특성상 2-wire 방식을 사용

<img width="1100" height="274" alt="image" src="https://github.com/user-attachments/assets/85ea445e-a5b1-4c2d-8c09-2031236123e2" />

- 맨 앞의 8bit는 모듈 고유의 주소 값, 다음 8bit는 레지스터의 주소 값, 마지막 8bit는 입력 데이터 총 24bit의 데이터 전송으로 프로토콜이 동작

<img width="938" height="417" alt="image" src="https://github.com/user-attachments/assets/ecfda5da-e749-4d65-ae9d-b621f5e86ba6" />

<img width="1175" height="447" alt="image" src="https://github.com/user-attachments/assets/8dc97b9c-a58b-49c1-a322-eb1361c204e4" />

### Architecture

<img width="774" height="473" alt="image" src="https://github.com/user-attachments/assets/4976e92b-30e0-4f4f-b392-a149c8d5d5fa" />

### Simulation

<img width="1078" height="423" alt="image" src="https://github.com/user-attachments/assets/8641de21-52a8-4e28-8be5-954edb067874" />

<img width="1076" height="429" alt="image" src="https://github.com/user-attachments/assets/2b4a0cc4-80fa-4acf-9ceb-6fb12553c6cd" />

---

## 알고리즘

### 픽셀 라벨링

<img width="964" height="297" alt="image" src="https://github.com/user-attachments/assets/d9083cc5-5311-4755-a8a4-0f693e86475b" />

- 실시간으로 들어오는 데이터의 특징을 추출하고 라벨링을 진행.
- 라벨링 된 데이터를 기반으로 특정 객체를 분류하는 알고리즘.

<img width="1121" height="554" alt="image" src="https://github.com/user-attachments/assets/f9de0a31-1a5f-4604-87fa-2291c7135194" />

- 라벨링 된 데이터를 기반으로 횡단보도를 구현하는 방식에 반복 패턴을 감지하는 방식과 라벨 비율로 감지하는 방식을 구상.
- 반복 패턴은 말 그대로 흰색 검은색의 반복으로 횡단보도를 추출하는 방식, 라벨 비율은 각 라인별로 들어오는 전체 라벨 중 흰색 픽셀의 비율을 계산하여 횡단보도 구간을 추출.
- 두 방식 모두 사용해본 결과 라벨 비율 방식이 더 정확도가 높아 라벨 비율 방식으로 횡단보도를 구분하기로 결정.

### 객체 탐지

<img width="1057" height="310" alt="image" src="https://github.com/user-attachments/assets/56a7ba19-0025-4fd7-b03e-ffe3cf233372" />

-이렇게 분류 된 횡단보도 구간을 집중 탐지 구역(ROI)으로 설정한 후 횡단보도 구간 내에 특정 객체 픽셀 값이 임계 값을 넘어가면 객체를 탐지해냄.


### 유동량 파악 및 신호 조정

<img width="444" height="424" alt="image" src="https://github.com/user-attachments/assets/c26d44c5-04f4-47df-927e-ed6e4f2a71fc" />

- 차량의 탐지를 통해 차량을 카운트하고 카운트 된 값을 기반하여 유동량을 파악
- 파악된 유동량을 기반하여 차량이 많으면 차량 신호를 길게 하고 차량이 적으면 차량 신호를 짧게 하여 원활한 신호를 받을 수 있도록 설계.

### 신호 위반

<img width="1079" height="394" alt="image" src="https://github.com/user-attachments/assets/62090a95-354c-4110-950a-3d0b17aded25" />

- 횡단보도 구간과 현재 도로 신호를 기반으로 객체가 신호를 위반할 시 즉시 외부로 음성 신호를 보내고 그 객체를 탐지함.

---

## Architecture

