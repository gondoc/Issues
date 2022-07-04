# gondo's log

220704 데이터베이스(DB DataBase) 이중화
	정의 : 
		데이터베이스(Master 이후 'M')의 변경된 데이터를 
		물리적으로 떨어진 각각의 데이터베이스(Slave 이후 'S')에 동일하게 유지하여 관리하는 것
	이유 : 
		데이터의 유실 방지, 무중단 서비스를 위해 이중화가 필수임.
	과정 : 
		1. M 으로 사용하려는 DB가 설치된 서버의 WAL(Write Ahead Log)에 
		   M 에서 발생하는 모든 작업을 로그로 기록한다.
			-> WAL 로그가 쌓이게 됨.
			ex) MySQL / binary log == Postgresql M / WAL
		2. 해당 로그를 S로 사용하려는 DB가 설치된 서버로 전달한다.
		3. S 에서 받은 로그를 재실행함으로써 M와 S간 데이터를 동일하게 유지할 수 있음. 
	방식 : 
		Log-shipping / Streaming 두 가지 방식이 존재함.
			-> Streaming 방식(9.x ↑에서 사용가능)
		
		Log-shipping 방식의 특장점.
			M 의 WAL 파일 자체를 전송하는 방식. 
				Async 방식만 가능함.
					Async 방식은 S 에 WAL 파일이 제대로 복제되었다는 응답을 받지 않음\
					이에 따라, 속도는 빠르나, 데이터 유실 가능성이 존재함. 
				
			M 의 WAL 파일이 크기를 다 채워야만 S 로 전송하기 때문에 
			전송에 필요한 WAL 파일의 크기를 채우는 동안은 
			M 과 S 간 데이터 갭이 발생한다. (동일하지 않은 데이터 상태임)
				이때, M 에서 장애가 발생한다면 데이터의 유실이 일어남. 
			
		Streaming 방식의 특장점
			M 의 WAL 내용을 전송하는 방식.
				Async 방식과 Sync 방식 모두 가능함. 
					Sync 방식은 S 에 전달한 내용이 제대로 복제되었다는 응답을 받음. 
					이에 따라, 속도는 느리나, 데이터 유실 가능성이 낮음. 
					
			하지만 데이터 유실 가능성이 낮을 뿐,
			아래의 상황에서 데이터 유실 가능성이 있음.
				
				Sync 방식의 데이터 유실 가능성
					(예시)
					1. 저장할 수 있는 WAL 파일의 최대 개수를 지정하는 postgresql.conf의
					wal_keep_segments 옵션의 값을 10으로 지정한 상태에서 
					
					2. S 에서 긴 시간의 서버 장애가 발생한 경우
					
					3. M 에서는 장애가 발생하지 않아 WAL 을 지속적으로 쌓아두기만 함. 
						WAL 1(AAA), WAL 2(BBB), WAL 3(CCC), ...

					4. S 의 장애가 해결되지 않은 상태로 
						M 의 WAL 최대 파일 개수인 10 을 넘어가버리면 
						WAL 1(KKK) 가 새로 덮어쓰게 됨. 
							-> 데이터 유실 발생!
						S 의 장애가 지속적으로 해결되지 않는다면,
						WAL 1, WAL 2, WAL 3, ... 계속 돌아가면서 덮어쓰여지게 됨. 
						
					5. S 의 장애가 해결된 후 M 으로부터 S 에게 WAL 전송이 이루어지더라도
						덮어쓰여진 WAL 1(KKK), ... 가 쓰여지게 됨. 
						덮어쓰여져 유실된 기존 WAL 1(AAA), WAL 2(BBB), ... 등은 복구 어려움 
					
			또 하나의 환경적 특징
				M 에서는 WAL Sender Process가 동작하고,
				S 에서는 WAL Receiver Process가 동작함. 
				
			
	세팅 : 
		외부접근 허용 (M and S)
			postgresql.conf
				listen_addresses='*'

---

