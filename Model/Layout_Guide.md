# 1-slot CE / Data Layout

현재 실행 대상은 **기존 1-slot, 1-port 데이터**입니다. 모든 크기의 마지막 `×2`는 R/I입니다. RS는 Comb2의 60개 Subcarrier, Data는 RS 양 끝 사이의 119개 Subcarrier입니다. Real 정답은 보간하지 않은 실제 채널입니다.

## 공통 설정

```python
COMMON_CONFIG = {
    'n_slots': 1,
    'history_symbols': [2,11],  # 1-slot에서는 과거 슬롯 입력이 없으므로 사용하지 않음
    'current_symbols': [2],    # [2] / [11] / [2,11]
    'rs_subcarriers': list(range(0,119,2)),
    'data_subcarriers': list(range(119)),
    'data_symbols': list(range(3,11)),
}
```

`current_symbols=[11]`이면 아래 표의 기준 심볼 `a`가 11번이 됩니다. `[2,11]`이면 기준 입력은 `Symbol 2,11: 60×2×2`입니다. 입출력 크기를 따로 중복 입력하지 않고, Subcarrier·Symbol List로 계산합니다. `n_slots`는 기본 1로 고정하여 실행하며, 이번 수정·비교에서는 5-slot History 분기를 활성화하지 않았습니다.

CE Part의 `output_symbols='current'`는 공통 설정의 현재 심볼을 자동으로 따릅니다. `[2]`, `[11]`, `[2,11]`을 직접 지정할 수도 있습니다. 미래 심볼을 CE 출력에 추가하면 그 영역은 관측 채널 정제가 아니라 미래 채널 예측이 됩니다.

## 구조 설명표

아래는 `current_symbols=[2]`, `output_symbols='current'`, Data Symbol 3~10인 기본 설정입니다. `C2`는 학습 완료 후 고정한 CE 모델 출력, `L2`는 저장된 LS 채널, `Pj`는 모델이 예측한 Data Symbol j입니다.

| 구조 이름 | 구조 방식: 1~2줄 설명 | CE 모델 입력 → 출력 | Data 입력이 CE일 때: 입력 → 출력 | Data 입력이 LS일 때: 입력 → 출력 |
|---|---|---|---|---|
| **All** | CE 완료 후 현재 기준 심볼의 채널로 Data 3~10을 한 번에 출력. Data 모델 1개. | Symbol 2 LS: `60×1×2` → Symbol 2 Real: `60×1×2` | Symbol 2 C2: `60×1×2` → Symbol 3~10: `119×8×2` | **CE Part 생략.** Symbol 2 L2: `60×1×2` → Symbol 3~10: `119×8×2` |
| **Half** | 3~6과 7~10을 두 모델로 나눔. 기준이 2·11 둘 다이면 앞 모델은 2번, 뒤 모델은 11번 사용. | 기본은 위와 동일. 둘 다 관측 시 `60×2×2` → `60×2×2`. | 기본: C2 `60×1×2` → 3~6 `119×4×2`; 별도 C2 `60×1×2` → 7~10 `119×4×2` | **CE Part 생략.** 위 입력의 C2 대신 L2 사용. 두 기준일 때는 각각 L2/L11 사용. 출력 크기는 동일. |
| **One** | 심볼별 보정 모델 8개가 동일한 기준 입력을 받아 각자 한 심볼을 출력. 순차 의존 없음. | 기본은 All과 동일. | Symbol 2 C2 `60×1×2` → Symbol j `119×1×2`, j=3~10, 모델 8개 | **CE Part 생략.** L2 `60×1×2` → 각 Symbol j `119×1×2`, 모델 8개 |
| **ARSeparate** | 이전 예측 하나로 다음 심볼을 예측. 단계마다 다른 모델 8개. | 기본은 All과 동일. | C2 `60×1×2` → P3 `119×1×2`; 이후 Pj `119×1×2` → P(j+1) `119×1×2` | **CE Part 생략.** 첫 단계만 L2 `60×1×2` → P3. 이후 이전 예측을 사용하는 방식·출력은 동일. |
| **ARShared** | 첫 심볼 전용 모델 1개 + 이후 공유 모델 1개. 공유 모델을 Symbol 4~10에 7번 호출. | 기본은 All과 동일. | 첫 C2 `60×1×2` → P3; 공유 Pj `119×1×2` → P(j+1), 7회 | **CE Part 생략.** 첫 입력 C2를 L2로 교체. 이후 공유 모델의 입력·출력은 동일. |
| **ARAnchorSeparate** | 기준 CE를 매 단계 유지하고 이전 예측 하나와 함께 입력. 단계별 모델 8개. | 기본은 All과 동일. | 첫 C2 → P3. 이후 **Symbol 2 C2 `60×1×2` + Symbol j Pj `119×1×2`** → Symbol j+1 `119×1×2` | **CE Part 생략.** 고정 기준 C2 대신 L2. 이전 예측 Pj와 결합하는 방식·출력은 동일. |
| **ARAnchorShared** | 위 기준+이전 예측 입력을 사용하되 첫 모델 1개와 이후 공유 모델 1개. | 기본은 All과 동일. | 첫 C2 → P3; 이후 C2 `60×1×2` + Pj `119×1×2` → P(j+1), 공유 7회 | **CE Part 생략.** 기준으로 L2 사용. 공유 구조·출력은 동일. |
| **Joint** | 별도 CE 모델 없이 LS → CE 정답과 Data 정답을 한 모델에서 동시 출력. Data가 출력 CE를 입력으로 다시 받지 않음. | 별도 CE Part 없음. 같은 모델의 출력에 CE 영역 포함. | 이 구조는 `input_by_layout={'Joint':'LS'}`로 선택. | LS 입력 Symbol 2 `60×1×2` → RS Symbol 2 `60×1×2` **+** Data Symbol 3~10 `119×8×2`. CE 출력 심볼을 둘 다 지정하면 RS 출력은 `60×2×2`. |

