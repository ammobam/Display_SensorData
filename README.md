# Display_SensorData

# 프로젝트 개요
## 프로젝트 목적
- 웨이퍼 식각 후 테스트 단계에서 센서 측정값의 1시간 평균 데이터를 분석하여 저수율 요인을 찾고자 함
- 테스트 과정에서 웨이퍼 상태의 반도체 칩의 불량여부를 선별 가능함
- 저수율 요인을 찾아 설계상의 문제점이나 제조상의 문제점을 발견해 수정 가능함

## 프로젝트 배경
### 웨이퍼 식각 공정
- 반도체 공정 중 하나
- TFT(박막트랜지스터)의 회로 패턴을 만들기 위해 웨이퍼의 필요한 부분만 남기고 불필요한 부분은 깎아내는 공정

### 저수율 웨이퍼가 발생하는 원인
- 반도체 공정의 각 프로세서에서 레시피(온도,압력,가공 시간 등)대로 작업이 이루어지지 않았기 때문

### 저수율 요인 파악의 필요성
- 공정 중 저수율 요인을 찾아내면 해당 프로세서의 집중적인 관리를 통해 고수율 웨이퍼의 생산효율을 극대화 할 수 있음
- 최적의 Etching 공정 레시피를 제공하고자 함

### 프로젝트 목표 목표
- 생산공정 중 양품/불량품에 큰 영향을 미치는 저수율 요인 5개를 찾아내는 것을 목표로 함


---
# 머신러닝을 이용한 중요 피처 찾기
- 목표 : 디스플레이 불량률에 가장 큰 영향을 미치는 피처 찾기
- 데이터 : 디스플레이 Etching 공정의 센서데이터 (.csv)
- 840 컬럼 x 8145개 행
- 각 공정은 사람이 검수함. 각 피처는 휴먼에러를 포함하고 있을 수 있음.
- 각 행은 센서 데이터의 1시간 평균값을 나타냄


## 데이터 불러오기
- 데이터 불러오기
- 원본데이터 보존을 위해 copy 데이터 만들어서 사용하기
- (선택) 생산라인 L, R 구분

## 데이터 전처리
- 결측치 처리
- 분산 0인 피처 제거
- 상관관계 높은 피처 제거
- VIF > 30 이상의 피처 제거

### 분산 0인 데이터 제거
- 분산이 0인 데이터를 제거하는 이유?
    - 어떤 피처의 분산이 0이라는 것은 그 피처의 데이터가 모든 행에 대해 거의 변하지 않은 것을 의미함
    - 어떤 경우에도 같은 값을 내는 컬럼이 불량률에 영향을 주고 있다고 보기 어려움
- 수행 방법
    - 방법1 : sklearn의 VarianceThreshold 사용
    - 방법2 : var() 사용
- 결과
    - 컬럼수 : 831->818


### 피처 간 상관관계 높은 피처 제거
- 상관관계 : 피처 간의 종속된 정도
- 상관관계가 높은 피처를 제거하는 이유?
    - 두 피처 간의 상관관계가 높다는 것은, 하나의 피처 값이 다른 피처의 값에 큰 영향을 주고있음을 의미함
    - 두 피처는 동일한 원인에 기인하여 변하는 것으로 추측할 수 있음
    - 이를 제거하지 않고 두면 사실상 같은 의미인 데이터가 모델링에 여러 번 반영됨
    - 사실상 종속관계에 있는 피처들이 모델링에 크게 기여하는 것과 같음
    - 모델링에 영향을 미치는 원인들이 모두 비슷한 중요도로 반영되게 하려면 종속성이 낮은 피처들만을 이용하여 모델을 만드는 것이 타당함
- 여기서는 피처 간 상관계수의 절대값이 0.9 이상인 경우를 종속된 것으로 봄

### (안함) VIF > 30 이상의 피처 제거
- VIF : Variation Inflation Factor, 분산팽창요인
    - 다중회귀분석 시 종속변수 Y를 제외하고 독립변수(피처)에 대해서만 판단함
    - 피처 사이에 회귀분석을 실시하여 결정계수(R2)가 높으면 다중공선성의 문제가 발생할 가능성이 높음
    - 피처의 특정 조합에서 회귀선의 설명력(결정계수)이 높으면 VIF 값이 커짐