220701 etc
	
	지난 JINJU-PUBLISH GIS evtPage 구현 및 리팩토링 완료
	현재 추후 프로젝트 투입을 위한 React 예제 학습 중 
	
---
	
220630 GIS projectReview's log

	사용스택 및 기능
	1. jq ajax / json
		1-1. getEventList
			db 접근 및 evtList(30)건 데이터 목록 표출
			json으로 받아 for문 돌면서 json.length 만큼 json[i] 마다 
			뷰단의 특정 태그에 append 시키는 식으로 진행함.
			값은 data-attribute 이용하여 태그 attr 로 접근할 수 있도록 함. 
				뷰단의 태그에 필요한 value 들이 있어 추후 동일한 api를 호출하지 않을 수 있음. 
				추후 event가 생성될 수 있으니, for문 처음 시작할때엔 초기화 처리를 해줘야 한다.
				
		1-2. getEventDetail 
			뷰단에서 출력된 태그 셀렉터, data-attribute 이용하여 data 접근.
			표출된 목록 중 임의의 1건 onClick 시 상세정보 표출
			1-1과 동일하게 for문 돌면서 특정 태그 append 하는 식으로 처리함. 
				for문 첫 시작시 첫 줄로 특정 태그에 append 했던 태그들을 .empty() 해줘야 한다.
				
		1-3. getCctvList
			window가 onload되면 동작하도록 코드를 작성하였고,
			featureCollection들의 data를 readFeatures 하여 vectorSource에 addFeatures 함.
				promise 함수 .then(() => { 하여 $.("#loading").hide(); 하였음.
					대개의 로딩화면은 해당 방식으로 구성됨. 
					
				+. 각각의 자바스크립트들의 로드 순서는 아무리 개발자가 순서를 잡아주어도 로드 순서를 보장할 수 없다. 
					예를 들어, for문 중 첫번째의 동적태그가 생성된 직후 다른 자바스크립트가 동작하여
					추후 생성될 동적태그가 생성되지 않은 상태에서 자바스크립트가 동작한다던지...
				-> 해결방안
					자바스크립트들이 불규칙적으로 로드되어 사용자가 예상하지 못한 환경에서 
					onClick onKeyUp 등 이벤트를 발생시키지 않게끔하기 위해 
					화면 모두 덮어버리는 style.css를 가진 <div id="loading"> 를 생성한다.
					해당 div 태그를 onload 후 처리되는 .then() 부분에 hide() 처리하면 된다. 
					사용자가 할 수 있는 이벤트를 원천적으로 봉쇄하는 것.
					
	2. openLayers
		2-1. map.view
			ol.source.XYZ openLayers {x}{y}{z}.png 이용
			ol.source.Cluster

		2-2. map.feature
			특정 location btn onClick event발생시 btn의 좌표 접근 및 animate 처리하여 화면이동 표출
			화면 이동 후 해당 좌표에 feature를 신규 생성함.  
			신규 feature 는 feature -> source -> layer 순으로 작업하여 map 에 포함시킴. 

		2-2. cctv 투망
			특정 상황 이벤트 발생 후 해당 이벤트의 cctv btn onClick event발생시 
			btn의 좌표 접근 및 투망 모니터링 표출
			이때, 화면에 표출되는 cctv 들은 map 화면을 구성할 때 존재하는 featureCollection들의 정보를 변환하여 
			화면에 표출한다.
			
			jq ajax로 호출하여 map에 표출된 feature들의 정보를 받아 array화 한다.
			해당 array의 crdnt, selected event crdnt 간 distance를 wgs84Sphere.haversineDistance(c1, c2)하여 
			신규 array에 push 한다. 이때 clustered 된 subFeature까지 찾아서 array에 push하는 것이 중요함.
		
			distance 적은 순(이벤트 발생 위치와 cctv들 중 가장 가까운 cctv)으로 화면에 표출 하는 것이 
			목적이기 때문에 distance를 기준으로 sort desc 해주어야 한다. 
				arr.sort((a, b) => a.distance - b.distance);

---

220629 codeReview 

	feature clustered 상태에서 map에 올려진 모든 feature들을 접근하고 싶을때?
	-> cluster : zoom out 을 하다보면 특정 근거리 n개의 feature 들이 1개의 feature로 합쳐 표출되는 기능. 
	
		이때, 실제 feature을 접근해보면 feature안에 features(array)가 존재한다.
			-> 겹쳐져 보이지 않는 subFeatures까지 모두 합산하여 처리해야 한다.
			-> 테스트 프로젝트를 수행할 때엔, for문 돌리고 arr.push(subFeature)하여 처리했다.
			-> 드물게 cluster가 안되어 있는 1개의 Cam이 있는 경우가 있다.(보통 외지임.)
				-> 만약 해당 cam 이 있다면 if문 처리도 해야한다.
							
				const subFeatures = feature.get("features");
				for(let subFeature of subFeatures)  {
					arr.push({
						feature : subFeature
					});
				}
			
	지도에서 point A와 point B 간 distance를 구해야 한다면?
	-> 이때, 보이는 지도에서 구하는 것이 맞는지, 지구에서 실제 이동하는 것인지 판단해야 한다.
		-> 평면이 아닌 지구는 구 형태이다.
		
		const wgs84Sphere = new ol.Sphere(6378137);		
		const result = wgs84Sphere.haversineDistance( c1, c2 );
			// 고정적으로 radius 값을 6378137 meter 로 준다.
			-> 지구의 반지름은 일반적으로 6378137 미터(적도반경)
			// c1, c2 == 비교할 두 좌표배열
				-> [selectedX, selectedY], [eventX, eventY]
		
	배열을 정렬해야 한다면?
		오름차순 
		arr.sort((a, b) => a - b);
		
		내림차순
		arr.sort((a, b) => b - a);

		-> 만약 배열의 요소를 기준으로 정렬해야 한다면?
			오름차순 
			arr.sort((a, b) => a.key - b.key);
			
			내림차순
			arr.sort((a, b) => b.key - a.key);
		
	for문의 오해?
		기본 for문과 forEach문 둘다 특장점이 있으니, 
		하나를 주로 사용하려고 하지말고 적재적소에 사용할 것.
	
		흔히 처음 배우는 형태의 for문
			ex 
				for ( i=0; i<count; i++) {
		
		그 다음으로 배우는 for문
			ex
				for (BoardVO pojo : boardList) {

		+ for in / for of
			ex
				for(let x in y) {
				for(let x of y) {
			
	for문을 돌릴때, 동적태그로 생성되거나, 뷰단의 나열되는 number에 접근하고 싶을때,
		selector 에도 ${i} 형태로 백틱을 넣을 수 있음. 
			ex 
				$(`#example_title_${i}`).text(example.get('someThing'));
				
	openlayers featureCollection coordinate 접근할때의 ol 라이브러리
		ol.ol.js / ol.debug.js 
	
		배포 : ol.js 
			-> feature.getGeometry().getFirstCoordinate();
		개발 : debug.js
			-> feature.getGeometry().flatCoordinate();
		
	외부로 설정한 각각의 js 파일들이 로드되는 시점이 겹쳐 원하는 시점, 원하는 변수가 생성되지 않을 때 
		js 가 동작해버린다면 promise 함수를 이용하자.
			+ 다른 방법도 있음. 하단 참고.
	
		$.ajax ({
			url : '/some/thing/list',
			method : 'get',
			dataType : 'json',
			success : function(json) {
				...
			}
		}).then(()=> {
			// promise function 
			// 여기서 원하는 동작을 처리하면 된다. 
		});
	
			+. 화면에서 새로운 <div class="loading" id="loading"> 만들어 loadingStyle.css를 준다.
			화면을 아예 덮어버리는 css를 가진 div 태그를 화면에 보이게 하고 $("#loading").show();
			
			loadingStyle.css
				position: absolute;
				top: 0px;
				left: 0px;
				width: 100%;
				height: 100%;
				z-index: 3000;
				background-color: rgba(200,200,200,0.5);
			
			ajax 통신 후 promise 함수 처리로 
			아래처럼 해당 화면을 덮어버리는 태그를 숨기는 것도 방법임.
			
			.then(() => {
				$("#loading").hide();
			});
			
			로드되지 않은 상태의 화면 자체를 차단하는 것.
				-> 보통 일반적인 로딩화면을 이렇게 구성한다고 한다.
				-> 상기 내역에서는 적절한 css만 주었지만, 
					이미지와 간단한 애니메이션 gif 만 들어가면 
					우리가 통상적으로 알고 있는 로딩 화면이 구성된다.

---

220624 List Set Map 비교

	[List]
		특징
			순서 유지 O 
			데이터의 중복 허용

		구현 클래스
			ArrayList	
					단방향 포인터 구조
						자료에 대한 순차적인 접근에 강점
					상당히 빠르고 가변 크기인 배열형태
			LinkedList	
					양방향 포인터 구조
						데이터의 삽입, 삭제가 빈번한 경우 빠른 성능 보장.
			Vector		
					ArrayList의 구형 버전임. 잘 쓰이진 않음.
		추가
			List는 객체 자체를 저장하는 것이 아니라,
			객체의 번지(index)를 참조하는 것.
			.get(int index) 메서드를 제공한다.
			null도 저장가능함.
		
		결론
			List : 저장공간이 필요에 의해 자동으로 늘어난다 (순서가 있는 저장공간)
			특징 : 순서가 있고, 중복을 허용(배열과 유사)
			장점 : 가변적인 배열
			단점 : 원하는 데이터가 뒤쪽에 위치하는 경우 속도의 문제
			방식 : equals()를 이용한 데이터 검색

	[Set]
		특징
			순서 유지 X 
			데이터의 중복 불허용
		구현 클래스
			HashSet		
					순서를 보장하지 않는 Set
			TreeSet
					Binary Search Tree 구조
					추가 삭제에는 시간이 좀 더 소요되나,
					정렬 및 탐색에 성능이 좋음.
					오름차순으로 데이터를 저장함.
			LinkedHashSet
					데이터가 들어간 순서대로 저장하는 Set
		추가
			순서의 개념자체가 없어서 .get(int index) 메서드가 없음.
			단, 반복자가 있음 .iterator()
			반복자는 전체 객체를 대상으로 한번씩 반복해서 가져온다.
			null 저장가능하나, 데이터의 중복을 불허용하기 때문에 
			하나의 null 값만 저장가능하다.
		결론
			Set : 집합, 순서가 없다. 집합이므로 중복된 데이터가 들어갈 수 없다.
			중복되지 않는 숫자(데이터)를 구할 때 사용하면 유용하다.
			특징 : 순서가 없고, 중복을 허용하지 않는다
			장점 : 빠른 속도
			단점 : 단순 집합의 개념으로 정렬하려면 별도의 처리가 필요하다

	[Map]
		특징
			순서 유지 X 
			key와 value의 쌍으로 이루어진 데이터의 집합.
			키는 중복을 허용하지 않으며, 값의 중복을 허용한다.
		구현클래스
			HashMap
					순서를 보장하지 않는 Map. 
					Key와 Value형태로 null 허용한다.
			HashTable
					동기화를 지원하는 Map.
					Key와 Value로 null 불허용한다.
			TreeMap
					이진 검색 Tree 구조의 Map, 
					저장시 Key 기준으로 오름차순 저장됨
			Properties
					들어간 순서대로 저장되는 Map
		추가
			중복된 Key가 들어오면 새로운 Value로 대치됨. (기존 값 삭제)
		결론		
			Map : 키와 데이터를 같이 저장
			특징 : Key(키)랑 Value(값)으로 나눠서 데이터 관리
			장점 : 빠른 속도
			단점 : Key의 검색 속도가 검색 속도를 좌우


	List와 Set의 차이
		List는 순서대로 데이터가 들어가며 중복을 허용한다.
		Set은 순서가 보장되지 않고 중복을 허용하지 않는다.
		Map은 순서가 보장되지 않고 key의 중복을 허용하지 않고, Value의 중복은 허용한다. 

---

220620 11:33

	jquery 통일 작업해볼 것.
	모든 .js 파일 jquery화 할 것.
	
	jq 통일 작업 이후 feature controll 작업 진행할 것.

	deadlock 부분 공부 및 해결 방안 4가지에 대해 숙지할 것.
	js 파일이 교착상태에 빠진다면 dom이 생성될 때 기능이 같이 생성되도록 만들던가,
	무조건 해당 기능을 바라보게 만들던가 해야함.
	
	vanila script를 지향하지만, jq또한 많은 숙지가 필요하다.
	해당 부분 교착상태에 있던 것들을 파악하고 인지하고 있을 것.

---

220610 코드리뷰 및 qna 메모
	
	ol.source.cluster
		zoom lv 에 맞춰 map에 띄워진 feature 들이 중첩되어서 출력되는 경우
		해당 feature 들을 겹치게끔 할 수 있는데, 
		상기 기능이 cluster이다.
		ex. 네이버 지도 내 줌 아웃 하다보면 pin marker 들 겹치게 하는 기능
		
	.js .css ... lib file 주입 습관
		si 사업 특성상 고객사 혹은 제품 설치 환경에는 
		폐쇄망(인터넷 사용불가 환경)일 경우가 높다.
		
		해당 환경에는 통상적으로 정적리소스 파일을 넣어주던 방식인
		cdn 처리 방식, link href 방식 모두 정상적으로 처리가 어렵다.

		이에 따라, 활용하고자 하는 정적 리소스 파일의 공식 홈페이지 내 
		정적리소스를 직접 내려받아 path 처리해야 한다. 
		
		습관적으로 링크를 넣으면 정상적으로 구동이 절대 불가하다.
		
	뷰단에서도 로직을 분리해야 한다.
		뷰단에서는 화면(레이아웃, 골격 등)을 구성하는 로직이 있을 것이고,
		화면을 구성한 로직에서도 로직 내에서 화면 내 데이터를 만드는 로직이 있을 것이다.
		화면에 뿌려주는 데이터를 만드는 로직도 화면 구성 로직과 구분하는 것이 바람직하다.
		
		결론
			화면 구성 로직과 
			화면 데이터 로직을 구분하고 분리하자.
		
		하는 이유? 
			유지보수성
			협업의 용이함 
		
	F12 chrome devTools
		netword tab
			disabled cache 체크박스를 항상 체크해놓자
			캐시로 저장된 데이터를 지속적으로 새로 받을 수 있게 할 수 있음. 
			
			요청에 따라 캐시를 지속적으로 확인해야하는 개발자들에게 꿀같은 기능임.
			
		Fetch btn 
			비동기 통신, ajax, fetch 함수를 쓸 때, RestApi Test 를 할 때 
			response 값을 확인 할 수 있음.
		
		ctrl + R 
			devTools 화면에서 새로고침(ctrl + R)을 해도 view 화면 새로 고침 가능
			
	쿼리로 고민하지 말 것.
		테이블 내 특정 컬럼만 조회해도 되는 경우 
		모든 컬럼을 조회할 때 하단 코드처럼 하던가
			select (*) from (table); 
		특정 컬럼값들을 조회할 때, 하단 코드처럼 하게 되는데 
			select (co1, col2, ...) from (table);		
		조회하고자 하는 데이터들이 수십, 수백만 건 이라도 성능차이가 크지 않다.
		맘편하게 아스타로 조회하고, 조회한 데이터를 핸들링하자.

---

220609 코드 리뷰 및 qna 메모
	
	암호화 해싱
		해싱 기본 개념 
		"password1234" ------------------------------------> "34SavAD@F23#BJKLF!..."
			(평문)		  단방향해싱			(다이제스트_digest)


		키 스트레칭
		"password1234" ------------------------------------> "34SavAD@F23#BJKLF!..."
			(평문)		  xN 회 해싱			(다이제스트_digest)
			하는 이유? 
			-> 브루트 포스 방지(무차별 대입 공격)
				-> 해싱 특성상 어떠한 값을 무작위로 넣다보면 언젠간(?) 맞을텐데 그것을 방지함. 
		
	jasypt
		개발자가 암호화에 대해 깊은 지식이 없더라도
		암호화를 할 수 있게 돕는 java lib

	datasource의 설정 트렌드?
		요즘은 datasource의 설정을 
		application context에서 하는 것이 아닌
		.properties(.yml)에서 하는 것이 트렌드.

	entity? vo? domain? dto?
		비슷한 듯, 같은 역할을 하듯 
		크고 작은 차이점이 있음.
		
			entity
			테이블과 1:1 매칭되는 객체로,
			가장 core 한 객체로 볼 수 있음.
			jpa에서 사용되는 개념.
				mybatis를 사용할 땐
				entity, vo, dto를 정확히 구분하여 개발하는 것이 
				생산성이 더 떨어진다고 보고 있음. 
			요청이나 응답에 사용되어서는 안되며 만약 @entity를 사용하여 
			entity 코드를 작성했다면 dto 객체를 만들어 
			requestDto / responseDto 객체로 계층 처리를 해야한다. 
			BoardEntity -> .toEntity
						
			vo
			값이나 특성을 지니고 있는 객체
			계산식 같은 메서드를 포함하고 있을 수 있음.
			get, set으로 컨트롤 할 수 있음
			
			dto
			계산식 등 로직을 갖고 있지 않다.
			계층간 이동만을 목적으로 하는 객체
			immutable 특성.
				
			domain
			final 클래스 사용 
			
		결론
			그동안 매개변수나, 리턴을 vo로 컨트롤 했다면
			환경에 따라 인자를 reqDto, 리턴을 resDto로 
			view -> controller -> service -> repository(abstract JpaRepository) -> db ctrl
	
	@Transactional
		service단에서 주로 사용되며,
		자동으로 crud / 오류 발생시 롤백 / 커밋 까지 해주는 어노테이션
			
	@Pathvariable
		@Requestmapping의 정의 부분과 
		method parameter부분에 정의하여 사용 가능
		
	만약 디펜전시의 버전을 내릴때, 다른 디펜전시 간의 호환성 오류가 확인될때?
		한 두개의 디펜전시를 손보는 경우라면 상황이 간단히 해결 되겠지만 
		몇십개의 디펜전시를 버전 확인 해야 하는 경우 
		답이 없다. 노가다 시작임. 
		하나씩 낮추거나 높이며 테스트 돌려봐야한다.
		화이팅.
	
	https://caniuse.com/
		브라우저 버전별로 라이브러리 등등 구동가능/불가 여부를 표상태로 알려준다.
		만약 내 코드가 환경에서 동작하지 않는다면 한번쯤 테스트 돌려보는 것도 좋을 것 같다.
	
	gis engine openlayers, leaflet
		현 openlayers 를 gis 할때 많이 사용하지만 
		leaflet 또한 많이 사용한다.
	
	webpack bundling
		.html 내 들어가는 .js 파일을 하나의 js 파일로 만드는 것을 
		모듈 번들링이라고 하는데 그중 보편적으로 사용하는 것이 webpack 
			-> 걍 여러 js 파일을 하나의 js 파일로 만드는 것
	
		하는 이유?
			웹 페이지 성능 최적화
			같은 타입의 파일을 묶어 요청/응답을 받기 때문에 네트워크 코스트가 줄게 된다.
			
		결론 
			웹페이지 성능 최적화를 위해
			여러 js 파일을 하나의 js 파일로 번들링 하는 것.
			(html, css, png, jpg, js, ...) 가능

---

220608 10:21

	openlayers map 표출 및 webpack5 라인에서 앞단 표출 및 번들링 작업 후 
	서버 작업진행 하려 하였으나, 다음 과제 부여받음.

	다음 과제 : gisDashBoard
	기존 사내 프로젝트로 진행된 진주시 대시보드 프로젝트(퍼블리싱만 된 프로토타입)를 받아
	앞단, 서버단 구현할 것.
		+. gisDashBoard repository는 사내 로컬 서버.
	
	작업과제
		1. openlayers map 구현 
		2. 구현된 map 위 POI layer 신규 생성 및 표출
		3. evtList 표출 작업 
		...

---

220603 17:35

	localhost:xxxx에서 cannot get / 을 해결하고나니
	화면에 button은 정상 출력되나 map이 출력이 되지 않는 상황임. 해결중.

	-> 해결함.
	webpack.config.js 파일에 devMiddleware 설정이 잘못되어 있었음.
	
	devServer: {
	...
		devMiddleware: { publicPath: './dist' },
	...
	}
	해당 devMiddleware 설정은 
	번들된 파일을 다른 곳에서 찾으려고 하는 것을 방지하기 위함으로 
	webpack이 깔끔하게 정리된 블로그를 보고 참조해서 한 줄 넣었는데..
	해당 경로 설정으로 인해 오히려 번들 파일을 못찾고 있었다.

	차주엔 feature 들을 만들고 컨트롤 해 볼 예정이고
	최종적으로는 db까지 구성하여 서버단까지 같이 돌아가게 할 예정.

---

220603 16:24

	어제 localhost에 구동되지 않는 오류(지속적인 cannot get / 오류)를 해결함. 
	경로 상의 문제였음.
	template으로 설정한 index.html 파일이 /root 에 있었어야 했는데, 
	/src 에 넣어두고 같이 번들링을 돌려버렸다.
	template으로 설정했다는 것은 default view의 template 개념이었음.
	bundling 할 필요가 없었음. 

	localhost:xxxx에서 cannot get / 을 해결하고나니
	화면에 button은 정상 출력되나 map이 출력이 되지 않는 상황임. 해결중.
	
	local에 올라갔으니 상기이슈 해결 후 
	feature 들을 만들고 컨트롤 해 볼 예정임.

	추후엔 db까지 구성하여 서버단까지 같이 돌아가게 할 예정.

---

220602 20:24
	
	webpack5으로 번들링한 js, css, html 파일에 지도 출력.
	좌표계 EPSG:3857 -> EPSG:4326 변경.
	지도 상단 버튼 생성하여 home, company onclick시 evtLisnter -> map.Point('좌표')처리
	
	ps. 현재 모든 작업을 /dist 에 있는 index.html 템플릿을 우클릭 open with live server 해서 보고 있는데
	아무래도 localhost에 올라가지 않는게 (지속적인 cannot get / 오류) 이상해서 찾아보니까
	webpack 5점대 에서 dev server 관련 구동 이슈가 있다는 것을 확인함.
	webpack 4점 라인으로 다운 그레이드를 시도하려 하였으나, 연관 다른 디펜전시와 충돌이 일어나는 것을 확인함. 
	build, run 둘다 안되는 상황. 이슈 해결 중.

---

220602 10:20

	대과제 : react, typeScript 번들링 -> node.js / spring
	npm + typeScript
	
	소과제 : react + typeScript 형태로 ol 띄워볼 것.
	
	npm, yarn 설치 및 vite, webpack구동 가능
	tsconfig.json, webpack.config.js 설정

	js, css 파일 번들링 작업 완료
  
  ---
  
  220531

	과제 진행중 : react, typeScript 번들링 -> node.js / spring
	
		npm + typeScript
		webpack bundling
		(* bundling : 모듈화된 소스 코드를 브라우저에서 실행할 수 있는 파일로 한데 묶어 연결해주는 작업)
		
		react + typeScript 형태로 ol 띄워볼 것.

		현재 
		타입스크립트 감 잡는 중
		ts 감 잡고 ol 적용 예정.
		
		npm, yarn 설치 및 vite 구동 가능
		tsconfig 설정 하면서 감 잡는 중

		작업해보면서 npm 명령어를 사용해야 하는 일이 생겼는데,
		linux명령어에 대해 좀 감을 잡았다.
		(실수로 c/window/system32 에 연습용 tsproject를 생성하고 삭제하긴 했지만)

---

220524

	[BoardEntity deleteYn Column]

		BoardEntity 내 컬럼을 
		char boardDeleteYn; 을 두면서 

		실제 db에서 
		delete query를 날리는 것이 아니라 
		update query를 날려 boardEntity 'N' 값을 'Y' 으로 바꾸고, 
		List<BoardEntity> boardList = boardService.findByBoardList();
		할때 특정 컬럼값을 'Y'인 것은 제외하고 'N'인 boardEntity 들만 조회하는 경우도 있는데 

		어느 상황에서 실제 db를 삭제하고,
		어느 상황에서 deleteYn 처리를 하는가?

			-> 데이터를 보존해야할 이유가 있거나, 
			기획 단계에 따라 차이가 있을 수 있다.

			가령 회원 탈퇴의 경우 
			회원 탈퇴시 회원탈퇴 관련 고지의 의무가 있는데,
			고지 내 데이터를 특정 기간동안 보존해야 할 의무가 있다면 
			특정 컬럼이 'Y'가 되고 난 이후 고지한 기간동안만 보관하고 삭제처리 할 수도 있다.

			필요없는 데이터라면 즉시 삭제 가능

		결론 : deleteYn 처리를 하는 것은 "기획 차이 혹은 데이터의 보존 여부" 차이 

	[update를 할 때 페이지 하나를 더 주는 것이 옳은가?]
	 == 옳다 그르다 할 순 없다. 이것도 기획차이

		-> 기존 소스 코드 상황
		(update.html을 새로 만들어 페이지를 주지 않고
		viewBoard.html 에서 preUpdate btn을 onclick했을때 
		기존 preUpdate btn, delete btn을 display: none처리를 하고 
		신규 goUpdate btn, cancel btn를 display: inline-block 처리.
		goUpdate btn을 onclick했을때 심어둔 .js 파일내 update() fetch(ajax) 동작)
		
			// 상기 상황에서의 개선 방향 
			동일한 값의 2개의 태그를 주고 
			각각의 태그 내 속성 style="display:none"; 을 컨트롤 하는 것이 아니라 

			viewBoard 할 때 
			값 자체를 input 태그 내 넣어두고 disabled를 설정한다.

			update를 할 때에는 disabled 를 해제하는 식도 가능하다.
			그렇다면 태그 하나를 더 안써도 된다. 

			(단, disabled를 했을때 bg-color가 달라지므로, 
			작업 공간의 bg에 맞춰 색만 똑같이 설정해주어야 한다.)
		
		그렇게도 처리 할 수 있다.
		코드에 정답은 없다. 
		기획에 맞춰서 작업하는 것이 좋다. 

	[html와 thymeleaf를 지향하는 이유]
	
		html이 아닌 .jsp로 할 수 있고, 
		코드를 더 복잡하게 짤 수 있지만 정말 간결하고 메서드당 기능 하나, 
		이런 식으로 코드를 작성하는 이유

		협업을 위해.
		내가 작성한 코드를 타인이 수정해야 하는 상황이라면?
		당장 타임리프의 태그를 쓰지 않는 이유도 
		태그 내 class, id 등 퍼블리셔 분의 작업 용이함을 위해.
		el은 최대한 걷어내고 기본에 충실하게 코드를 작성하는 것이 좋다. 
		순수한 코드 그 자체로 작성하는 것이 협업을 위해 옳을 수 있다.

	[builder 패턴] 

	[update query를 확인 후 피드백]
	
		간단한 쿼리라면 @query를 써서 처리할 수 있지만 쿼리가 복잡해질 경우 
		클래스로 처리하는 것이 옳을 수 있다. 

	JINJU(GIS 대시보드 개발)
		- jsp 개발 
		
	ULJU(GIS 대시보드 개발)
		- kotlin
		- thymeleaf
		- js, main
		- .yml

	[etc]

		1. VO / DAO
			VO == entity / domain 
			DAO == DAO / repository
			
			+ result/response