Transformer의 ARAnchor/ARC 혼합 입력은 Subcarrier 축을 119에 맞추고, RS Token의 앞 60개 Feature에 실제 관측을 넣습니다. 이는 패딩이며 보간이 아닙니다. R/I별 Token 구분은 유지합니다. DNN/ELM 계열은 실제 값만 flatten하여 입력합니다.

## 폐기물 구조
| **ARCSeparate** | 기준과 이전 단계의 **전체 예측 구간**을 누적. 각 단계가 전체 구간을 다시 예측. 모델 8개. | 기본은 All과 동일. | C2 `60×1×2` → P3 `119×1×2`; C2+P3 → P3~4 `119×2×2`; …; C2 `60×1×2`+P3~9 `119×7×2` → P3~10 `119×8×2` | **CE Part 생략.** 고정 기준으로 L2 사용. 입력의 나머지는 이전 Student 예측이며 누적 출력은 동일. |
| **ARCShared** | ARC의 최대 크기 모델 하나를 8단계에서 공유. 미생성 입력은 정규화 공간의 0, 미생성 출력은 Loss/LS에서 제외. | 기본은 All과 동일. | 유효 입력은 ARCSeparate와 동일. 최대 입력 `(60+119×7)×2`, 최대 출력 `119×8×2`. 매 단계 유효 prefix를 사용. | **CE Part 생략.** 기준은 L2. 최대 크기·공유 호출·출력 prefix는 동일. |

## 학습 순서와 정답 누출 방지

- **CE 입력:** CE Train 학습 → 최적 CE 고정 → `CE_Train.h5`에 Train 출력만 저장 → Data Train에서 해당 출력을 읽음. Val CE는 고정된 CE 모델로 계산하며, Val/Test 출력 파일은 만들지 않습니다.
- **LS 입력:** CE 학습, CE 가중치, CE Train 파일을 모두 생략합니다. Data 정답은 동일한 원본 Real 채널입니다.
- **Separate AR/ARC:** 앞 단계의 학습된 Student 출력으로 다음 단계 Train/Val 입력을 만듭니다.
- **Shared AR/ARC:** 고정된 Real 이전 채널/prefix로 학습 쌍을 구성합니다. Validation/Test는 Student 예측만 순차 입력합니다. ELM 공유형은 이 고정 학습 쌍에 대한 LS이며, 자기 예측을 학습 입력으로 반복 재회귀하는 방식이 아닙니다.
- **ELM_Distill:** 같은 입력, 같은 Layout, 같은 샘플의 Transformer Train 중간 출력으로 Stage 1을 학습합니다. Stage 2 정답은 Real 또는 같은 단계의 Teacher 출력. Val/Test는 실제 Student 추론입니다. `Joint`는 기존 1-slot Distillation 대상에 포함되지 않습니다.

## CE / Data 크기 설정

각 학습 파일에서 **CE_CONFIG['model']**과 **DATA_CONFIG['model']**을 별도로 수정합니다. `model_by_layout`은 해당 Layout에서만 크기를 바꿉니다.

