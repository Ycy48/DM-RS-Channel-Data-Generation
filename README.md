# Data_Generation_DMRS.ipynb

현재 파일은 표준 PDSCH DMRS를 만든 후 물리 채널에 파형을 통과시키고, 수신 RS에서 LS/Oracle LMMSE를 계산하여 Real과 함께 저장합니다. COST2100·QuaDRiGa 생성부는 외부 `.m` 실행기 없이 Python으로 연결했습니다.

## 구현 범위를 먼저 확인

| 채널 | 현재 구현 |
|---|---|
| TDL-A~E / CDL-A~E | NeoRadium 0.5.3 기반 기존 배치 구현 유지. 기존 출력과 bitwise 비교 통과 |
| COST2100 | 공식 5개 network의 환경 파라미터·VR·cluster·MPC·local cluster·이동·경로손실·LOS 계산을 Python으로 옮긴 **1 BS·1 MS 경로** |
| QuaDRiGa | 공식 시나리오 파라미터, LSP/SOS, cluster delay/power/angle, scatterer, drifting, legacy/mmMAGIC subpath, GR, 추가 TOA 계산을 Python으로 옮긴 **고정 Tx·이동 Rx 단일 트랙 경로** |

**MATLAB/Octave는 실행하지 않습니다.** NumPy/SciPy/PyTorch와 Python 소스만 사용합니다. 임의 TDL/CDL 데이터를 COST2100·QuaDRiGa라는 이름으로 저장하지 않습니다.

다만 이는 이 실험에 필요한 경로의 source-equation port입니다. QuaDRiGa 전체 라이브러리의 모든 API를 옮긴 것은 아닙니다. 다중 트랙 병합·segment 전환·dual mobility·임의 측정 안테나 패턴·위성 궤도 트랙·Laplacian PAS는 현재 경로에 포함되지 않습니다. COST의 동시 다중 BS/MS 링크와 큰 배열 전체 출력도 포함되지 않습니다. 미지원 입력을 자동으로 비슷한 모델로 대체하지 않습니다.

**공식 MATLAB 출력과의 수치 동등성은 아직 검증하지 못했습니다.** 아래 검사는 실제 Python 생성·파형·추정·저장·재현 및 독립 수식 대조입니다. 원본 API 전체 이식 완료나 MATLAB parity 완료라고 해석하면 안 됩니다.

## 실행 순서

1. 첫 설정 셀에서 DATA_MODE, CHANNEL_TYPE_LIST, 수량, 안테나, 슬롯 수와 경로를 지정합니다.
2. 위에서 아래로 정의 셀을 실행합니다.
3. RS/데이터 배치 확인 셀에서 환경 목록·배치·예상 크기를 확인합니다.
4. 환경별 생성 셀을 실행합니다. Fix는 각 환경의 Train/Val/Test, Mix는 각 환경의 Test를 생성합니다.
5. 저장된 Test를 확인하고, 마지막 Mix 셀에서 전체 Train/Val을 생성합니다. Fix이면 마지막 셀은 건너뜁니다.

설정 변경 후에는 설정 셀부터 다시 실행합니다. native Python 파일을 수정했다면 **커널을 재시작**해야 합니다. 실행 중 로드된 코드와 디스크 코드가 다르면 명시적으로 오류를 냅니다. 사용자 커널이나 기존 학습은 수정 작업에서 재시작하지 않습니다.

COST2100·QuaDRiGa의 확률 변수·기하 생성은 현재 NumPy/SciPy로 CPU에서 계산하고, 파형 FIR·FFT·LS/MMSE는 COMPUTE_DEVICE에 따라 PyTorch CPU/GPU에서 계산합니다. 생성기 전체를 GPU로 옮긴 구현은 아닙니다. 특히 1,000 MPC를 사용하는 COST close-user 환경은 작은 IndoorHall보다 훨씬 오래 걸릴 수 있습니다.

## 공통 설정

