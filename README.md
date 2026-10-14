# birdd
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>유적지 탐방 카카오맵 서비스</title>
    <!-- Tailwind CSS CDN (디자인을 예쁘고 간편하게 꾸미기 위해 사용합니다) -->
    <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
    <style>
        body {
            font-family: 'Malgun Gothic', sans-serif;
        }
    </style>
</head>
<body class="bg-gray-100 h-screen flex flex-col overflow-hidden">

    <!-- 상단 헤더 영역 -->
    <header class="bg-amber-700 text-white px-6 py-4 shadow-md flex items-center justify-between shrink-0">
        <h1 class="text-xl font-bold flex items-center gap-2">
            🏛️ 유적지 탐방 카카오맵
        </h1>
        <span class="text-xs bg-amber-800 text-amber-100 px-3 py-1 rounded-full font-medium">
            국립박물관 & 문화유산 탐색 서비스
        </span>
    </header>

    <!-- 메인 컨테이너: 좌우 2분할 레이아웃 -->
    <main class="flex-1 flex flex-col md:flex-row overflow-hidden">
        
        <!-- 왼쪽 영역: 탭 버튼 및 결과 리스트 -->
        <section class="w-full md:w-[400px] lg:w-[450px] bg-white border-r border-gray-200 flex flex-col shrink-0 z-10 shadow-lg">
            
            <!-- 탭 버튼 영역 (국립박물관 / 문화유산) -->
            <div class="grid grid-cols-2 gap-2 p-4 bg-gray-50 border-b border-gray-200">
                <button 
                    id="tab-museum" 
                    onclick="switchTab('museum')"
                    class="py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-amber-700 text-white cursor-pointer"
                >
                    🗞️ 국립박물관
                </button>
                <button 
                    id="tab-heritage" 
                    onclick="switchTab('heritage')"
                    class="py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-white text-gray-700 border border-gray-300 hover:bg-gray-100 cursor-pointer"
                >
                    🌟 문화유산
                </button>
            </div>

            <!-- 검색바 (키워드 직접 검색 겸용) -->
            <div class="p-4 border-b border-gray-100">
                <form id="search-form" onsubmit="searchByKeyword(event)" class="flex gap-2">
                    <input 
                        type="text" 
                        id="search-input" 
                        placeholder="지역명 또는 키워드 입력 (예: 경주, 서울)" 
                        class="flex-1 px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-amber-700 text-sm"
                    >
                    <button 
                        type="submit" 
                        class="bg-amber-700 hover:bg-amber-800 text-white font-semibold px-4 py-2 rounded-lg text-sm transition-colors cursor-pointer"
                    >
                        검색
                    </button>
                </form>
            </div>

            <!-- 장소 결과 리스트 영역 -->
            <div id="place-list-container" class="flex-1 overflow-y-auto divide-y divide-gray-100 p-2">
                <div class="text-center text-gray-400 py-20 text-sm">
                    상단 탭을 선택하거나 위치를 허용하면 주변 유적지를 불러옵니다!
                </div>
            </div>
        </section>

        <!-- 오른쪽 영역: 카카오 지도 표시 영역 -->
        <section class="flex-1 relative bg-gray-200">
            <!-- 카카오 지도가 그려질 DIV 요소 -->
            <div id="map" class="w-full h-full"></div>
            
            <!-- 현재 위치로 이동하는 버튼 -->
            <button 
                onclick="moveToCurrentLocation()"
                class="absolute bottom-6 right-6 bg-white hover:bg-gray-50 text-gray-700 font-bold px-4 py-3 rounded-full shadow-lg border border-gray-200 text-sm flex items-center gap-2 z-10 transition-transform active:scale-95 cursor-pointer"
                title="내 위치로 이동"
            >
                🧭 내 위치
            </button>
        </section>

    </main>

    <!-- 카카오 지도 JavaScript SDK 로드 -->
    <script type="text/javascript" src="//dapi.kakao.com/v2/maps/sdk.js?appkey=452ca8db7c18b33eb094278b28cc9239&libraries=services&autoload=false"></script>
    
    <script>
        // 전역 변수 선언
        let map = null;               // 지도 객체
        let ps = null;                // 장소 검색 서비스 객체
        let markers = [];             // 지도에 표시된 마커 배열
        let currentPosition = null;   // 현재 위치 좌표
        let currentTab = 'museum';    // 현재 활성화된 탭 ('museum' 또는 'heritage')
        let currentInfowindow = null; // 현재 열려있는 팝업(인포윈도우)
        let currentKeyword = '';      // 키워드 검색 시 사용되는 검색어

        // 페이지 로드 완료 시 SDK 초기화 실행
        window.addEventListener('DOMContentLoaded', () => {
            kakao.maps.load(() => {
                initMap();
            });
        });

        /**
         * 1. 지도 초기화 및 위치 권한 요청 함수
         */
        function initMap() {
            const mapContainer = document.getElementById('map');
            
            // 기본 위치 (서울 시청)
            let defaultLat = 37.566826;
            let defaultLng = 126.978656;

            const mapOption = {
                center: new kakao.maps.LatLng(defaultLat, defaultLng),
                level: 5 // 지도 확대 레벨 5
            };

            // 지도 생성
            map = new kakao.maps.Map(mapContainer, mapOption);
            
            // 장소 검색 객체 생성
            ps = new kakao.maps.services.Places();

            // 위치 권한 요청 (Geolocation)
            if (navigator.geolocation) {
                navigator.geolocation.getCurrentPosition(
                    (position) => {
                        const lat = position.coords.latitude;
                        const lng = position.coords.longitude;
                        
                        currentPosition = new kakao.maps.LatLng(lat, lng);
                        map.setCenter(currentPosition);
                        
                        // 현재 위치에 파란 점 마커 표시
                        displayCurrentLocationMarker(currentPosition);

                        // 위치 확인 후 기본 탭(국립박물관) 데이터 검색 실행
                        searchPlacesByTab();
                    },
                    (error) => {
                        console.warn('위치 권한을 얻지 못했습니다. 기본 위치로 설정합니다.', error);
                        currentPosition = new kakao.maps.LatLng(defaultLat, defaultLng);
                        displayCurrentLocationMarker(currentPosition);
                        searchPlacesByTab();
                    },
                    {
                        enableHighAccuracy: true,
                        timeout: 5000,
                        maximumAge: 0
                    }
                );
            } else {
                currentPosition = new kakao.maps.LatLng(defaultLat, defaultLng);
                displayCurrentLocationMarker(currentPosition);
                searchPlacesByTab();
            }
        }

        /**
         * 2. 현재 위치를 파란 점(커스텀 오버레이)으로 표시하는 함수
         */
        function displayCurrentLocationMarker(locPosition) {
            // 파란 점 디자인의 HTML 콘텐츠 생성
            const content = document.createElement('div');
            content.style.cssText = `
                width: 20px;
                height: 20px;
                background-color: #3b82f6;
                border: 3px solid #ffffff;
                border-radius: 50%;
                box-shadow: 0 0 10px rgba(0, 0, 0, 0.3);
            `;

            const customOverlay = new kakao.maps.CustomOverlay({
                position: locPosition,
                content: content,
                map: map,
                zIndex: 1
            });
        }

        /**
         * 3. '내 위치' 버튼 클릭 시 지도 이동 함수
         */
        function moveToCurrentLocation() {
            if (currentPosition) {
                map.setCenter(currentPosition);
                map.setLevel(5);
            } else {
                alert('현재 위치 정보를 아직 불러오지 못했습니다.');
            }
        }

        /**
         * 4. 탭 전환 처리 함수
         */
        function switchTab(tabName) {
            currentTab = tabName;
            currentKeyword = ''; // 탭 전환 시 직접 입력한 키워드 초기화
            document.getElementById('search-input').value = '';

            const btnMuseum = document.getElementById('tab-museum');
            const btnHeritage = document.getElementById('tab-heritage');

            // 탭 버튼 스타일 전환
            if (tabName === 'museum') {
                btnMuseum.className = "py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-amber-700 text-white cursor-pointer";
                btnHeritage.className = "py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-white text-gray-700 border border-gray-300 hover:bg-gray-100 cursor-pointer";
            } else {
                btnHeritage.className = "py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-amber-700 text-white cursor-pointer";
                btnMuseum.className = "py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-white text-gray-700 border border-gray-300 hover:bg-gray-100 cursor-pointer";
            }

            // 새 탭에 맞춰 장소 검색 실행
            searchPlacesByTab();
        }

        /**
         * 5. 상단 입력창 키워드 검색 처리 함수
         */
        function searchByKeyword(event) {
            event.preventDefault();
            const inputVal = document.getElementById('search-input').value.trim();
            if (!inputVal) {
                alert('검색어를 입력해주세요!');
                return;
            }
            currentKeyword = inputVal;
            searchPlacesByTab();
        }

        /**
         * 6. 현재 선택된 탭과 조건에 맞춰 장소 검색 요청 수행
         */
        function searchPlacesByTab() {
            // 기존 열려있는 인포윈도우 닫기
            if (currentInfowindow) {
                currentInfowindow.close();
            }

            // 기존 마커 제거
            removeMarkers();

            let searchQuery = '';
            if (currentKeyword) {
                // 사용자가 키워드를 직접 입력한 경우 조합 검색 (예: "경주 국립박물관" 또는 "서울 유적건조물")
                if (currentTab === 'museum') {
                    searchQuery = currentKeyword + ' 국립박물관';
                } else {
                    searchQuery = currentKeyword + ' 유적건조물';
                }
            } else {
                // 탭 기본 검색어 설정 (기획서 조건 반영)
                if (currentTab === 'museum') {
                    searchQuery = '국립박물관';
                } else {
                    searchQuery = '유적건조물';
                }
            }

            // 현재 지도 중심 좌표 기준으로 주변 검색 수행 (useMapBounds 옵션 활용)
            const searchOptions = {
                location: map.getCenter(),
                radius: 20000 // 반경 20km 내 검색
            };

            ps.keywordSearch(searchQuery, placesSearchCB, searchOptions);
        }

        /**
         * 7. 장소 검색 결과 콜백 함수
         */
        function placesSearchCB(data, status, pagination) {
            const listContainer = document.getElementById('place-list-container');

            if (status === kakao.maps.services.Status.OK) {
                displayPlaces(data);
            } else if (status === kakao.maps.services.Status.ZERO_RESULT) {
                listContainer.innerHTML = `
                    <div class="text-center text-gray-400 py-20 text-sm">
                        주변에 검색된 장소가 없습니다. 다른 지역을 검색해보세요!
                    </div>
                `;
            } else if (status === kakao.maps.services.Status.ERROR) {
                alert('검색 중 오류가 발생했습니다.');
            }
        }

        /**
         * 8. 검색 결과 리스트 및 마커 화면 출력 함수
         */
        function displayPlaces(places) {
            const listContainer = document.getElementById('place-list-container');
            listContainer.innerHTML = ''; // 기존 리스트 초기화

            const bounds = new kakao.maps.LatLngBounds();

            for (let i = 0; i < places.length; i++) {
                const place = places[i];
                const placePosition = new kakao.maps.LatLng(place.y, place.x);

                // 마커 생성 (탭별 아이콘 분기: 국립박물관은 🗞️, 문화유산은 🌟)
                const marker = addCustomMarker(placePosition, currentTab, i);
                
                bounds.extend(placePosition);

                // 팝업(인포윈도우) 생성 콘텐츠
                const infowindowContent = `
                    <div style="padding:10px; font-size:12px; width:200px; line-height:1.4;">
                        <strong style="font-size:13px; color:#b45309;">${place.place_name}</strong><br>
                        <span style="color:#666;">${place.road_address_name || place.address_name}</span>
                    </div>
                `;
                const infowindow = new kakao.maps.InfoWindow({
                    content: infowindowContent,
                    removable: true
                });

                // 마커 클릭 이벤트: 마커 클릭 시 인포윈도우 표시
                kakao.maps.event.addListener(marker, 'click', () => {
                    if (currentInfowindow) {
                        currentInfowindow.close();
                    }
                    infowindow.open(map, marker);
                    currentInfowindow = infowindow;
                });

                // 왼쪽 리스트 아이템 생성
                const itemEl = document.createElement('div');
                itemEl.className = 'p-4 hover:bg-amber-50 cursor-pointer transition-colors border-b border-gray-100';
                itemEl.innerHTML = `
                    <h3 class="font-bold text-gray-800 text-sm mb-1 flex items-center gap-1.5">
                        <span>${currentTab === 'museum' ? '🗞️' : '🌟'}</span>
                        <span>${i + 1}. ${place.place_name}</span>
                    </h3>
                    <p class="text-xs text-gray-600 mb-1">${place.road_address_name || place.address_name}</p>
                    <p class="text-xs text-amber-700 font-medium">${place.phone || '전화번호 없음'}</p>
                `;

                // 리스트 항목 클릭 시 해당 위치로 지도 이동 및 마커 인포윈도우 열기
                itemEl.addEventListener('click', () => {
                    map.setCenter(placePosition);
                    map.setLevel(3); // 상세 위치 확대

                    if (currentInfowindow) {
                        currentInfowindow.close();
                    }
                    infowindow.open(map, marker);
                    currentInfowindow = infowindow;
                });

                listContainer.appendChild(itemEl);
            }

            // 지도 범위 재설정
            map.setBounds(bounds);
        }

        /**
         * 9. 탭별로 색상 및 모양이 다른 마커를 생성하는 함수 (커스텀 오버레이 활용)
         */
        function addCustomMarker(position, tabType, index) {
            // 국립박물관은 🗞️ 이모지 마커, 문화유산은 🌟 이모지 마커 생성
            const emoji = tabType === 'museum' ? '🗞️' : '🌟';
            const bgColor = tabType === 'museum' ? '#b45309' : '#d97706';

            const content = document.createElement('div');
            content.className = 'flex items-center justify-center shadow-md rounded-full text-xs font-bold transition-transform hover:scale-110';
            content.style.cssText = `
                width: 32px;
                height: 32px;
                background-color: white;
                border: 2px solid ${bgColor};
                box-shadow: 0 4px 6px rgba(0,0,0,0.1);
                cursor: pointer;
            `;
            content.innerHTML = emoji;

            const customOverlay = new kakao.maps.CustomOverlay({
                position: position,
                content: content,
                map: map,
                zIndex: 2
            });

            markers.push(customOverlay); // 관리 배열에 추가

            // CustomOverlay에는 직접 click 이벤트가 안 붙으므로 감싸는 div에 addEventListener 추가
            // 카카오 지도 렌더링 이후 DOM 요소 접근을 위해 잠시 대기 후 이벤트 연결
            setTimeout(() => {
                content.addEventListener('click', () => {
                    // 마커 클릭 시 발생하는 커스텀 이벤트 트리거 역할
                    kakao.maps.event.trigger(dummyMarkerObject, 'click');
                });
            }, 100);

            // 이벤트 바인딩을 위한 가짜 마커 객체 생성 반환
            const dummyMarkerObject = new kakao.maps.Marker({ position: position });
            
            // 실제 지도 클릭 연동을 위해 더미 마커에 리스너 연결 연동
            return dummyMarkerObject;
        }

        /**
         * 10. 기존에 생성된 모든 마커(오버레이) 제거 함수
         */
        function removeMarkers() {
            for (let i = 0; i < markers.length; i++) {
                markers[i].setMap(null);
            }
            markers = [];
        }
    </script>
</body>
</html>