```python
# DNN/ELM 예시: CE와 Data를 독립적으로 설정
CE_CONFIG['model'] = {'hidden':[512]}
DATA_CONFIG['model'] = {'hidden':[1024]}
DATA_CONFIG['model_by_layout']['ARCSeparate'] = {'hidden':[256]}

# Transformer 예시
CE_CONFIG['model'] = {'embed_dim':64,'depth':1,'mlp_dim':128,'num_heads':4,'drop':0.1}
DATA_CONFIG['model'] = {'embed_dim':128,'depth':4,'mlp_dim':256,'num_heads':4,'drop':0.1}

# CE를 생략하는 Data 전용 비교
DATA_CONFIG['input'] = 'LS'
```

ELM_Linear의 기본 `hidden=[]`는 비선형 은닉층 없이 Linear LS입니다. ELM_Distill의 CE 크기는 고정 Teacher CE 크기이며, Data의 `stage1_hidden/stage2_hidden`과 선택 Encoder를 별도로 설정합니다. 추가 은닉층 기본값은 빈 List로, Teacher 중간 정답을 잇는 두 Linear를 사용합니다.

## 새 저장 경로

```text
CE_Result/
  Mix/                         # 또는 Fix
    CDL-C/
      RS60x1_Data119x8_BWP120/  # [2,11] 입력이면 RS60x2. Port=1 고정, P1 제외
        DNN/                   # ELM / ELM_Linear / ELM_Distill / Transformer
          ARCSeparate/         # 학습 순서 접미사 없음
            Model/<CE크기_Data크기_InputCE_or_LS_슬롯수_ConfigID_MixID>/Model.pth
            Channel/<동일 case>/CE_Train.h5
            Channel/<동일 case>/Data_Train.h5   # Distillation=True인 경우만
            NMSE/<Test 환경>/<SNR_case>/Results.json
            NMSE/<Test 환경>/<SNR_case>/Settings.json
            NMSE/<Test 환경>/<SNR_case>/Model_NMSE.csv
            Comparison/<case>/<환경>/rs.png, data.png
```

Fix는 Layout 아래에 학습 환경 폴더를 추가하며, Model/NMSE 이하 SNR 구분은 기존 양식을 따릅니다. CE 크기, Data 크기, 입력 출처, 관측 심볼/좌표, 슬롯 수, Teacher 선택이 Config ID에 포함되므로 서로 덮어쓰지 않습니다. 원본 Dataset의 기존 `P1` 폴더명은 변경하지 않습니다.

기존 1-slot 결과·가중치는 새 Mix/Fix/채널/격자/모델/Layout 경로로 이동했습니다. 기존 case명과 가중치·NMSE 값은 유지하며, 5-slot 결과와 원본 Dataset 경로는 유지합니다. Result는 기존 완료된 1-slot 결과도 학습 순서 접미사 없이 표시합니다. 호환 가능한 기존 2번 기준 All/AR/ARC 가중치는 새 경로에서 재사용하며, 구조·입력·정규화가 맞지 않는 가중치는 임의로 로드하지 않습니다. 새 `Model.pth`가 이미 완료 상태이면 재학습 없이 Test합니다.

## Result 설정

`CURRENT_SYMBOLS=[2]`, `[11]`, `[2,11]`로 동일 관측 조건을 선택합니다. `DATA_INPUT_LIST=['CE','LS']`로 입력 출처를 비교하고, CE/Data 크기는 `CE_MODEL_CONFIG_FILTER`와 `DATA_MODEL_CONFIG_FILTER`에서 각각 선택합니다. 같은 모델의 여러 크기는 Dictionary의 중복 키 대신 List로 설정합니다. 현재 5-slot 결과는 읽기 단계에서 제외합니다.

기본 출력은 전체 모델 CE 그림, 전체 모델 Data 그림, 모델/입력 출처별 Layout 비교 그림입니다. 같은 모델의 여러 크기도 한 그림에서 범례로 구분합니다. 각 그림의 범례는 아래에 따로 표시합니다. CE 그림은 선택한 동일 RS 심볼끼리 비교하므로, 미래 11번을 포함한 CE 합산값과 현재 2번만의 값을 섞지 않습니다. CE 재사용/2D 보간 표시 설정은 기존 On/Off 선택을 유지합니다.

Throughput/BLER는 학습 파일 마지막 Cell의 `THROUGHPUT=False` 분기로 분리돼 있습니다. True일 때 저장된 가중치를 직접 실행하여 원본 Test 채널에서 CRC/LDPC/복조를 평가합니다. Result 마지막 분기는 저장된 Throughput/BLER를 읽어 표시합니다.