```python
TRAIN_SAMPLES = 100_000
VAL_SAMPLES = 30_000
TEST_SAMPLES = 20_000
DATA_MODE = 'Mix'                    # 'Fix' 또는 'Mix'
CHANNEL_TYPE_LIST = ['CDL-C']         # 현재 기본값 유지
# CHANNEL_TYPE_LIST = ['TDL-A','CDL-C']
# CHANNEL_TYPE_LIST = ['COST2100']
# CHANNEL_TYPE_LIST = ['QuaDRiGa']
# CHANNEL_TYPE_LIST = ['COST2100','QuaDRiGa']

N_TX = 1
N_RX = 1
N_SLOTS = 1
N_RBS = 10
SCS_KHZ = 15
DMRS_SYMBOLS = (2,11)
INTERP = False
NORMALIZE_AVERAGE_POWER = True
DATA_ROOT = Path('E:/DMRS Channel')
COMPUTE_DEVICE = 'cuda'               # cpu / cuda / auto
GENERATION_BATCH_SIZE = 1000
RESUME = True
OVERWRITE = False
```

TDL/CDL은 `CHANNEL_TYPE_LIST × Delay × Doppler × SNR` 조합을 사용합니다. `['TDL','CDL']`은 각 PROFILE_LIST의 A~E로 전개됩니다.

COST2100·QuaDRiGa는 **선택 시나리오 × SNR**입니다. 기존 Delay/Doppler 리스트로 native 환경을 강제로 스케일하지 않습니다. 지연과 시간 변화는 native 시나리오·위치·속도에서 나옵니다.

## COST2100 선택

```python
CHANNEL_TYPE_LIST = ['COST2100']
COST2100_SCENARIO_LIST = ['IndoorHall_5GHz','SemiUrban_300MHz']
```

COST2100_SETTINGS에서 각 network의 다음 값을 지정합니다.

```python
'scenario': 'LOS',                 # 지원 profile에서만 NLOS 허용
'link': 'Single',                  # 현재 port는 1 BS / 1 MS
'band': 'Wideband',                # DMRS 파형 생성용
'frequency_range_hz': [5.29e9,5.31e9],
'bs_position_m': [10,10,0],
'ms_position_m': [10,5,0],
'ms_velocity_mps': [0,-0.1,0],
'power_calibration_samples': 32,    # 생략 시 32
```

공식 network는 IndoorHall_5GHz, SemiUrban_300MHz, Indoor_CloselySpacedUser_2_6GHz, SemiUrban_CloselySpacedUser_2_6GHz, SemiUrban_VLA_2_6GHz입니다. Indoor 두 종류는 공식 파라미터상 LOS만 지원하고, SemiUrban 세 종류는 LOS/NLOS를 선택할 수 있습니다. 설정의 주파수·위치 예시는 공식 demo를 기준으로 했으며, 예제 값이 공식 허용 범위 전체를 의미하지는 않습니다.

COST omni 배열 응답은 공식 SISO/2×2 식을 사용하므로 현재 Tx/Rx 각각 최대 2입니다. VLA profile은 BS-VR 통계를 적용하되 이 실험에서는 한 BS 위치의 omni 링크를 생성합니다. 공식 128개 큰 배열 전체를 출력하는 기능과 구분합니다.

공식 소스는 활성 cluster가 없으면 재시작을 요구합니다. 현재 코드는 그 경우만 명시적 seed 파생 방식으로 재시도하며, 0 채널을 대신 저장하지 않습니다. 기본 최대 재시도는 32회이고 그 후에도 실패하면 오류를 냅니다. seed로 재현 가능합니다.

## QuaDRiGa 선택

```python
CHANNEL_TYPE_LIST = ['QuaDRiGa']
QUADRIGA_SCENARIO_LIST = ['3GPP_38.901_UMi_LOS','3GPP_38.901_UMi_NLOS']
QUADRIGA_SETTINGS = {
    'frequency_hz':3.5e9,
    'tx_position_m':[0,0,10],
    'rx_position_m':[100,0,1.5],
    'rx_velocity_mps':[1,0,0],
    'absolute_delays':True,
    'power_calibration_samples':32,
}
QUADRIGA_SCENARIO_SETTINGS = {
    '3GPP_38.901_UMa_LOS': {'tx_position_m':[0,0,25]},
}
```

