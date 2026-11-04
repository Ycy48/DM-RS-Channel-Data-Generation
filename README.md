# Data_Generation_DMRS.ipynb

최종 대조: **2026-11-04**. 같은 폴더의 현재 노트북 설정·생성 함수·저장 경로와 학습 파일의 로더/출력 코드를 기준으로 작성했습니다.

## 현재 설정으로 실행하면

현재 선택은 `DATA_MODE = 'Mix'`, `CHANNEL_TYPE_LIST = ['CDL-C']`입니다. **그대로 실행하면 CDL-C만 생성합니다.** COST2100·QuaDRiGa 설정이 셀에 있어도 `CHANNEL_TYPE_LIST`에서 선택하지 않으면 생성하지 않습니다.

| 구분 | Delay (ns) | Doppler (Hz) | SNR (dB) | 생성 수량 |
|---|---|---|---|---|
| Mix Train/Val | 25, 75, 150, 200, 300 | 50, 200, 500 | 10, 15, 20 | 45개 조합에 Train 총 100,000 / Val 총 30,000 균등 배분 |
| Mix Random Test | 25, 75, 150, 200, 300 | 50, 200, 500 | −10, −5, 0, 5, 10, 15, 20 | 105개 조합에 Test 총 20,000 균등 배분: 환경당 190~191 |

Train과 Val은 기존 Random HDF5 세트의 서로 다른 구간에 들어갑니다. Test는 별도의 **Random Test HDF5 세트**입니다. Mix에서는 환경별 Test 파일을 새로 만들지 않습니다. 현재 총수는 **Train 100,000 + Val 30,000 + Test 20,000 = 150,000 샘플**이며 각 세트의 LS/MMSE/Real에 같은 Pair와 순열을 적용합니다.

기본 SISO·1슬롯·complex64에서 채널 배열만의 비압축 크기는 약 **1.716 GB**입니다. HDF5 메타데이터·임시 파일은 제외한 값이며 실제 파일 크기는 압축률에 따라 달라집니다. 같은 목록을 Fix로 생성하면 105개 환경마다 150,000개씩이므로 약 **180.18 GB**입니다.

이 문서의 기본 수량은 현재 설정의 예시입니다. 실제 생성량은 노트북의 목록과 `TRAIN_SAMPLES`·`VAL_SAMPLES`·`TEST_SAMPLES`를 따릅니다.

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
4. `Fix 전체 split / Mix Random Test 생성` 셀을 실행합니다. Fix는 환경마다 Train/Val/Test를, Mix는 Test 목록의 전체 조합에 Test 총수를 배분한 Random Test 세트를 생성합니다.
5. 저장된 채널을 확인하고, 마지막 Mix 셀에서 Train/Val 목록의 전체 조합에 Train/Val 총수를 배분합니다. Fix이면 마지막 셀은 건너뜁니다. Test와 Train/Val 생성 순서는 서로 의존하지 않습니다.

설정 변경 후에는 설정 셀부터 다시 실행합니다. native Python 파일을 수정했다면 **커널을 재시작**해야 합니다. 실행 중 로드된 코드와 디스크 코드가 다르면 명시적으로 오류를 냅니다. 사용자 커널이나 기존 학습은 수정 작업에서 재시작하지 않습니다.

COST2100·QuaDRiGa의 확률 변수·기하 생성은 현재 NumPy/SciPy로 CPU에서 계산하고, 파형 FIR·FFT·LS/MMSE는 COMPUTE_DEVICE에 따라 PyTorch CPU/GPU에서 계산합니다. 생성기 전체를 GPU로 옮긴 구현은 아닙니다. 특히 1,000 MPC를 사용하는 COST close-user 환경은 작은 IndoorHall보다 훨씬 오래 걸릴 수 있습니다.

## 공통 설정

```python
TRAIN_SAMPLES = 100_000
VAL_SAMPLES = 30_000
TEST_SAMPLES = 20_000                # Fix: 환경마다 / Mix: Test 전체 총수
DATA_MODE = 'Mix'                    # 'Fix' 또는 'Mix'
CHANNEL_TYPE_LIST = ['CDL-C']         # 이 목록에 선택된 채널만 생성
# CHANNEL_TYPE_LIST = ['TDL-A','CDL-C']
# CHANNEL_TYPE_LIST = ['COST2100']
# CHANNEL_TYPE_LIST = ['QuaDRiGa']
# CHANNEL_TYPE_LIST = ['COST2100','QuaDRiGa']
# CHANNEL_TYPE_LIST = ['CDL-C','COST2100','QuaDRiGa']

# Fix 전체 split / Mix Random Test 환경 목록에 사용
SNR_LIST_DB = [-10,-5,0,5,10,15,20]
DELAY_SPREAD_LIST_NS = [25,75,150,200,300]
DOPPLER_LIST_HZ = [50,200,500]

# Mix Train/Val에 사용
MIX_SNR_LIST = [10,15,20]
MIX_DELAY_LIST_NS = [25,75,150,200,300]
MIX_DOPPLER_LIST_HZ = [50,200,500]
MIX_SEED = 42
SEED = 20261021

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
PATH_CHUNK_SIZE = 4
COMPUTE_DTYPE = 'complex64'
HDF5_COMPRESSION = 'lzf'
SHOW_PROGRESS = True
RESUME = True
OVERWRITE = False
```

