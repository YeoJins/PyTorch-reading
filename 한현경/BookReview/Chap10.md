📕 <b> [10장] 여러 데이터 소스를 통합 데이터셋으로 합치기</b>

- 해야할 일: 원본 CT 스캔 데이터와 데이터에 달아놓은 에노테이션 목록으로 훈련 샘플을 만드는 것.
### 10.1 원본 CT 데이터 파일
- CT데이터는 두 종류로 나뉜다:
    1. 메타데이터 헤더 정보가 포함된 .mhd 파일
    2. 3차원 배열을 만들 원본 데이터 바이트를 포함하는 .raw 파일
- 각 파일의 이름은 `시리즈 UID`라고 불리는 CT스캔 단일 식별자로 시작한다. 
    - 예를 들어, `시리즈 UID 1.2.3`의 경우 `1.2.3.mhd`와 `1.2.3.raw` 두 가지의 파일이 있다.
- [1]`Ct 클래스`를 사용해서 다음과 같은 작업을 수행한다:
    - 두 파일을 읽어 3차원 배열을 만든다.
    - 환자 좌표계를 배열에서 필요로 하는 인덱스, 행, 열 좌표로 바꿔주는 변환 행렬을 만든다.
- [2]`LUNA annotation 데이터`를 읽어들인다:
    - annotation 종류: 결절의 좌표 목록, 악성 여부, CT스캔의 시리즈 UID
    - 결절 좌표가 좌표계 변환 정보를 거치면 결절의 중심에 해당하는 복셀의 인덱스, 행, 열 정보가 생긴다.
- [3](I, R, C)좌표를 사용해 CT데이터의 부분적인 3차원 단면을 얻어 모델에 대한 입력으로 사용한다.
- [4]훈련 샘플 튜플의 나머지를 구성한다.
    - 샘플 배열, 결절의 상태 플래그, 시리즈 UID, CT리스트 내 샘플의 인덱스 등
    - 이 때 생성되는 튜플은 원본 데이터를 표준 구조의 파이토치 텐서로 변환하는 과정의 마지막 부분이다.
- 데이터를 제한하거나 노이즈가 끼지 않게 해야 하며, 지나치게 걸러내어 중요한 값을 상실해서도 안된다.
- 정규화 이후 데이터값의 범위가 적절한지도 확인해야 한다.
- 데이터에 극단적으로 튀는 값들이 많은 경우, 이상값을 제거하는 것이 좋다.
- 입력 변환을 위해 직접 고안한 알고리즘을 사용할 수도 있다. (=feature engineering)

### 10.2 LUNA 에노테이션 데이터 파싱
-데이터 로딩
    - LUNA에서 제공하는 CSV 파일을 먼저 파싱하여 각 CT스캔 중 관심 있는 부분을 파악하자.
    - CT정보가 포함된 `candidates.csv`파일에 좌표 정보, 해당 좌표 지점이 결절인지 여부, CT스캔에 대한 고유 식별자 등이 존재함.
        - 이 데이터를 사용해 전체 후보 리스트를 만들자.
        - 추후에 훈련 데이터셋과 검증 데이터셋으로 나누자.

### 10.2.1 훈련셋과 검증셋
- 모든 지도학습(Supervised Learning)에서는 데이터를 훈련 셋(Training Set)과 검증 셋(Validation Set)으로 나눈다. 
- 검증셋과 훈련셋은 모두 현실 세계의 데이터를 사용해야 함.
- ...

### 10.3 개별 CT 스캔 로딩
- <b>디스크에서 CT 데이터를 얻어와 파이썬 객체로 변환 후 3차원 결절 밀도 데이터로 사용할 수 있게 만드는 작업</b>
- `결절 에노테이션 정보`란, 원본 데이터에서 얻어내고자 하는 영역에 대한 맵이다. 이 맵을 사용하여 관심있는 부분을 추출하려면, 먼저 데이터를 주소로 접근 가능하게 해야 한다.
- LUNA는 우리가 사용할 데이터를 사용하기 쉬운 `MetalO`포맷으로 변환해뒀다. 이 책에서는 데이터 파일 포맷을 블랙박스로 간주하고 친숙한 넘파이 배열로 읽어들이기 위해 `SimpleITK`를 사용한다.
- 특정 CT스캔을 식별하기 위해 현재 CT가 만들어질 때 부여된 `시리즈 인스턴스 UID`를 사용 중이다. DICOM은 개별 DICOM 파일, 여러 파일 그룹, 처리 과정 등에 단일 식별자(UID)를 사용하고 있다. 여기에서는 UID를 CT스캔을 참조 시 사용할, ASCII문자열로 만든 단일 키로 취급한다. 
    - 공식적으로  DICOM의 UID는 0~9와 점(.)을 사용하지만 경우에 따라서 16진수나 스펙에 없는 값으로 바꿔 사용하기도 한다.

### 10.3.1 하운스필드 단위
- `__init__` 메소드 작성 시 `ct_a`값 지워주기
- CT 스캔 복셀은 `하운스필드 단위`로 표시함
    - 공기: -1000HU
    - 물: 0HU
    - 뼈: +1000HU
- 이 프로젝트의 경우, 환자의 몸 외부는 모두 공기이므로 -1000HU이하 값을 버리고, 뼈나 금속 이식물에 대한 밀도값도 필요 없으므로 +1000HU이상인 값도 버린다. 

```python
ct_a.clip(-1000, 1000, ct_a)
```

- 우리가 관심 있는 종양의 경우 대체로 0HU근처이다.
- 데이터에서 이와 같이 이상값을 제거하는 과정은 매우 중요하다!
- 이렇게 만들어진 값을 `self`에 할당한다.

```python
self.series_uid =  series_uid
self.hu_a = ct_a
```

### 10.4 환자 좌표계를 사용해 결절 위치 정하기
- 통상적으로 딥러닝에서는 고정된 크기의 입력을 요구한다.(입력 뉴런 수가 정해져 있기 때문에)
- 따라서 입력으로 사용할 고정된 크기의 결절 후보를 담을 배열을 만들어야 한다. 모델의 훈련에는 CT스캔에서 깔끔하게 잘라낸 중심이 잘 잡힌 후보 데이터를 사용해 모델이 수행할 작업을 수월하게 해주자.
- 환자 좌표계
    - 이제까지 읽어들인 데이터는 복셀(voxel)이 아니라 밀리미터(mm)단위이다.
    - 따라서 mm기반 (X,Y,Z)좌표계로부터 voxel기반 (I,R,C)좌표계로 좌표를 변환해야 함.
-mm를 voxel로 변환하기
    - (X,Y,Z)좌표는 코드에서 `_xyz`로 끝나는 이름을 가진다.
    - (I,R,C)좌표는 코드에서 `_irc`로 끝나는 이름을 가진다.
    