共通 설정 위에 시나리오별 QUADRIGA_SCENARIO_SETTINGS가 덮어써집니다. 방향은 속도 벡터 xyz 성분으로 정합니다. 현재 안테나는 수직 편파 omni element와 반파장 간격 ULA입니다. 물리 Rx/Tx 위치를 사용해 배열별 거리·위상·지연을 계산합니다.

공식 91개 config의 시나리오·pathloss 테이블을 사용합니다. 91개 이름 모두에 대한 작은 수치 생성 smoke check는 수행했지만, 그것이 각 시나리오의 실제 적용 범위·모든 geometry·통계·MATLAB 동등성을 검증했다는 뜻은 아닙니다. 예를 들어 UMi 설정의 Tx 높이를 위성 시나리오에 그대로 사용하는 것은 의미 있는 위성 실험이 아닙니다. 해당 환경의 실제 위치·주파수·이동 설정을 별도로 지정해야 합니다.

O2I는 `o2i_loss_db`, `indoor_distance_m`을 명시해야 합니다. 누락 시 일반 outdoor로 대체하지 않고 오류를 냅니다. 위성 orbital mobility와 track-segment 전환은 구현되지 않았습니다. 현재 이동은 지정한 3차원 선형 속도입니다.

`absolute_delays=True`는 전파 지연을 유지합니다. 너무 긴 절대 지연이 FFT 길이보다 길면 오류를 냅니다. 긴 링크에서 native `absolute_delays=False`를 명시하면 배열 기준 직접 경로 지연을 빼고 상대 지연을 사용할 수 있습니다. 배열의 일부 소자가 기준점보다 먼저 도착해 음의 상대 지연이 생기면 임의 clipping을 하지 않고 오류로 알려줍니다.

## Fix / Mix

- Fix: 선택한 환경마다 Train 10만 / Val 3만 / Test 2만. 파일 안의 순서는 Train→Val→Test.
- Mix: 선택한 조합 N개에 전체 Train 10만 / Val 3만을 균등 배분합니다. 환경별 Train/Val 원본 파일을 만들지 않고 직접 생성한 후 split별로 섞습니다. 각 환경의 Test 2만은 따로 저장합니다.
- LS/MMSE/Real에는 같은 Source_Index / Source_Row / Split_Index와 순열을 사용합니다. 슬롯은 한 샘플 안에서 함께 이동합니다.

TDL/CDL의 Mix Train/Val은 MIX_DELAY_LIST_NS / MIX_DOPPLER_LIST_HZ / MIX_SNR_LIST를 사용합니다. native 두 종류는 COST2100_SCENARIO_LIST / QUADRIGA_SCENARIO_LIST와 MIX_SNR_LIST를 사용합니다. 평가 SNR은 일반 SNR_LIST_DB입니다.

## DMRS·Real·저장 크기

0-based 인덱스입니다. 15 kHz Normal CP, 14 symbols/slot, 슬롯 길이 1 ms입니다. Type-1 mapping A, single-symbol DMRS, first position 2, additional position 1로 심볼 2·11에 RS를 넣습니다. SISO RS 위치는 0,2,…,118이며 두 심볼의 위치는 같고 시퀀스 값은 다릅니다.

데이터는 심볼 3~10, RS의 첫·마지막 사이인 SC 0~118에 배치합니다. 이는 표준 DMRS에 이 실험의 데이터 영역 제한을 적용한 것입니다.

| 값 | SISO / 1 slot 실제 HDF5 shape |
|---|---|
| LS_Channel | N×1×1×1×60×2 complex64 |
| MMSE_Channel | N×1×1×1×60×2 complex64 |
| Real_Channel | N×1×1×1×119×10 complex64 |

축은 sample,slot,port,rx,subcarrier,symbol입니다. R/I는 별도 축이 아닌 복소수로 저장합니다. Real에서 RS 위치 60×2를 CE 정답으로, 심볼 3~10의 119×8을 Data 정답으로 추출합니다.

