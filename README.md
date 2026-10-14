<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>유적지 탐방 & 로드뷰 서비스</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
    <style>
        body {
            font-family: 'Malgun Gothic', sans-serif;
        }
        /* 스크롤바 디자인 */
        ::-webkit-scrollbar {
            width: 6px;
        }
        ::-webkit-scrollbar-thumb {
            background-color: #cbd5e1;
            border-radius: 3px;
        }
    </style>
</head>
<body class="bg-gray-100 h-screen flex flex-col overflow-hidden">

    <!-- 상단 헤더 영역 -->
    <header class="bg-amber-800 text-white px-6 py-3 shadow-md flex items-center justify-between shrink-0">
        <h1 class="text-lg font-bold flex items-center gap-2">
            🏛️ 유적지 탐방 & 로드뷰 서비스
        </h1>
        <span class="text-xs bg-amber-900 text-amber-100 px-3 py-1 rounded-full font-medium">
            국립박물관 & 문화유산 내 손 안의 가이드
        </span>
    </header>

    <!-- 메인 컨테이너: 좌우 2분할 레이아웃 (반응형) -->
    <main class="flex-1 flex flex-col md:flex-row overflow-hidden">
        
        <!-- 왼쪽 영역: 탭, 검색바, 장소 리스트 -->
        <section class="w-full md:w-[420px] lg:w-[460px] bg-white border-r border-gray-200 flex flex-col shrink-0 z-10 shadow-lg">
            
            <!-- 탭 버튼 영역 (국립박물관 / 문화유산) -->
            <div class="grid grid-cols-2 gap-2 p-4 bg-gray-50 border-b border-gray-200">
                <button 
                    id="tab-museum" 
                    onclick="switchTab('museum')"
                    class="py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-amber-800 text-white cursor-pointer"
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

            <!-- 키워드 직접 검색바 -->
            <div class="p-4 border-b border-gray-100 bg-white">
                <form id="search-form" onsubmit="searchByKeyword(event)" class="flex gap-2">
                    <input 
                        type="text" 
                        id="search-input" 
                        placeholder="지역명 입력 (예: 경주, 서울, 부여)" 
                        class="flex-1 px-4 py-2.5 border border-gray-300 rounded-xl focus:outline-none focus:ring-2 focus:ring-amber-800 text-sm"
                    >
                    <button 
                        type="submit" 
                        class="bg-amber-800 hover:bg-amber-900 text-white font-semibold px-5 py-2.5 rounded-xl text-sm transition-colors cursor-pointer"
                    >
                        검색
                    </button>
                </form>
            </div>

            <!-- 장소 결과 리스트 영역 -->
            <div id="place-list-container" class="flex-1 overflow-y-auto divide-y divide-gray-100 p-3 space-y-2">
                <div class="text-center text-gray-400 py-20 text-sm">
                    위치를 허용하거나 검색어를 입력하면 유적지를 불러옵니다!
                </div>
            </div>
        </section>

        <!-- 오른쪽 영역: 지도 및 로드뷰 화면 -->
        <section class="flex-1 relative bg-gray-200 flex flex-col">
            
            <!-- 뷰 전환 토글 버튼 (지도 / 로드뷰) - 확실하게 보이도록 상단에 배치 -->
            <div class="absolute top-4 left-4 z-20 bg-white shadow-lg rounded-xl p-1.5 flex gap-1 border border-gray-200">
                <button 
                    id="btn-map" 
                    onclick="changeViewMode('map')" 
                    class="px-4 py-2 rounded-lg text-xs font-bold bg-amber-800 text-white transition-colors cursor-pointer flex items-center gap-1.5 shadow-sm"
                >
                    🗺️ 지도 보기
                </button>
                <button 
                    id="btn-roadview" 
                    onclick="changeViewMode('roadview')" 
                    class="px-4 py-2 rounded-lg text-xs font-bold text-gray-700 hover:bg-gray-100 transition-colors cursor-pointer flex items-center gap-1.5"
                >
                    🚗 로드뷰 보기
                </button>
            </div>

            <!-- 카카오 지도 컨테이너 -->
            <div id="map" class="w-full h-full"></div>

            <!-- 카카오 로드뷰 컨테이너 (기본 숨김) -->
            <div id="roadview" class="w-full h-full hidden"></div>
            
            <!-- 현재 위치로 이동하는 버튼 -->
            <button 
                onclick="moveToCurrentLocation()"
                class="absolute bottom-6 right-6 bg-white hover:bg-gray-50 text-gray-700 font-bold px-4.5 py-3 rounded-full shadow-xl border border-gray-200 text-sm flex items-center gap-2 z-20 transition-transform active:scale-95 cursor-pointer"
                title="내 위치로 이동"
            >
                🧭 내 위치
            </button>
        </section>

    </main>

    <!-- 카카오 지도 SDK (libraries=services 포함) -->
    <script type="text/javascript" src="//dapi.kakao.com/v2/maps/sdk.js?appkey=452ca8db7c18b33eb094278b28cc9239&libraries=services&autoload=false"></script>
    
    <script>
        // 전역 변수 선언
        let map = null;               // 지도 객체
        let ps = null;                // 장소 검색 객체
        let markers = [];             // 마커 객체 배열
        let currentPosition = null;   // 사용자 현재 위치 좌표
        let currentTab = 'museum';    // 현재 탭 ('museum' 또는 'heritage')
        let currentInfowindow = null; // 열려있는 인포윈도우 팝업
        let currentKeyword = '';      // 검색어 키워드

        // 로드뷰 관련 변수
        let roadview = null;          
        let roadviewClient = null;    
        let activeViewMode = 'map';   // 현재 화면 모드 ('map' 또는 'roadview')
        let selectedPosition = null;  // 현재 선택된 장소의 좌표

        // 페이지 로드 시 SDK 초기화
        window.addEventListener('DOMContentLoaded', () => {
            kakao.maps.load(() => {
                initMapAndRoadview();
            });
        });

        /**
         * 1. 지도 및 로드뷰 초기화 함수
         */
        function initMapAndRoadview() {
            const mapContainer = document.getElementById('map');
            
            // 기본 중심 좌표 (서울 시청)
            let defaultLat = 37.566826;
            let defaultLng = 126.978656;

            const mapOption = {
                center: new kakao.maps.LatLng(defaultLat, defaultLng),
                level: 5 // 요구 조건: 지도 확대 레벨 5
            };

            // 지도 생성
            map = new kakao.maps.Map(mapContainer, mapOption);
            ps = new kakao.maps.services.Places();

            // 로드뷰 객체 생성
            const roadviewContainer = document.getElementById('roadview');
            roadview = new kakao.maps.Roadview(roadviewContainer);
            roadviewClient = new kakao.maps.RoadviewClient();

            // 위치 권한 요청 (Geolocation)
            if (navigator.geolocation) {
                navigator.geolocation.getCurrentPosition(
                    (position) => {
                        const lat = position.coords.latitude;
                        const lng = position.coords.longitude;
                        
                        currentPosition = new kakao.maps.LatLng(lat, lng);
                        selectedPosition = currentPosition;
                        map.setCenter(currentPosition);
                        
                        displayCurrentLocationMarker(currentPosition);
                        searchPlacesByTab(); // 초기 검색 실행
                    },
                    (error) => {
                        console.warn('위치 권한을 얻지 못했습니다. 기본 위치로 설정합니다.', error);
                        currentPosition = new kakao.maps.LatLng(defaultLat, defaultLng);
                        selectedPosition = currentPosition;
                        displayCurrentLocationMarker(currentPosition);
                        searchPlacesByTab();
                    },
                    { enableHighAccuracy: true, timeout: 5000, maximumAge: 0 }
                );
            } else {
                currentPosition = new kakao.maps.LatLng(defaultLat, defaultLng);
                selectedPosition = currentPosition;
                displayCurrentLocationMarker(currentPosition);
                searchPlacesByTab();
            }
        }

        /**
         * 2. 현재 위치 파란 점 표시 함수
         */
        function displayCurrentLocationMarker(locPosition) {
            const content = document.createElement('div');
            content.style.cssText = `
                width: 22px;
                height: 22px;
                background-color: #2563eb;
                border: 3px solid #ffffff;
                border-radius: 50%;
                box-shadow: 0 0 10px rgba(0, 0, 0, 0.4);
            `;

            new kakao.maps.CustomOverlay({
                position: locPosition,
                content: content,
                map: map,
                zIndex: 1
            });
        }

        /**
         * 3. '내 위치' 버튼 클릭 시 이동
         */
        function moveToCurrentLocation() {
            if (currentPosition) {
                map.setCenter(currentPosition);
                map.setLevel(5);
                selectedPosition = currentPosition;
                if (activeViewMode === 'roadview') {
                    updateRoadview(currentPosition);
                }
            } else {
                alert('현재 위치 정보를 아직 불러오지 못했습니다.');
            }
        }

        /**
         * 4. 탭 전환 처리 함수
         */
        function switchTab(tabName) {
            currentTab = tabName;
            currentKeyword = '';
            document.getElementById('search-input').value = '';

            const btnMuseum = document.getElementById('tab-museum');
            const btnHeritage = document.getElementById('tab-heritage');

            if (tabName === 'museum') {
                btnMuseum.className = "py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-amber-800 text-white cursor-pointer";
                btnHeritage.className = "py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-white text-gray-700 border border-gray-300 hover:bg-gray-100 cursor-pointer";
            } else {
                btnHeritage.className = "py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-amber-800 text-white cursor-pointer";
                btnMuseum.className = "py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-white text-gray-700 border border-gray-300 hover:bg-gray-100 cursor-pointer";
            }

            searchPlacesByTab();
        }

        /**
         * 5. 검색창 키워드 검색 처리 함수 (전국 검색 가능하도록 수정)
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
         * 6. 탭 및 검색어 조건에 따른 장소 검색 요청
         */
        function searchPlacesByTab() {
            if (currentInfowindow) {
                currentInfowindow.close();
            }
            removeMarkers();

            let searchQuery = '';
            if (currentKeyword) {
                // 사용자가 키워드(예: 경주)를 입력하면 전국 어디든 검색되도록 조합
                searchQuery = currentTab === 'museum' ? currentKeyword + ' 국립박물관' : currentKeyword + ' 유적건조물';
            } else {
                // 키워드가 없으면 현재 지도 중심 주변 검색
                searchQuery = currentTab === 'museum' ? '국립박물관' : '유적건조물';
            }

            // 키워드가 입력된 경우 지역 제한 없이 검색하기 위해 searchOptions 생략 또는 전국 단위 검색 적용
            if (currentKeyword) {
                ps.keywordSearch(searchQuery, placesSearchCB);
            } else {
                const searchOptions = {
                    location: map.getCenter(),
                    radius: 20000
                };
                ps.keywordSearch(searchQuery, placesSearchCB, searchOptions);
            }
        }

        /**
         * 7. 검색 결과 콜백 함수
         */
        function placesSearchCB(data, status, pagination) {
            const listContainer = document.getElementById('place-list-container');

            if (status === kakao.maps.services.Status.OK) {
                displayPlaces(data);
            } else if (status === kakao.maps.services.Status.ZERO_RESULT) {
                listContainer.innerHTML = `
                    <div class="text-center text-gray-400 py-20 text-sm">
                        검색 결과가 없습니다. 다른 지역이나 검색어를 입력해보세요!
                    </div>
                `;
            } else if (status === kakao.maps.services.Status.ERROR) {
                alert('검색 중 오류가 발생했습니다.');
            }
        }

        /**
         * 8. 검색 결과 리스트 및 마커 출력
         */
        function displayPlaces(places) {
            const listContainer = document.getElementById('place-list-container');
            listContainer.innerHTML = '';

            const bounds = new kakao.maps.LatLngBounds();

            for (let i = 0; i < places.length; i++) {
                const place = places[i];
                const placePosition = new kakao.maps.LatLng(place.y, place.x);

                const marker = addCustomMarker(placePosition, currentTab);
                bounds.extend(placePosition);

                const infowindowContent = `
                    <div style="padding:12px; font-size:12px; width:220px; line-height:1.5;">
                        <strong style="font-size:14px; color:#92400e;">${place.place_name}</strong><br>
                        <span style="color:#555;">${place.road_address_name || place.address_name}</span><br>
                        <span style="color:#2563eb; font-weight:600;">${place.phone || '번호 없음'}</span>
                    </div>
                `;
                const infowindow = new kakao.maps.InfoWindow({
                    content: infowindowContent,
                    removable: true
                });

                // 마커 클릭 시 이벤트
                kakao.maps.event.addListener(marker, 'click', () => {
                    if (currentInfowindow) {
                        currentInfowindow.close();
                    }
                    infowindow.open(map, marker);
                    currentInfowindow = infowindow;

                    selectedPosition = placePosition;
                    if (activeViewMode === 'roadview') {
                        updateRoadview(placePosition);
                    }
                });

                // 왼쪽 카드 리스트 아이템 생성
                const itemEl = document.createElement('div');
                itemEl.className = 'bg-white p-4 rounded-xl border border-gray-200 hover:border-amber-600 hover:shadow-md cursor-pointer transition-all';
                itemEl.innerHTML = `
                    <div class="flex items-start justify-between mb-1">
                        <h3 class="font-bold text-gray-900 text-sm flex items-center gap-1.5">
                            <span class="text-base">${currentTab === 'museum' ? '🗞️' : '🌟'}</span>
                            <span>${i + 1}. ${place.place_name}</span>
                        </h3>
                    </div>
                    <p class="text-xs text-gray-600 mb-1.5 pl-6">${place.road_address_name || place.address_name}</p>
                    <p class="text-xs text-amber-800 font-medium pl-6">${place.phone || '전화번호 정보 없음'}</p>
                `;

                // 리스트 카드 클릭 시
                itemEl.addEventListener('click', () => {
                    map.setCenter(placePosition);
                    map.setLevel(3);
                    selectedPosition = placePosition;

                    if (activeViewMode === 'roadview') {
                        updateRoadview(placePosition);
                    }

                    if (currentInfowindow) {
                        currentInfowindow.close();
                    }
                    infowindow.open(map, marker);
                    currentInfowindow = infowindow;
                });

                listContainer.appendChild(itemEl);
            }

            map.setBounds(bounds);
        }

        /**
         * 9. 탭별 커스텀 마커 생성
         */
        function addCustomMarker(position, tabType) {
            const emoji = tabType === 'museum' ? '🗞️' : '🌟';
            const borderColor = tabType === 'museum' ? '#92400e' : '#d97706';

            const content = document.createElement('div');
            content.className = 'flex items-center justify-center shadow-lg rounded-full text-xs font-bold transition-transform hover:scale-125';
            content.style.cssText = `
                width: 36px;
                height: 36px;
                background-color: white;
                border: 3px solid ${borderColor};
                cursor: pointer;
            `;
            content.innerHTML = emoji;

            new kakao.maps.CustomOverlay({
                position: position,
                content: content,
                map: map,
                zIndex: 2
            });

            const dummyMarker = new kakao.maps.Marker({ position: position });
            
            setTimeout(() => {
                content.addEventListener('click', () => {
                    kakao.maps.event.trigger(dummyMarker, 'click');
                });
            }, 100);

            markers.push(dummyMarker);
            return dummyMarker;
        }

        /**
         * 10. 기존 마커 전체 삭제
         */
        function removeMarkers() {
            for (let i = 0; i < markers.length; i++) {
                markers[i].setMap(null);
            }
            markers = [];
        }

        /**
         * 11. 지도 보기 / 로드뷰 보기 모드 전환 함수
         */
        function changeViewMode(mode) {
            activeViewMode = mode;
            const mapDiv = document.getElementById('map');
            const roadviewDiv = document.getElementById('roadview');
            const btnMap = document.getElementById('btn-map');
            const btnRoadview = document.getElementById('btn-roadview');

            if (mode === 'map') {
                mapDiv.style.display = 'block';
                roadviewDiv.style.display = 'none';
                btnMap.className = "px-4 py-2 rounded-lg text-xs font-bold bg-amber-800 text-white transition-colors cursor-pointer flex items-center gap-1.5 shadow-sm";
                btnRoadview.className = "px-4 py-2 rounded-lg text-xs font-bold text-gray-700 hover:bg-gray-100 transition-colors cursor-pointer flex items-center gap-1.5";
                map.relayout();
            } else {
                mapDiv.style.display = 'none';
                roadviewDiv.style.display = 'block';
                btnRoadview.className = "px-4 py-2 rounded-lg text-xs font-bold bg-amber-800 text-white transition-colors cursor-pointer flex items-center gap-1.5 shadow-sm";
                btnMap.className = "px-4 py-2 rounded-lg text-xs font-bold text-gray-700 hover:bg-gray-100 transition-colors cursor-pointer flex items-center gap-1.5";
                
                roadview.relayout();
                if (selectedPosition) {
                    updateRoadview(selectedPosition);
                }
            }
        }

        /**
         * 12. 특정 좌표의 로드뷰 반영 함수
         */
        function updateRoadview(position) {
            roadviewClient.getNearestPanoId(position, 50, function(panoId) {
                if (panoId) {
                    roadview.setPanoId(panoId, position);
                } else {
                    alert('이 주변에는 제공되는 로드뷰 사진이 없습니다.');
                }
            });
        }
    </script>
</body>
</html>