TDL/CDL은 선택한 채널마다 `Delay × Doppler × SNR` 조합을 사용합니다. `['TDL','CDL']`은 `TDL_PROFILE_LIST`, `CDL_PROFILE_LIST`의 A~E로 전개됩니다.

현재 구현은 **10 RB·15 kHz·Normal CP·DMRS 심볼 (2,11)**로 고정되어 있습니다. 첫 설정 셀의 값을 생성 직전에 다시 읽으며, 이 조건을 벗어나는 `N_RBS`·`SCS_KHZ`·`DMRS_SYMBOLS`는 검사에서 거부합니다. `CarrierSettings`의 기본값만 바꿔서 상단 설정과 다른 파형을 만들지 않습니다.

COST2100·QuaDRiGa는 **선택 시나리오 × SNR**입니다. 기존 Delay/Doppler 리스트로 native 환경을 강제로 스케일하지 않습니다. 지연과 시간 변화는 native 시나리오·위치·속도에서 나옵니다.

## COST2100 선택

```python
CHANNEL_TYPE_LIST = ['COST2100']
COST2100_SCENARIO_LIST = ['IndoorHall_5GHz','SemiUrban_300MHz']
```

COST2100_SETTINGS에서 각 network의 다음 값을 지정합니다.

```python
COST2100_SETTINGS['IndoorHall_5GHz'] = {
    'scenario': 'LOS',              # 지원 profile에서만 NLOS 허용
    'link': 'Single',               # 현재 port는 1 BS / 1 MS
    'band': 'Wideband',             # DMRS 파형 생성용
    'frequency_range_hz': [5.29e9,5.31e9],
    'bs_position_m': [10,10,0],
    'ms_position_m': [10,5,0],
    'ms_velocity_mps': [0,-0.1,0],
    'power_calibration_samples': 32, # 생략 시 32
}
```

현재 생성 경로가 지원하는 조합은 다음 8개입니다. 노트북에 적힌 기본 COST 시나리오는 `IndoorHall_5GHz` 하나입니다.

| Network 이름 | 지원 설정 |
|---|---|
| `IndoorHall_5GHz` | LOS |
| `SemiUrban_300MHz` | LOS / NLOS |
| `Indoor_CloselySpacedUser_2_6GHz` | LOS |
| `SemiUrban_CloselySpacedUser_2_6GHz` | LOS / NLOS |
| `SemiUrban_VLA_2_6GHz` | LOS / NLOS |

각 항목의 `COST2100_SETTINGS[network]['scenario']`를 선택합니다. 현재 DMRS 어댑터에는 **`link='Single'`, `band='Wideband'`**를 사용해야 합니다. 한 network를 여러 번 적는 중복 목록은 허용하지 않습니다.

설정의 주파수·위치 예시는 공식 demo를 기준으로 했으며, 예제 값이 공식 허용 범위 전체를 의미하지는 않습니다. `frequency_range_hz`의 중심이 native carrier가 되고, NR 파형의 활성 대역폭은 10 RB × 12 × 15 kHz = 1.8 MHz입니다. 두 값을 같은 대역폭 설정으로 혼동하지 않습니다.

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

공통 `QUADRIGA_SETTINGS` 위에 선택한 시나리오의 `QUADRIGA_SCENARIO_SETTINGS`가 덮어써집니다. 방향은 속도 벡터 xyz 성분으로 정합니다. 현재 안테나는 수직 편파 omni element와 반파장 간격 ULA입니다. 물리 Rx/Tx 위치를 사용해 배열별 거리·위상·지연을 계산합니다.

현재 기본 선택은 `QUADRIGA_SCENARIO_LIST = ['3GPP_38.901_UMi_NLOS']`입니다. 선택 가능한 공식 이름은 노트북의 `QUADRIGA_OFFICIAL_SCENARIOS`에서 확인합니다. 공식 91개 config의 시나리오·pathloss 테이블을 사용합니다. 91개 이름 모두에 대한 작은 수치 생성 smoke check는 수행했지만, 그것이 각 시나리오의 실제 적용 범위·모든 geometry·통계·MATLAB 동등성을 검증했다는 뜻은 아닙니다. 예를 들어 UMi 설정의 Tx 높이를 위성 시나리오에 그대로 사용하는 것은 의미 있는 위성 실험이 아닙니다. 해당 환경의 실제 위치·주파수·이동 설정을 별도로 지정해야 합니다.

