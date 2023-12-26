# MapServiceTest

#### 🚨네이버 맵과 Kakao API 접근키는 제거되어있습니다.
#### private 레포에 저장된 토이프로젝트를 옮긴 레포입니다.

## 요구사항 분석

#### 1. GeoJSON parsing

- 특정 쿼리 URL 호출해 응답값에 담긴 GeoJSON 형식 파일을 파싱, CoreData나 UserDefault에 저장해 사용한다
- 파싱된 데이터와 Naver Map SDK의 PolygonOverlay 기능을 사용하여 맵 위에 표시한다

#### 2. 지도 화면 내 카페 검색

- Kakao Local API 중 Category 검색 API 호출을 통해 데이터를 취득한 후 해당 카테고리 내 좌표를 이용해 Naver Map 위에 마커로 표시한다
- 마커는 선택가능해야하고, 선택된 마커는 이미지를 변경한다
- 마커를 선택했을 때 하단 화면(바텀시트)에 '''장소명''', '''전화번호''', '''주소''' 를 표기해야하며, 상세 정보 URL이 있는 경우 상세정보페이지로 넘어갈 수 있는 버튼을 구현해야 한다
- 버튼 탭을 하면 해당 URL을 웹뷰를 통해 오픈한다
- 지도를 탭하면 바텀시트를 종료시키고 선택되었던 마커는 일반 마커로 변경한다 -> SDK 내 지도 탭 핸들러 사용
- 마커 데이터는 지도가 이동 되는 시점에 호출하고, 호출이 시작되면 기존 마커는 제거한다
- 기존 선택된 마커가 있을 때, 해당 반경에 다시 들어갈 경우 마커는 선택된 마커로 표시한다 -> 바텀시트가 떠 있는 상태에서 지도를 움직일 때의 동작인 것으로 이해함

-----------------------
### 구상

#### 프로젝트 구성
- 의존성관리 : CocoaPods
- 형상관리 : Github, SourceTree

#### 작업분류
1. 프로젝트 생성 및 github repo 연동
2. cocoaPods 설치 및 필요 라이브러리 다운로드
3. 네이버, Kakao API 키 발급
4. API 응답값 모델 구현 및 통신 함수 구현
> Alamofire 라이브러리 사용
> Postman으로 응답값 확인 및 파싱에 사용될 Codable 모델 구현
5. 지도 마커 표시기능 구현
6. 마커 탭 이벤트 핸들러 구현
7. 바텀시트, 웹뷰 구현
8. GeoJSON 파싱 모델 구현
> GeoSwift 라이브러리 사용
>> 공식문서로부터 사용방법, 내부 구조, 구조 타입에 대해 분석
9. 폴리곤 오버레이 함수 구현
10. Rx 검토 및 리팩토링

##### 작업 기간 산정
1일차 분류 1-3번 (프로젝트 기본설정 및 각종 키 발급)

2,3일차: 분류 4-7번 (데이터 통신 및 응답값 분석, 모델 구현)

4일차: GeoSwift 라이브러리 사용 방법 습득 및 폴리곤 데이터 추출

5,6일차 : PolygonOverlay 구현 및 리팩토링

------------------------

#### CocoaPods(Library)
~~~cocoaPods
# 사용된 라이브러리

# Network
Alamofire', '5.6.0'

# Map, Coordinate
NMapsMap, '3.16.2'
GEOSwift, '10.1.0'

# Rx
RxSwift, '6.5.0'
RxCocoa, '6.5.0'

# UI
SnapKit, '5.6.0'
Then, '3.0.0'
~~~
_______________________
### 구현

#### 데이터 구조

1. 카카오 API: local/category 의 응답값은 codable Protocol을 채택한 구조체로 작성한다.
> 해당 응답값의 경우 새롭게 갱신되기 전까지 마커 탭, bottomSheet 업데이트에 등에 사용해야하므로 해당 화면의 ViewController나 ViewModel에서 가지고 있어야 한다.

2. GeoJson: GeoSwift 라이브러리를 사용해 데이터를 파싱하고, NaverMap SDK에서 사용하는 타입에 맞게 좌표 배열을 구성한다.
> '앱 첫 실행' 시 데이터를 저장하고, 이후 실행은 저장된 데이터로 PolygonOverlay를 그릴 데이터로 가공한다.
> > FileManager + UserDefaults 조합 사용
> > UserDefaults에 파일 경로를 저장, 앱 구동시 불러와 다시 parsing 및 데이터 추출

* 데이터 통신과 파싱은 completion 호출을 사용해 해당 블록의 모든 작업이 끝난 후 이후 동작을 할 수 있도록 구현한다.
------------------------

#### UI 구성 (iPhone 14 Pro Max)

#### PolygonOverlay

<img width="1181" alt="스크린샷 2023-09-27 02 19 18" src="https://github.com/YongJinLeee/MapTestApp/assets/40759743/4db5024b-a26d-49d2-b2c1-d7eb82890924">

#### Marker / BottomSheet

<img width="1247" alt="스크린샷 2023-09-27 02 54 32" src="https://github.com/YongJinLeee/MapTestApp/assets/40759743/e923d349-ace2-41a0-be78-cf8f878b61ed">


#### UI 설계


#### MainViewController (지도, 바텀시트)

- main.Storyboard를 중심으로 바텀시트는 BottomSheetView 코드기반 UI로 구현
- View 위에 MapContainer View를 배치하고, naverMapView 인스턴스를 생성, 속성을 설정 한 뒤 addSubView로 컨테이너에 적재
- BottomSheet의 경우 BottomSheetView 커스텀 뷰 클래스를 상속하고, 싱글톤으로 내부 데이터를 갱신할 수 있도록 구현