- 이 데이터의 경우 컬럼 수가 많아서 VIF 기준을 30 정도로 잡아야 함
- VIF를 이용해서 컬럼을 제거하는 방법:
    - (1) VIF 계산
    - (2) VIF가 가장 큰 피처를 제외하고 다시 VIF 계산
    - (3-1) 가장 큰 VIF가
        - 이전의 VIF보다 커지거나, 
        - 무한으로 발산하는 경우에는
        - 제외했던 컬럼을 다시 포함하고 2순위의 컬럼을 제외하여 (1), (2) 반복
    - (3-2) 가장 큰 VIF를 확인한 결과 이전의 VIF보다 작고, 30 이상이면 (1), (2) 반복

- 안 하는 이유 :
    - 위와 같은 방법으로 피처를 제거한 결과, 남은 피처가 모델을 충분히 설명하지 못하는 것 같음
    - 순차적으로 진행되는 작업으로 수행 시간이 오래 걸리는 작업인 것에 비해 중요 피처가 삭제된 것 같아서 생략함
    - 대신 RFE를 수행해보기로 함
    
## 분류 모델링을 위한 전처리
- 325개의 컬럼으로 수행
- 피처 324개 + 레이블 1개

### PCA 수행하여 주성분 몇 개가 적절한지 조사
- 적당한 n_components 개수 정하기
- 방법1
    - 설명변수가 급감하는 때의 주성분 인덱스를 찾음
    - 이 경우 최소한의 주성분 개수로 전체 데이터의 경향을 설명할 수 있음
- 방법2
    - 설명변수 비율의 누적합이 0.95일 때의 주성분 개수로 선택할 수 있음
    - 이 경우 전체 데이터의 95%를 설명하기 위한 주성분 개수를 구하는 것과 같음
    
### 레이블 기준 설정
- 불량품/양품 나누는 기준


## 분류 모델 수행
- SVM
- Emsemble Method
    - voting : 여러 알고리즘을 사용하여 비교하고 가장 좋은 모델을 선정
        - hard voting(1, 0으로 리턴)
        - soft voting(확률로 리턴)
    - Bagging : 한 가지 알고리즘을 사용하고 데이터 샘플링을 달리하여 비교
    
    - Boosting :이전 모델 훈련 결과 나온 가중치를 다음 모델 훈련에 적용
    - Stacking
    
    
### Xgboost
- 파이썬 외부모듈
- 데이터의 구조를 변경해서 사용해야 함
- XGB Classifier 이용
- 참고 : https://statkclee.github.io/model/model-python-xgboost-hyper.html

### 평가지표
- 어떤 평가지표를 사용할지 고민해야 함
    - 정확도(accuracy) : TN + TP / 전체
    - 정밀도(precision) : TP / (FP + TP)
        - Pos로 예측한 것 중 실제 Pos였던 것
        - 양성예측도
        - Pos 예측 성능을 더 정밀하게 측정하기 위한 평가지표
        - FP를 낮추는 데 초점
    - 재현율(recall) : TP / (FN + TP)
        - 실제 Pos인 것 중 실제 Pos였던 것
        - 민감도, TPR(True Positive Rate)
        - Pos를 Neg로 판단하면 치명적인 경우 사용
        - FN을 낮추는 데 초점
    - F1 Score
        -  2pr/(p+r)
    - ROC 곡선
        - 이진분류의 예측 성능 측정에 사용
        - FP비율 - TP비율(recall) 곡선
- 이 데이터에서는 극소수의 불량품을 판정하지 못하는 것이 치명적임
    - 양품은 고객에게 배송되고, 불량품은 재검수 해보는 과정을 생각해보면
    - 불량품을 양품으로 판정한 경우, 재검수가 이뤄지지 않고 불량품이 고객에게 배송되면 브랜드 이미지에 타격이 가는 등의 치명적인 문제가 발생함
- 따라서 정확도는 적절치 않음. 재현율(recall)이 적절한 지표
    - FP가 높고, FN이 낮은지 중점적으로 살펴야 함
    - F1 Score가 적절할 것 같음

### RFE 클래스를 이용한 중요 피처 선정
- 참고링크
    - https://scikit-learn.org/stable/modules/generated/sklearn.feature_selection.RFE.html#sklearn.feature_selection.RFE
    - https://hongl.tistory.com/116
- RFE
    - recursive feature elimination (재귀적 피처 제거)
    - class sklearn.feature_selection.RFE
    - RFE(estimator, *, n_features_to_select=None, step=1, verbose=0, importance_getter='auto')
        - estimator : 훈련할 모델
        - n_features_to_select : 중요 피처 개수 설정(남길 피처 개수). float으로 설정하면 비율
        - step : 반복할 때마다 삭제할 피처 개수 설정. float으로 설정하면 비율
        - importance_getterstr : 삭제 기준. auto로 두면 회귀에서는 .coef, 분류에서는 feature_importance로 정함