O2I는 `o2i_loss_db`, `indoor_distance_m`을 명시해야 합니다. 누락 시 일반 outdoor로 대체하지 않고 오류를 냅니다. 위성 orbital mobility와 track-segment 전환은 구현되지 않았습니다. 현재 이동은 지정한 3차원 선형 속도입니다.

`absolute_delays=True`는 전파 지연을 유지합니다. 너무 긴 절대 지연이 FFT 길이보다 길면 오류를 냅니다. 긴 링크에서 native `absolute_delays=False`를 명시하면 배열 기준 직접 경로 지연을 빼고 상대 지연을 사용할 수 있습니다. 배열의 일부 소자가 기준점보다 먼저 도착해 음의 상대 지연이 생기면 임의 clipping을 하지 않고 오류로 알려줍니다.

## Fix / Mix

두 모드 모두 `CHANNEL_TYPE_LIST`의 선택 항목을 순차적으로 생성합니다. 여러 family를 선택한 Mix에서는 그 항목들을 합친 **목록별 전체 조합 수 N**을 기준으로 수량을 배분합니다. Train/Val 목록의 N과 Test 목록의 N은 별도입니다. 예를 들어 COST network 하나와 QuaDRiGa 시나리오 하나, Mix SNR 세 개를 선택하면 N=6입니다. native 환경에는 TDL/CDL Delay·Doppler 목록을 추가 곱하지 않습니다.

- Fix: 선택한 환경마다 Train 10만 / Val 3만 / Test 2만. 파일 안의 순서는 Train→Val→Test.
- Mix: `MIX_*_LIST` 조합에 Train 총 10만 / Val 총 3만, 일반 `SNR_LIST_DB`·`DELAY_SPREAD_LIST_NS`·`DOPPLER_LIST_HZ` 조합에 Test 총 2만을 각각 균등 배분합니다. 환경별 원본 파일을 만들지 않고 필요한 quota만 직접 생성합니다. 각 split의 생성이 끝나면 LS/MMSE/Real을 동일한 순열로 한 번 섞습니다.
- Random Mix의 LS/MMSE/Real에는 같은 `Source_Index` / `Source_Row` / `Split_Index`와 순열을 사용합니다. 슬롯은 한 샘플 안에서 함께 이동합니다.

환경마다 `총수 // N`을 기본으로 할당하고 나머지를 배분하므로, 같은 split에서 환경별 수량 차이는 최대 1개입니다. Train·Val·Test는 각각 따로 섞으며 LS/MMSE/Real의 Pair는 함께 이동합니다. Test 총수가 환경 조합 수보다 작으면 모든 환경을 평가할 수 없으므로 생성 전에 오류를 냅니다. Train/Val과 Test를 합쳐서 다시 섞지 않습니다.

| Random Mix 데이터셋 | 의미 |
|---|---|
| `Source_Index` | `Sources_JSON[index]['condition']`에 연결되는 환경 번호 |
| `Source_Row` | 그 환경의 물리 샘플 난수 번호. 외부 원본 파일을 읽으라는 행 번호가 아님 |
| `Split_Index` | Train/Val Random: 0=Train, 1=Val / 별도 Random Test: 모든 행이 2 |
| `Sources_JSON`, `Metadata_JSON` | 환경 목록·설정·수량·quota·구현 ID·난수 seed |

기본 물리 샘플 번호는 Train `[0,100000)`, Val `[100000,130000)`, Test `[130000,150000)`입니다. Mix는 각 환경에서 필요한 번호만 선택해 직접 생성하고, 환경별 Test는 Test 번호 구간을 사용합니다. 실제 구간은 설정 수량에 따라 계산됩니다. 같은 환경의 Train/Val/Test 번호가 겹치지 않으며, SNR만 바꿔 비교할 때는 동일 번호의 채널·송신 데이터·기저 잡음 난수를 공유하도록 설계되어 있습니다.

이 인덱스 덕분에 섞인 데이터에서도 환경별 Train/Validation Loss와 Test MSE/NMSE를 집계할 수 있습니다. 환경 표시는 모델 입력에 추가되는 값이 아닙니다.

TDL/CDL의 Mix Train/Val은 MIX_DELAY_LIST_NS / MIX_DOPPLER_LIST_HZ / MIX_SNR_LIST를 사용합니다. native 두 종류는 COST2100_SCENARIO_LIST / QUADRIGA_SCENARIO_LIST와 MIX_SNR_LIST를 사용합니다. 평가 SNR은 일반 `SNR_LIST_DB`입니다. Random Test에는 이 목록의 낮은 SNR도 포함됩니다. 두 종류의 데이터 세트는 서로 독립된 목록·물리 샘플 번호 구간을 사용합니다.

