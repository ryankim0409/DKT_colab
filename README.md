

## 설치 방법
1. 운영체제와 GPU 사용 여부 등을 고려하여 `pytorch`를 설치한다.(https://pytorch.org/get-started/locally/) (Google Colab에서 사용시 생략 가능)
2. 터미널 혹은 명령 프롬프트에서 `pip install juno-dkt`를 실행한다.
3. `juno_dkt`를 import 하여 사용한다. *(pip 패키지 이름은 '-'로 연결되어 있으나, import 할때의 패키지 이름은 '_'로 연결되어 있음에 주의)*


### Result

![figure](figure.png)

Test score :  ROC AUC 0.73230  / Binary Cross Entropy 0.54384


## API Reference

### ItemEncoder Class
명목형으로 된 학생 id, 문항 id, 정오답 여부에 대한 데이터를 one-hot encoding한 후 `torch.Tensor`의 `list`로 바꿔준다.

**ItemEncoder(n_items=None, binary_only=False)**
* `n_items` _(int, Default:None)_ - 문항 수를 명시적으로 지정해야 할 경우 사용하고, 그렇지 않을 경우 기본값 `None`으로 설정하면 데이터에 의해 결정됨.
* `binary_only` _(bool, Default:False)_ - 문항의 정오답 여부가 이진 값(0 또는 1)인 것만 필터링 할 경우 True로 설정, 그렇지 않을 경우 0.3과 같은 실수 값은 `{0.3정답, 0.6오답}`과 같이 인코딩 됨.

**ItemEncoder.transform(self, students, items, answers)**

_students, items, answers 파라미터의 길이는 동일해야 함._
* `students` _(list of int or str)_ - 정수형 혹은 문자열로 된 학생들의 id 목록
* `items` _(list of int or str)_ - 정수형 혹은 문자열로 된 문항이나 작업의 id 목록
* `answers` _(list of float or int)_ 학생의 정오답 여부를 0~1로 나타낸 값의 목록


### DKT Class

**DKT(n_hidden, batch_size, lr, n_embedding=None, device='cpu')**

*Deep knowledge tracing (Piech, Chris, et al., 2015) 에 기초한 모델을 생성한다. Adam optimizer를 사용하여 학습된다.*
* `n_hidden` _(int)_ - 모델의 시계열 분석에 활용되는 LSTM의 은닉층 차원
* `batch_size` _(int)_ - 하나의 배치(batch)에 들어갈 데이터의 학생 수
* `lr` _(float)_ - Adam optimizer의 학습률
* `n_embedding` _(int, Default:None)_ - Compressed sensing 방식으로 구현할 때, 입력층의 one-hot vector가 인코딩되는 차원. 기본값인 `None`일 경우 compressed sensing을 사용하지 않고 one-hot vector가 직접 LSTM으로 입력됨.
* `device` _(str, Default:'cpu')_ - 학습 및 추론시 사용할 연산장치. 기본값인 `'cpu'`일 경우 cpu를 사용하고, `'cuda'`일 경우 그래픽카드 사용. _(cuda 버전에 알맞은 pytorch 설치 필요)_

**DKT.fit(batches, n_iter, test_set=None)**

_모델을 주어진 데이터로 학습시키고 평가한다._

* `batches` _(list of torch.Tensor)_ - ItemEncoder에 의해 변환된 학습 데이터
* `n_iter` _(int)_ 전체 데이터를 반복하여 학습할 횟수(epoch)
* `test_set` _(list of torch.Tensor, Default:None)_ - 훈련 과정애서 각각의 epoch이 끝난 후 지표를 평가할 테스트집합. 기본값인 `None`일 경우 평가를 생략함.

**DKT.roc_auc_score(batches)**

_데이터에 대한 ROC AUC(수신자 조작 특성 곡선의 밑넓이) 점수를 반환함_
* `batches` _(list of torch.Tensor)_ - ItemEncoder에 의해 변환된 데이터
* **return** _(float)_ - ROC AUC 점수

**DKT.bce_score(batches)**

_데이터에 대한 binary cross entropy 점수를 반환함_
* `batches` _(list of torch.Tensor)_ - ItemEncoder에 의해 변환된 데이터
* **return** _(float)_ - Binary cross entropy 점수

**DKT.y_true_and_score(batches)**

_데이터에 대해 참값과 예측값을 반환함. (flattened)_
* `batches` _(list of torch.Tensor)_ - ItemEncoder에 의해 변환된 데이터
* **return** `y_true, y_score` _(np.ndarray, np.ndarray)_ - 입력된 데이터의 정오답 참값과 예측값에 대해 나열된 `np.ndarray`형태의 데이터

**DKT.predict(data)**
* `data` _(list of torch.Tensor, or torch.Tensor)_ - ItemEncoder로 변환된 학생 데이터의 리스트, 혹은 개별 학생의 데이터.
* **return** `predictions` _(list of np.ndarray, or np.ndarray)_ 문항반응 예측의 변화를 나타낸 개별 `np.ndarray` 혹은 그 리스트. 각 학생에 대한 `np.ndarray`는 (학생의 응답 수, 전체 문항수)의 크기를 가진다.

**DKT.influence_matrix()**

_모델의 연관 행렬을 `np.ndarray` 형식으로 반환한다._
* **return** `matrix` _(np.ndarray)_ - 연관 행렬 $J$

**DKT.graph(item_encoder, method='conditional', use_label=False, threshold=0.1, pair_threshold=0)**

_모델의 지식 공간의 그래프를 `networkx.DiGraph` 형식으로 반환한다._
* `item_encoder` *(ItemEncoder)* - 데이터를 변환할 때 사용된 `ItemEncoder` 객체
* `method` _(str, Default:'conditional')_ - 그래프의 가중치를 부여하는 방법. 기본값인 `'conditional'`일 경우, 한 문항에서 다른 문항으로 전이될 확률(transition probability)에 모델이 추정한 점수를 곱하여 사용. `'transition'`일 경우, 전이 확률만을 사용.
* `use_label` _(bool, Default:False)_ - `ItemEncoder`에 입력된 원본 라벨을 사용할 것인지에 대한 여부. 기본값인 `False`로 설정시, 자연수 인덱스로 대체됨.
* `threshold` _(float, Default:0.1)_ - 그래프의 엣지를 생성하기 위한 임계값. 원본 논문에서는 Khan dataset에 대해 0.1을 사용함.
* `pair_threshold` _(float, Default:0)_ - 엣지가 생성되기 위해 문항의 {A, B} 순서쌍이 가져야 할 최소 빈도. 원본 논문에서는 Khan dataset에 대해 0.01을 사용함.
* **return** `g` _(networkx.DiGraph)_ - 가중치가 부여된 유향 그래프 객체