Real은 실제 물리 CFR을 각 FFT 구간에서 평균한 대각 채널입니다. LS·MMSE를 보간해서 만드는 정답이 아니며 전체 ICI 행렬이나 수신 신호 자체도 아닙니다. LS는 실제 수신 RS로 계산합니다. **MMSE는 Real을 공분산에 사용하는 Oracle LMMSE**이며 실용 수신기의 통계 추정형 LMMSE와 구분해야 합니다.

INTERP=True이면 LS/MMSE만 RS 경계 안에서 119×10의 2D 선형 보간 결과로 바뀝니다. Real의 정의는 동일합니다. 현재 RS 입력 학습에는 INTERP=False를 사용합니다.

SISO/SIMO는 RS별 Y/X이고 FD-CDM 묶음 평균을 하지 않습니다. 여러 Tx 포트는 표준 FD-CDM pilot matrix로 분리합니다. Tx=3·4일 때 포트별 짝/홀수 SC가 달라 공통 Data 구간은 118×8, Real 저장 범위는 120×10이 됩니다. COST 현재 omni 매핑은 Tx/Rx 최대 2만 허용합니다.

N_SLOTS=5면 슬롯 축이 5이며, 한 행 안에서 동일한 실현의 이동과 FIR 이력이 이어집니다. 파일의 다음 행까지 시간 연속이라는 뜻은 아닙니다.

## 물리 채널의 시간 처리와 SRS 원본의 차이

SRS `Data_Generation_TDL_Integrated copy.ipynb`는 심볼마다 새 탭을 만들고 `h_l=alpha*h_(l-1)+sqrt(1-alpha²)*h_new`, `alpha=J0(2πfD*Tsymbol)`인 AR(1)로 갱신합니다. 한 심볼에서는 채널이 고정됩니다.

현재 DMRS TDL/CDL은 ray/SOS 채널을 anchor에서 계산한 뒤 시간 샘플의 복소 경로 이득을 선형으로 연결합니다. **SRS에서 복사한 방식이 아니라 DMRS 구현에 추가된 방식**입니다. NeoRadium 원래 applyToSignal의 심볼별 고정 방식과도 차이가 있습니다. 이 부분은 native 생성기 추가 작업에서 바꾸지 않았습니다.

native 두 종류는 시간에 따른 경로 계수와 지연으로 fractional-delay FIR을 만들고, FIR anchor 사이를 보간하며 이전 심볼·슬롯의 송신 파형까지 포함하여 선형 컨볼루션합니다. 이 시간 처리는 RS 추정 후의 INTERP 2D 보간과 별개입니다.

## 전력·SNR·지연 해석

native 생성기는 공식 모델의 경로손실·shadow fading·native carrier phase를 먼저 계산합니다. COST는 절대 RF 식을 baseband로 옮기며 carrier phase를 한 번만 포함합니다. QuaDRiGa 계수에 carrier phase를 중복 적용하지 않습니다.

NORMALIZE_AVERAGE_POWER=True면 독립 calibration 실현으로 계산한 **고정 ensemble 배율**을 모든 샘플에 동일하게 적용합니다. 샘플별 크기를 1로 맞추지 않으므로 상대 fading/SF 변화는 유지됩니다. False면 native 절대 경로손실까지 저장합니다. False에서 Tx EPRE 기준 SNR을 그대로 사용하면 수신 채널 전력이 매우 작아 LS 오차가 커질 수 있습니다.

SNR은 Tx EPRE 대비 FFT 이후 복소 AWGN 분산 기준이며 실측 수신 SINR이 아닙니다. 현재 FFT=2048, fs=30.72 MHz입니다. TDL-E/CDL-E 300 ns 같은 긴 지연에서는 CP를 초과한 ISI가 생길 수 있고 큰 Doppler에서는 ICI가 생깁니다. 이를 LS 구현 오류와 구분해야 합니다.

## 저장 경로