## DMRS·Real·저장 크기

0-based 인덱스입니다. 15 kHz Normal CP, 14 symbols/slot, 슬롯 길이 1 ms입니다. Type-1 mapping A, single-symbol DMRS, first position 2, additional position 1로 심볼 2·11에 RS를 넣습니다. SISO RS 위치는 0,2,…,118이며 두 심볼의 위치는 같고 시퀀스 값은 다릅니다.

데이터는 심볼 3~10, RS의 첫·마지막 사이인 SC 0~118에 배치합니다. 이는 표준 DMRS에 이 실험의 데이터 영역 제한을 적용한 것입니다. CSI-RS는 생성하지 않습니다. HDF5에는 채널과 배치·난수·환경 메타데이터를 저장하며 송수신 파형이나 QAM bit 배열 자체를 저장하지는 않습니다.

| 값 | SISO / 1 slot 실제 HDF5 shape |
|---|---|
| LS_Channel | N×1×1×1×60×2 complex64 |
| MMSE_Channel | N×1×1×1×60×2 complex64 |
| Real_Channel | N×1×1×1×119×10 complex64 |

축은 sample,slot,port,rx,subcarrier,symbol입니다. R/I는 별도 축이 아닌 복소수로 저장합니다. Real에서 RS 위치 60×2를 CE 정답으로, 심볼 3~10의 119×8을 Data 정답으로 추출합니다.

Real은 실제 물리 CFR을 각 FFT 구간에서 평균한 대각 채널입니다. LS·MMSE를 보간해서 만드는 정답이 아니며 전체 ICI 행렬이나 수신 신호 자체도 아닙니다. LS는 실제 수신 RS로 계산합니다. **MMSE는 Real을 공분산에 사용하는 Oracle LMMSE**이며 실용 수신기의 통계 추정형 LMMSE와 구분해야 합니다.

INTERP=True이면 LS/MMSE만 RS 경계 안에서 119×10의 2D 선형 보간 결과로 바뀝니다. Real의 정의는 동일합니다. 현재 RS 입력 학습에는 INTERP=False를 사용합니다.

SISO/SIMO는 RS별 Y/X이고 FD-CDM 묶음 평균을 하지 않습니다. 여러 Tx 포트는 표준 FD-CDM pilot matrix로 분리합니다. Tx=3·4일 때 포트별 짝/홀수 SC가 달라 공통 Data 구간은 118×8, Real 저장 범위는 120×10이 됩니다. COST 현재 omni 매핑은 Tx/Rx 최대 2만 허용합니다.

`N_SLOTS=5`면 LS/MMSE는 `N×5×1×1×60×2`, Real은 `N×5×1×1×119×10`입니다. 한 행 안에서 동일한 실현의 이동과 FIR 이력이 슬롯 경계를 넘어 이어집니다. 파일의 다음 행은 독립 샘플이며, 행 사이까지 연속인 것은 아닙니다.

슬롯 수를 늘려도 환경 조합이 같으면 최종 LS/MMSE/Real 파일 개수는 같고 슬롯 축이 늘어납니다. 비압축 배열 원소 수는 슬롯 수에 비례하지만 압축 후 파일 크기가 정확히 그 배수가 되지는 않을 수 있습니다. `Slot_Indices`, `Slot_Start_Times_s`, `Symbol_FFT_Start_Times_s`에 슬롯 번호와 실제 시간 좌표를 저장합니다.

## 물리 채널의 시간 처리와 SRS 원본의 차이

SRS `Data_Generation_TDL_Integrated copy.ipynb`는 심볼마다 새 탭을 만들고 `h_l=alpha*h_(l-1)+sqrt(1-alpha²)*h_new`, `alpha=J0(2πfD*Tsymbol)`인 AR(1)로 갱신합니다. 한 심볼에서는 채널이 고정됩니다.

현재 DMRS TDL/CDL은 ray/SOS 채널을 anchor에서 계산한 뒤 시간 샘플의 복소 경로 이득을 선형으로 연결합니다. **SRS에서 복사한 방식이 아니라 DMRS 구현에 추가된 방식**입니다. NeoRadium 원래 applyToSignal의 심볼별 고정 방식과도 차이가 있습니다. 이 부분은 native 생성기 추가 작업에서 바꾸지 않았습니다.

native 두 종류는 시간에 따른 경로 계수와 지연으로 fractional-delay FIR을 만들고, FIR anchor 사이를 보간하며 이전 심볼·슬롯의 송신 파형까지 포함하여 선형 컨볼루션합니다. 이 시간 처리는 RS 추정 후의 INTERP 2D 보간과 별개입니다.