#### WebView(상세보기, MarkerDetailInfoWebViewController)

- MainViewController로 부터 전달받은 String Type URL을 옵셔널 바인딩으로 URL type으로 캐스팅한 후 웹뷰 로드
- 웹뷰 로드, 종료에 필요한 delegate 코드와 화면 요소만 구현
------------------------

### 코드 호출 구조


#### MainViewController
##### ViewDidLoad
1. GeoJSON 파싱 및 polygonOverlay (비동기)
> GeoJson 파일 존재 여부 확인 조건문 -> 데이터 다운로드 여부 결정 -> 데이터 파싱
~~~swift
if UserDefaults.geoJsonFilePath.isEmpty {
    Connection.getGeoJsonFile() { self.geoJsonParsing() }
  } else {
   geoJsonParsing()
}
~~~


>> GeoSwift 라이브러리를 이용해 GeoJSON.polygon(Polygon) 타입으로 파싱된 후
>> 네이버 PolygonOverlay 기능 중 NMGPolygon(ring: NMGLineString(points: [NMGLatLon]), interiorRings: [NMGLineString<NMGLatLng>]) 인스턴스 생성에 필요한 데이터로 가공(고차함수 사용)
>> * 제공된 폴리곤 데이터는 exterior, holes 로 구분되어 파싱됨
>> -> 폴리곤 그리기 위해 구현한 setPolygonOverlay 함수 호출
~~~swift
if let geoJSON = try? decoder.decode(GeoJSON.self, from: data!),
    case let .geometry(.polygon(polygonizedData)) = geoJSON,
    let holesData: [Polygon.LinearRing] = try? polygonizedData.holes,
    let exteriorData: Polygon.LinearRing = try? polygonizedData.exterior {
  
    let polygonRing = exteriorData.points.map({ NMGLatLng(lat: $0.y, lng: $0.x) })
    let polygonHoles = holesData.map { NMGLineString(points: $0.lineString.points.map({ NMGLatLng(lat: $0.y, lng: $0.x) })) }
            
    // 폴리곤 그리기
    setPolygonOverlay(exterior: polygonRing, interiorRings: polygonHoles)
    }
~~~
>> 폴리곤은 메인 스레드에서 async 방식으로 그려지도록 구현

2. 지도 및 마커 표시

> 파싱 조건문 수행 후 네이버 맵 로드, Rx binding 함수 호출
>> 네이버맵 로드 직후 mapViewCameraIdle delegate 함수 호출 되며 현재 지도컨텐츠의 중심점 좌표를 받아 카카오 로컬 카테고리 검색 API 호출
>> 호출 성공시 발생하는 completion() 콜을 통해 setMarkerOverlay 함수를 호출
>> -> 응답값 모델로 부터 각 배열요소의 좌표정보를 [NMGLatLng] 타입 배열로 추출하고 각 마커 데이터에 해당 좌표와 이벤트 핸들러를 삽입
~~~swift
// NMGLatLng 타입 좌표 배열
let markerCoord = documents.map {
    NMGLatLng(lat: Double($0.latitude) ?? 37.544186 , lng: Double($0.longitude) ?? 127.044127)
}
        
// 마커 위치, 이미지 데이터 인입
let cafeMarkers: [NMFMarker] = markerCoord.map {
  let markerElement = NMFMarker(position: $0)
  let detailData = self.documents.filter { $0.latitude == String(markerElement.position.lat) } // 각각의 마커 디테일 정보 추출
            
        markerElement.iconImage = detailData[0].id == selectedMarkerID ? selectedMarkerImage : markerImage
        markerElement.touchHandler = { (overlay: NMFOverlay) -> Bool in  // 터치 핸들러 등록
                
            if detailData.isEmpty == false {
                self.selectedMarkerID = detailData.first!.id
                self.bottomSheetUpdate(data: detailData.first!)
            }
                return true
            }
            return markerElement
    }
~~~
>> 이후 각 마커를 main.async 블록에서 지도에 표시하는 구간 호출

##### Marker touchHandler
- 마커 이벤트 핸들러
> 마커 이벤트 핸들러가 호출되면 해당 콜백 블럭 안에서 bottomSheetUpdate 함수가 호출되며, 해당 마커의 상세 정보에 따라 바텀시트 정보가 업데이트 됨
> > 마커가 탭 될때 해당 마커가 가지고 있던 document 타입 데이터에서 id를 추출해 전역 변수에 저장하며, 마커를 다시 그리는 과정에서 해당 아이디 값을 기준으로 마커 이미지를 결정하도록 구현

##### BottomSheetCustomView

- 상세보기 버튼 누를때 presentToMarkerDetailInfoWebViewController 함수 호출해 웹뷰를 overFullScreen 스타일의 모달로 표시

----------------------------

### 개선지점 고민

- 코드를 MVVM 패턴으로 작성하지 못한 부분이 가장 아쉬움
- 마커와 관련하여, 탭된 마커의 상세 정보를 매핑하는 과정에서 [Document] 배열에서 Latitude 값으로 필터링을 한 부분
    - 가게게 위치가 좌표상으로 겹치는 경우 배열의 element가 여러개 필터될 때 발생할 문제에 대한 대비를 할 수 없음 
    - 강제로 첫번째 배열의 정보만 가져오도록 했지만 더 나은 방법을 고려해봐야 함
    - 네이버 맵 SDK의 마커 모듈에서 제공하는 Dictionary 타입의 'userInfo' 프로퍼티를 사용해도 좋을 것 같다는 생각