```text
E:/DMRS Channel/SISO/Fix 또는 Mix/
  CDL-C 또는 TDL-A/LS|MMSE|Real/<DMRS allocation>/<기존 환경 파일명>.h5
  COST2100/<network>/LS|MMSE|Real/<DMRS allocation>/<Scene + Config hash 파일명>.h5
  QuaDRiGa/<scenario>/LS|MMSE|Real/<DMRS allocation>/<Scene + Config hash 파일명>.h5
  <단일 family 또는 Mixed>/Random LS|MMSE|Real Channel/<allocation>/<Random 파일>.h5
```

native 환경에 TDL/CDL의 Delay/Doppler 숫자를 임의로 붙이지 않습니다. `condition_json`, `native_source_json`, Mix config에 시나리오·주파수·위치·속도·구현 hash·소스 버전을 저장합니다. 설정이 다르면 Config hash가 달라집니다.

allocation의 P1은 포트 수이고 nslot1은 슬롯 수입니다. 여러 슬롯은 `nslot5`, `Slot 0-4` 형식입니다. CDM은 상위 경로 대신 속성에 기록합니다.

완료된 기존 TDL/CDL 파일은 bitwise 회귀 검사한 이전 구현에 한해 호환 재사용합니다. OVERWRITE=False에서 다른 설정의 데이터를 자동으로 덮어쓰지 않습니다. 실제 데이터·학습 결과를 수정 작업에서 이동·삭제하지 않았습니다.

## 학습 파일과 Result

기존 5개 학습 로더의 TDL/CDL Fix/Mix 경로는 유지됩니다. **native scenario 이름·환경 키를 사용하는 학습/Result 자동 검색·Plot까지 이번 생성기 이식으로 검증한 것은 아닙니다.** 현재 native 검증 범위는 데이터 생성과 저장·재로딩입니다. 기존 Delay×Doppler 형태로 native 환경을 오표시하지 않도록 구분해서 연결해야 합니다.

## 파일·검사·출처

native Python 구현·검사·backup·vendor는 `DMRS_CSI_Compression/GPT Output/native_channel_implementation_20261103` 안에 모았습니다. 노트북은 해당 폴더의 Python 모듈을 읽습니다. 이 폴더를 삭제하면 native 생성기를 실행할 수 없습니다. 전역 Python 환경에 패키지를 설치하지 않았습니다.

- 실제 COST·QuaDRiGa 생성→DMRS 파형→LS/MMSE/Real 크기·Pair·SNR 간 Real 일치·재현 검사.
- Fix/Mix 저장, 완료 Test 재사용, 기존 TDL/CDL Mix 파일 유지 검사.
- 기존 TDL/CDL 출력 bitwise 회귀 검사.
- 경로손실·프로파일 상수·DS 스케일링·scatterer 기하 제약·정지/이동·MIMO/여러 슬롯·CPU/CUDA 검사.
- 91개 공식 QuaDRiGa config 테이블 일치 및 작은 생성 smoke census. **공식 모델 전체 적합성 검증이나 MATLAB 출력 대조를 대신하지 않습니다.**

대량 E 드라이브 데이터 생성과 모델 학습은 실행하지 않았습니다. 검증에는 GPT Output 아래의 작은 별도 데이터를 사용했습니다.

공식 소스:

- [COST2100](https://github.com/cost2100/cost2100), commit `bafca0b84faa2aa0441cb3f03f9dbd65771f43b3`
- [QuaDRiGa](https://github.com/fraunhoferhhi/QuaDRiGa), commit `277866650eb115adb5b3e8ac252b0d1df073596d`
- [NeoRadium channels](https://interdigitalinc.github.io/NeoRadium/html/source/API/Channels.html)
- [pyqd-channel 0.2.3](https://pypi.org/project/pyqd-channel/): pure-Python scenario/pathloss/SOS 부분을 사용했습니다. 이 패키지에 없는 SSF·이동·drifting 경로는 별도로 이식했습니다.

원본 라이선스·저자 정보와 이식 시 변경점은 구현 폴더의 LICENSE/NOTICE에 보존했습니다.