## CPU/GPU와 실제 배치 크기

TDL/CDL은 NeoRadium의 설정·안테나·ray/SOS 계수를 준비한 뒤 배치 커널과 CP-OFDM/FIR/FFT/추정을 `COMPUTE_DEVICE`에 맞춰 실행합니다. native COST2100·QuaDRiGa의 확률·기하 생성은 CPU이고, 이후 파형/추정은 PyTorch CPU 또는 CUDA입니다.

현재 실제 생성 배치는 다음 규칙입니다.

```text
scale = N_SLOTS × N_TX × N_RX
divisor = scale 이상인 가장 작은 2의 거듭제곱
effective_batch_size = max(1, GENERATION_BATCH_SIZE // divisor)
```

기준 배치 1000일 때 SISO·1슬롯은 1000, SISO·5슬롯은 `1000 // 8 = 125`, 2×2 MIMO·1슬롯은 250입니다. 여기서 8은 채널 tap 수가 아니라 슬롯·안테나 크기에 대한 배치 축소 계수입니다. **여유 VRAM을 실시간으로 측정해서 최적 배치를 찾는 기능은 없습니다.** 메모리가 부족하면 `GENERATION_BATCH_SIZE`를 줄입니다. TDL/CDL의 FIR 경로 분할에는 `PATH_CHUNK_SIZE`도 적용되지만 native CIR 경로의 같은 메모리 조절 값은 아닙니다. 배치를 늘려도 CPU native 생성이 병목이면 같은 비율로 빨라지지 않습니다.

`SHOW_PROGRESS=True`는 생성/셔플 진행 bar를 표시합니다. 학습 파일은 로딩·학습·저장·Test의 현재 작업 bar를 표시하고, 완료되면 같은 줄을 비워 다음 작업에 재사용합니다. Data Generation의 생성 진행률 설정과는 별개입니다.

## 전력·SNR·지연 해석

native 생성기는 공식 모델의 경로손실·shadow fading·native carrier phase를 먼저 계산합니다. COST는 절대 RF 식을 baseband로 옮기며 carrier phase를 한 번만 포함합니다. QuaDRiGa 계수에 carrier phase를 중복 적용하지 않습니다.

NORMALIZE_AVERAGE_POWER=True면 독립 calibration 실현으로 계산한 **고정 ensemble 배율**을 모든 샘플에 동일하게 적용합니다. 샘플별 크기를 1로 맞추지 않으므로 상대 fading/SF 변화는 유지됩니다. False면 native 절대 경로손실까지 저장합니다. False에서 Tx EPRE 기준 SNR을 그대로 사용하면 수신 채널 전력이 매우 작아 LS 오차가 커질 수 있습니다.

SNR은 Tx EPRE 대비 FFT 이후 복소 AWGN 분산 기준이며 실측 수신 SINR이 아닙니다. 현재 FFT=2048, fs=30.72 MHz입니다. TDL-E/CDL-E 300 ns 같은 긴 지연에서는 CP를 초과한 ISI가 생길 수 있고 큰 Doppler에서는 ICI가 생깁니다. 이를 LS 구현 오류와 구분해야 합니다.

## 저장 경로

Fix 모드의 기본 SISO·1슬롯 환경별 파일 예시는 다음과 같습니다. 내용에 그 환경의 Train/Val/Test가 모두 들어갑니다.

```text
E:/DMRS Channel/SISO/Fix/CDL-C/LS/
  DMRS SC60xSym2 P1 nslot1 Comb2 RB0-9 l2,11/
    LS Channel 2D 0dB (Delay 1.5e-07, Slot 0, Doppler 50Hz, Tx1 Rx1 L1, BWP120).h5
```

MMSE/Real은 각각 `MMSE`, `Real` 폴더와 같은 종류의 파일명을 사용합니다. 5슬롯은 allocation에 `nslot5`, 실제 파일명에 `Slot 0-4`가 들어갑니다. `P1`은 포트 1개이며 슬롯 수가 아닙니다.

Random Mix 예시:

```text
E:/DMRS Channel/SISO/Mix/CDL-C/Random LS Channel/
  DMRS SC60xSym2 P1 nslot1 Comb2 RB0-9 l2,11/
    Random LS Channel 2D (DMRS SC60xSym2 P1 nslot1 Comb2 RB0-9 l2,11).h5
    Random Channel Config.txt
```

Random Test 예시 (Train/Val Random 파일과 별도):

```text
E:/DMRS Channel/SISO/Mix/CDL-C/Random Test LS Channel/
  DMRS SC60xSym2 P1 nslot1 Comb2 RB0-9 l2,11/
    Random Test LS Channel 2D (DMRS SC60xSym2 P1 nslot1 Comb2 RB0-9 l2,11).h5
    Random Test Config.txt
```

