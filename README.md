# birdd
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
    </style>
</head>
<body class="bg-gray-100 h-screen flex flex-col overflow-hidden">

    <!-- 상단 헤더 영역 -->
    <header class="bg-amber-700 text-white px-6 py-4 shadow-md flex items-center justify-between shrink-0">
        <h1 class="text-xl font-bold flex items-center gap-2">
            🏛️ 유적지 탐방 카카오맵 & 로드뷰
        </h1>
        <span class="text-xs bg-amber-800 text-amber-100 px-3 py-1 rounded-full font-medium">
            국립박물관 & 문화유산 탐색
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

            <!-- 키워드 직접 검색바 -->
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

        <!-- 오른쪽 영역: 지도 및 로드뷰 표시 영역 -->
        <section class="flex-1 relative bg-gray-200">
            <!-- 카카오 지도가 그려질 DIV 요소 -->
            <div id="map" class="w-full h-full"></div>
            
            <!-- 로드뷰가 그려질 DIV 요소 (기본 숨김 처리) -->
            <div id="roadview" class="w-full h-full absolute inset-0 hidden z-0"></div>
            
            <!-- 지도/로드뷰 전환 토글 버튼 -->
            <div class="absolute top-6 left-6 z-20 bg-white shadow-md rounded-lg p-1 flex gap-1 border border-gray-200">
                <button 
                    id="btn-map" 
                    onclick="toggleView('map')" 
                    class="px-3 py-1.5 rounded-md text-xs font-bold bg-amber-700 text-white transition-colors cursor-pointer"
                >
                    🗺️ 지도 보기
                </button>
                <button 
                    id="btn-roadview" 
                    onclick="toggleView('roadview')" 
                    class="px-3 py-1.5 rounded-md text-xs font-bold text-gray-700 hover:bg-gray-100 transition-colors cursor-pointer"
                >
                    🚗 로드뷰 보기
                </button>
            </div>

            <!-- 현재 위치로 이동하는 버튼 -->
            <button 
                onclick="moveToCurrentLocation()"
                class="absolute bottom-6 right-6 bg-white hover:bg-gray-50 text-gray-700 font-bold px-4 py-3 rounded-full shadow-lg border border-gray-200 text-sm flex items-center gap-2 z-20 transition-transform active:scale-95 cursor-pointer"
                title="내 위치로 이동"
            >
                🧭 내 위치
            </button>
        </section>

    </main>

    <!-- 카카오 지도 JavaScript SDK 로드 (libraries=services 포함) -->
    <script type="text/javascript" src="//dapi.kakao.com/v2/maps/sdk.js?appkey=452ca8db7c18b33eb094278b28cc9239&libraries=services&autoload=false"></script>
    
    <script>
        // 전역 변수 선언
        let map = null;               // 지도 객체
        let ps = null;                // 장소 검색 서비스 객체
        let markers = [];             // 지도에 표시된 마커 배열
        let currentPosition = null;   // 현재 위치 좌표
        let currentTab = 'museum';    // 현재 활성화된 탭 ('museum' 또는 'heritage')
        let currentInfowindow = null; // 현재 열려있는 팝업
        let currentKeyword = '';      // 키워드 검색어

        // 로드뷰 관련 전역 객체
        let roadviewContainer = null; 
        let roadview = null;          
        let roadviewClient = null;    
        let activeViewMode = 'map';   // 현재 보고 있는 화면 모드 ('map' 또는 'roadview')
        let selectedPlacePosition = null; // 사용자가 선택한 장소의 좌표

        // 페이지 로드 완료 시 SDK 초기화 실행
        window.addEventListener('DOMContentLoaded', () => {
            kakao.maps.load(() => {
                initMap();
                initRoadview();
            });
        });

        /**
         * 1. 지도 초기화 및 위치 권한 요청 함수
         */
        function initMap() {
            const mapContainer = document.getElementById('map');
            
            let defaultLat = 37.566826;
            let defaultLng = 126.978656;

            const mapOption = {
                center: new kakao.maps.LatLng(defaultLat, defaultLng),
                level: 5 // 지도 확대 레벨 5
            };

            map = new kakao.maps.Map(mapContainer, mapOption);
            ps = new kakao.maps.services.Places();

            if (navigator.geolocation) {
                navigator.geolocation.getCurrentPosition(
                    (position) => {
                        const lat = position.coords.latitude;
                        const lng = position.coords.longitude;
                        
                        currentPosition = new kakao.maps.LatLng(lat, lng);
                        selectedPlacePosition = currentPosition; // 초기 선택 좌표는 내 위치
                        map.setCenter(currentPosition);
                        
                        displayCurrentLocationMarker(currentPosition);
                        searchPlacesByTab();
                    },
                    (error) => {
                        console.warn('위치 권한을 얻지 못했습니다. 기본 위치로 설정합니다.', error);
                        currentPosition = new kakao.maps.LatLng(defaultLat, defaultLng);
                        selectedPlacePosition = currentPosition;
                        displayCurrentLocationMarker(currentPosition);
                        searchPlacesByTab();
                    },
                    { enableHighAccuracy: true, timeout: 5000, maximumAge: 0 }
                );
            } else {
                currentPosition = new kakao.maps.LatLng(defaultLat, defaultLng);
                selectedPlacePosition = currentPosition;
                displayCurrentLocationMarker(currentPosition);
                searchPlacesByTab();
            }
        }

        /**
         * 2. 로드뷰 객체 초기화 함수
         */
        function initRoadview() {
            roadviewContainer = document.getElementById('roadview'); // 로드뷰를 표시할 div
            roadview = new kakao.maps.Roadview(roadviewContainer);   // 로드뷰 객체
            roadviewClient = new kakao.maps.RoadviewClient();        // 좌표로부터 로드뷰 파노ID를 가져올 helper 객체
        }

        /**
         * 3. 특정 좌표의 로드뷰 실행 함수
         */
        function updateRoadview(position) {
            // 특정 위치의 좌표와 가까운 로드뷰의 panoId를 추출하여 로드뷰를 띄운다
            roadviewClient.getNearestPanoId(position, 50, function(panoId) {
                if (panoId) {
                    roadview.setPanoId(panoId, position); // panoId와 중심좌표를 통해 로드뷰 실행
                } else {
                    console.warn('이 위치 주변에는 제공되는 로드뷰가 없습니다.');
                }
            });
        }

        /**
         * 4. 지도 보기와 로드뷰 보기 화면 전환 함수
         */
        function toggleView(mode) {
            activeViewMode = mode;
            const mapDiv = document.getElementById('map');
            const rvDiv = document.getElementById('roadview');
            const btnMap = document.getElementById('btn-map');
            const btnRv = document.getElementById('btn-roadview');

            if (mode === 'map') {
                mapDiv.style.display = 'block';
                rvDiv.classList.add('hidden');
                btnMap.className = "px-3 py-1.5 rounded-md text-xs font-bold bg-amber-700 text-white transition-colors cursor-pointer";
                btnRv.className = "px-3 py-1.5 rounded-md text-xs font-bold text-gray-700 hover:bg-gray-100 transition-colors cursor-pointer";
            } else {
                mapDiv.style.display = 'none';
                rvDiv.classList.remove('hidden');
                btnRv.className = "px-3 py-1.5 rounded-md text-xs font-bold bg-amber-700 text-white transition-colors cursor-pointer";
                btnMap.className = "px-3 py-1.5 rounded-md text-xs font-bold text-gray-700 hover:bg-gray-100 transition-colors cursor-pointer";
                
                // 로드뷰 화면으로 전환될 때 선택된 장소 좌표로 로드뷰 갱신
                if (selectedPlacePosition) {
                    updateRoadview(selectedPlacePosition);
                }
            }
        }

        /**
         * 5. 현재 위치 파란 점 표시 함수
         */
        function displayCurrentLocationMarker(locPosition) {
            const content = document.createElement('div');
            content.style.cssText = `
                width: 20px;
                height: 20px;
                background-color: #3b82f6;
                border: 3px solid #ffffff;
                border-radius: 50%;
                box-shadow: 0 0 10px rgba(0, 0, 0, 0.3);
            `;

            new kakao.maps.CustomOverlay({
                position: locPosition,
                content: content,
                map: map,
                zIndex: 1
            });
        }

        /**
         * 6. '내 위치' 버튼 클릭 시 지도 이동 함수
         */
        function moveToCurrentLocation() {
            if (currentPosition) {
                map.setCenter(currentPosition);
                map.setLevel(5);
                selectedPlacePosition = currentPosition;
                if (activeViewMode === 'roadview') {
                    updateRoadview(currentPosition);
                }
            } else {
                alert('현재 위치 정보를 아직 불러오지 못했습니다.');
            }
        }

        /**
         * 7. 탭 전환 처리 함수
         */
        function switchTab(tabName) {
            currentTab = tabName;
            currentKeyword = '';
            document.getElementById('search-input').value = '';

            const btnMuseum = document.getElementById('tab-museum');
            const btnHeritage = document.getElementById('tab-heritage');

            if (tabName === 'museum') {
                btnMuseum.className = "py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-amber-700 text-white cursor-pointer";
                btnHeritage.className = "py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-white text-gray-700 border border-gray-300 hover:bg-gray-100 cursor-pointer";
            } else {
                btnHeritage.className = "py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-amber-700 text-white cursor-pointer";
                btnMuseum.className = "py-3 px-4 rounded-xl font-bold text-sm transition-all shadow-sm flex items-center justify-center gap-2 bg-white text-gray-700 border border-gray-300 hover:bg-gray-100 cursor-pointer";
            }

            searchPlacesByTab();
        }

        /**
         * 8. 상단 입력창 키워드 검색 처리 함수
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
         * 9. 현재 선택된 탭과 조건에 맞춰 장소 검색 요청 수행
         */
        function searchPlacesByTab() {
            if (currentInfowindow) {
                currentInfowindow.close();
            }

            removeMarkers();

            let searchQuery = '';
            if (currentKeyword) {
                searchQuery = currentTab === 'museum' ? currentKeyword + ' 국립박물관' : currentKeyword + ' 유적건조물';
            } else {
                searchQuery = currentTab === 'museum' ? '국립박물관' : '유적건조물';
            }

            const searchOptions = {
                location: map.getCenter(),
                radius: 20000
            };

            ps.keywordSearch(searchQuery, placesSearchCB, searchOptions);
        }

        /**
         * 10. 장소 검색 결과 콜백 함수
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
         * 11. 검색 결과 리스트 및 마커 화면 출력 함수
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
                    <div style="padding:10px; font-size:12px; width:200px; line-height:1.4;">
                        <strong style="font-size:13px; color:#b45309;">${place.place_name}</strong><br>
                        <span style="color:#666;">${place.road_address_name || place.address_name}</span>
                    </div>
                `;
                const infowindow = new kakao.maps.InfoWindow({
                    content: infowindowContent,
                    removable: true
                });

                // 마커 클릭 시: 지도 이동 및 로드뷰 좌표 갱신
                kakao.maps.event.addListener(marker, 'click', () => {
                    if (currentInfowindow) {
                        currentInfowindow.close();
                    }
                    infowindow.open(map, marker);
                    currentInfowindow = infowindow;

                    selectedPlacePosition = placePosition;
                    if (activeViewMode === 'roadview') {
                        updateRoadview(placePosition);
                    }
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

                // 리스트 클릭 시: 지도 중심 이동, 로드뷰 좌표 갱신, 팝업 열기
                itemEl.addEventListener('click', () => {
                    map.setCenter(placePosition);
                    map.setLevel(3);
                    selectedPlacePosition = placePosition;

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
         * 12. 탭별 커스텀 마커 생성 함수
         */
        function addCustomMarker(position, tabType) {
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
         * 13. 마커 전체 삭제 함수
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