MMSE/Real은 `Random Test MMSE Channel`, `Random Test Real Channel` 폴더를 사용합니다. Test 세트의 `counts`는 `[0,0,20000]`, 포맷은 `dmrs-random-test-1.0`(여러 슬롯은 `dmrs-multi-slot-random-test-1.0`)입니다. Train/Val 세트는 기존 이름·포맷과 `[100000,30000,0]`을 유지합니다.

여러 채널/프로파일을 한 Mix로 섞으면 `CDL-C` 자리는 `Mixed`입니다. 단일 COST2100 family만 선택하면 `COST2100`, 단일 QuaDRiGa family만 선택하면 `QuaDRiGa`입니다. 세 수량의 폴더명은 `Random LS Channel`, `Random MMSE Channel`, `Random Real Channel`입니다. `INTERP=True`일 때는 별도의 `Interp2D` 이름으로 저장합니다.

전체 구분:

```text
E:/DMRS Channel/SISO/Fix/
  CDL-C 또는 TDL-A/LS|MMSE|Real/<allocation>/<환경 파일명>.h5
  COST2100/<network>/LS|MMSE|Real/<allocation>/<Scene + Config hash 파일명>.h5
  QuaDRiGa/<scenario>/LS|MMSE|Real/<allocation>/<Scene + Config hash 파일명>.h5
E:/DMRS Channel/SISO/Mix/
  <단일 family 또는 Mixed>/Random LS|MMSE|Real Channel/<allocation>/<Random 파일>.h5
  <단일 family 또는 Mixed>/Random Test LS|MMSE|Real Channel/<allocation>/<Random Test 파일>.h5
```

native 환경에 TDL/CDL의 Delay/Doppler 숫자를 임의로 붙이지 않습니다. 환경별 파일은 `condition_json`, `native_source_json`에 시나리오·주파수·위치·속도·구현 hash·소스 버전을 기록합니다. Random Mix의 각 환경은 `Source_Index`로 `Sources_JSON`·`Metadata_JSON`의 조건에 연결하고, `Random Channel Config.txt` 또는 `Random Test Config.txt`에서도 목록을 확인합니다. native 설정이 다르면 환경별 파일명의 Config hash가 달라집니다.

allocation의 P1은 포트 수이고 nslot1은 슬롯 수입니다. 여러 슬롯은 `nslot5`, `Slot 0-4` 형식입니다. CDM은 상위 경로 대신 속성에 기록합니다.

`RESUME=True`, `OVERWRITE=False`가 기본입니다.

- 완료된 같은 설정의 LS/MMSE/Real 세트는 재계산하지 않고 재사용합니다. 허용된 이전 TDL/CDL 구현 ID도 호환 검사를 거칩니다.
- 생성이 중단되면 세 수량이 공통으로 기록 완료한 행부터 재개합니다. 일부 파일만 있거나 설정이 다르면 임의로 이어 붙이지 않습니다.
- 직접 Mix는 `.work.h5`, `.shuffle.h5` 임시 파일을 거칩니다. 중단 후 재개할 수 있도록 생성 중에는 이 파일을 보존합니다. 완료 후 최종 세트와 설정·Pair를 검사하고 작업 파일을 정리합니다.
- 같은 경로에 다른 Mix 설정이 있으면 `OVERWRITE=False`에서 오류가 납니다. 기존 데이터를 보존할 실험은 별도 `DATA_ROOT`를 사용합니다. `OVERWRITE=True`는 의도적인 재생성에만 사용합니다.
- 파일 크기만으로 완료 여부를 판단하지 않습니다. `status='complete'`, `num_written`, split 범위, `generation_id`와 LS/MMSE/Real의 Pair를 확인합니다. LZF 압축률은 LS/MMSE/Real마다 다를 수 있습니다.

## 학습 파일과 Result 연결

현재 DNN·ELM·ELM_Linear·ELM_Distillation·Transformer는 TDL/CDL의 SISO Fix/Mix 데이터에 연결되어 있습니다. 모델 구조 설명은 [Layout_Guide.md](Layout_Guide.md)를 참고합니다.

| Data Generation 설정 | 학습 파일에서 맞출 설정 |
|---|---|
| `DATA_ROOT` | `DATA_ROOT` |
| SISO | `DATA_BRANCH = 'SISO'` |
| `DATA_MODE = 'Mix'` | `MIXED = True` |
| CDL-C 하나로 만든 Random Mix | `MIX_CHANNEL_TYPE = 'CDL-C'` |
| 여러 채널/프로파일을 함께 만든 Random Mix | `MIX_CHANNEL_TYPE = 'Mixed'` |
| Random Test 환경 목록 | `TEST_CHANNEL_TYPE_LIST`, `TEST_DELAY_LIST`, `TEST_DOPPLER_LIST`, `TEST_SNR_LIST` |

Mix 학습의 Train/Val 환경·수량·순서는 실제 Random 파일의 메타데이터를 따릅니다. 학습 파일의 `TRAIN_*_LIST`는 `MIXED=False`일 때 개별 Fix 학습을 선택하는 목록입니다. Test는 별도 Random Test 세트를 한 번 읽고 배치 추론한 뒤, `Source_Index`로 환경별 MSE/NMSE를 집계합니다. 학습 파일의 Test 목록은 이 세트에서 평가할 환경을 선택하며 해당 환경의 샘플을 모두 사용합니다. `MAX_SAMPLES_PER_CONDITION`을 지정하면 환경별 상한이 적용됩니다. 따라서 Mix Train SNR이 10·15·20 dB이고 Test SNR이 −10~20 dB인 현재 설정은 의도적인 서로 다른 목록입니다.

Test MSE는 `오차 제곱합 / 원소 수`, NMSE는 `오차 제곱합 / 정답 전력합`으로 환경별 계산하며 dB 값을 평균내지 않습니다. 기존 CE/Data·심볼·head·CE reuse·LS/LMMSE 지표와 Plot은 유지합니다. `Channel_NMSE.json`의 각 지표에 `mse`도 추가합니다.

기존 Train/Val Random의 설정·파일과 Mix ID가 같으면 가중치를 재학습할 필요 없이 새 Random Test로 평가할 수 있습니다. 기존 환경별 Test 파일은 이동·삭제하지 않습니다. 새 Random Test가 없으면 학습 로더가 기존 환경별 Test 파일로 돌아가는 것을 출력하며, 새 파일이 있으면 Random Test를 우선 사용합니다.

환경당 샘플 수가 기존 20,000에서 현재 190~191로 줄므로 기존 개별 Test 결과와 같은 평가 집합이 아닙니다. `Result_Plot.ipynb`는 새 Random Test 결과가 있으면 가장 최근의 동일 Test 데이터셋 ID 결과만 모아 비교합니다. 아직 새 Test로 재평가하지 않은 모델의 과거 결과를 같은 기준선에 섞지 않습니다. 기존 결과 파일 자체는 지우지 않습니다.

생성 데이터에는 항상 심볼 2·11이 모두 있습니다. 학습 파일의 `COMMON_CONFIG['current_symbols']`에서 `[2]`, `[11]`, `[2,11]` 중 관측 입력을 선택합니다. 예를 들어 현재 `[2]` 모델은 기존 `60×2` 파일에서 `60×1`만 읽습니다. 이를 위해 데이터를 다시 생성할 필요는 없습니다. 생성기의 여러 슬롯 지원과 현재 학습 파일의 1슬롯 실험 설정도 구분합니다.

**COST2100·QuaDRiGa는 생성·파형·추정·저장 경로까지만 연결되어 있습니다. 현재 학습/Result의 환경 검색은 TDL/CDL의 Delay·Doppler 키를 요구하므로 native 시나리오 자동 학습·Test·Plot까지 지원하는 상태는 아닙니다.** native 환경의 Scene/Config 키를 처리하도록 로더와 Result를 연결한 뒤 사용해야 합니다. 생성기에서 두 family를 선택하는 것만으로 학습 파일까지 자동 전환되지는 않습니다.

학습 결과는 데이터 경로와 별도로 다음 구조를 사용합니다. Data 격자는 실제로 모델에서 선택한 RS 심볼 수를 반영하며, 데이터 allocation의 `P1` 표기는 이 결과 격자에 넣지 않습니다.

```text
CE_Result/Mix/<채널>/<RS 격자_Data 격자_BWP120>/<모델>/<Layout>/
  Model/<모델 설정·Mix ID>/Model.pth
  Model/<모델 설정·Mix ID>/Losses.json
  Model/<모델 설정·Mix ID>/Environment_Loss/<CE 또는 Data 학습 단계>/
  Channel/<모델 설정·Mix ID>/CE_Train.h5
  Channel/<모델 설정·Mix ID>/Data_Train.h5       # Distillation=True일 때
  NMSE/<Test 환경>/<SNR·모델 설정>/...
  Comparison/<모델 설정>/...
```

LS 입력은 별도 CE 모델과 CE Train 저장을 생략합니다. 현재 학습 출력은 로딩·CE/Data 학습·저장·Test·선택적 Throughput의 tqdm을 표시하며, `leave=False, position=0`으로 완료된 bar를 남기지 않고 같은 줄을 재사용합니다. Layout 평가 후 CE/Data 환경 격자 그림과 SNR별 `CE / Data` 표를 모아 출력합니다. LS 입력의 CE 그림은 생략하고 표의 CE는 `x`로 표시합니다. 학습 출력 설정은 생성기의 진행 bar와 별개입니다.

새 학습의 `Environment_Loss`는 Random 파일의 환경 번호로 Train/Val을 따로 집계합니다. DNN/Transformer는 epoch별, ELM 계열은 Ridge 후보별입니다. `History.csv`를 매 단계마다 누적하고, 학습 단계 종료 후 `By_Environment`·`By_Doppler` CSV/PNG와 전체 요약을 저장합니다. 각 CSV에는 Train/Val MSE·샘플 수·전체 Loss 기여율이 있습니다. 역전파·Early Stopping·Ridge 선택에는 기존 전체 Mix Loss를 사용합니다.

Teacher 중간 정답이 Train에만 있는 Distillation 단계의 Val Loss는 공란입니다. 이미 가중치를 불러 Test만 하는 실행에서는 과거 epoch별 Loss를 새로 만들지 않습니다. 원본 Mix가 그대로 있고 가중치가 저장되어 있으면 최종 모델의 환경별 Train/Val MSE는 별도 재평가가 가능하지만, 기록하지 않은 과거 epoch별 값은 복원할 수 없습니다.

Loss CSV 기록과 가중치 저장 시점은 별개입니다. 현재 학습 코드는 CE 완료와 Data 전체 학습 완료 시 `Model.pth`를 저장합니다. 여러 Data head 중 일부가 완료되어도 전체 Data 학습이 끝나기 전까지 최신 Data 가중치가 파일에 저장된 것은 아닙니다.

## 파일·검사·출처

native Python 구현·검사·backup·vendor는 `DMRS_CSI_Compression/GPT Output/native_channel_implementation_20261103` 안에 모았습니다. 노트북은 해당 폴더의 Python 모듈을 읽습니다. 이 폴더를 삭제하면 native 생성기를 실행할 수 없습니다. 전역 Python 환경에 패키지를 설치하지 않았습니다.

- 실제 COST·QuaDRiGa 생성→DMRS 파형→LS/MMSE/Real 크기·Pair·SNR 간 Real 일치·재현 검사.
- Fix/Mix 저장, 완료 Test 재사용, 기존 TDL/CDL Mix 파일 유지 검사.
- 기존 TDL/CDL 출력 bitwise 회귀 검사.
- 경로손실·프로파일 상수·DS 스케일링·scatterer 기하 제약·정지/이동·MIMO/여러 슬롯·CPU/CUDA 검사.
- 91개 공식 QuaDRiGa config 테이블 일치 및 작은 생성 smoke census. **공식 모델 전체 적합성 검증이나 MATLAB 출력 대조를 대신하지 않습니다.**

위 생성기 검사는 2026-11-03에 수행한 소형 독립 검사 기록입니다. 2026-11-04에는 Random Test 생성·재개·Pair·재현과 전체 추론 후 환경별 평가를 추가 검증했습니다. Fix의 채널 출력과 기존 Train/Val의 Mix ID·파일은 유지되는 것을 확인했습니다. 이번 문서 갱신에서는 대량 데이터 생성이나 모델 학습을 실행하지 않았습니다. 기존 E 드라이브 데이터의 완료 상태는 각 HDF5의 상태 속성과 메타데이터로 확인합니다.

Random Test 생성·로더·평가·Result 수정과 소형 검증·백업은 `GPT Output/random_test_20261104`에 있습니다. 이 변경에서 대량 E 드라이브 생성·학습은 실행하지 않았습니다.

환경별 학습 Loss 기록과 출력 형식의 수정·검사 자료는 각각 `GPT Output/environment_loss_20261104`, `GPT Output/training_output_20261104`에 있습니다. 이 기능들은 학습 노트북 안에 포함되어 있어 검사 스크립트를 학습 중에 import하지 않습니다.

공식 소스:

- [COST2100](https://github.com/cost2100/cost2100), commit `bafca0b84faa2aa0441cb3f03f9dbd65771f43b3`
- [QuaDRiGa](https://github.com/fraunhoferhhi/QuaDRiGa), commit `277866650eb115adb5b3e8ac252b0d1df073596d`
- [NeoRadium channels](https://interdigitalinc.github.io/NeoRadium/html/source/API/Channels.html)
- [pyqd-channel 0.2.3](https://pypi.org/project/pyqd-channel/): pure-Python scenario/pathloss/SOS 부분을 사용했습니다. 이 패키지에 없는 SSF·이동·drifting 경로는 별도로 이식했습니다.

원본 라이선스·저자 정보와 이식 시 변경점은 구현 폴더의 LICENSE/NOTICE에 보존했습니다.
