# ToyProject

- [Server Side]
    - Vapor
    - Fluent
    - Postgres
- [iOS]
    - SwiftUI
    - Combine
- [Android]
    - Jetpack Compose
    - Hilt
    - Retogit, OkHttp3, Gson
    - androidx.navigation



---

### [Server Side] - Vapor + Postgres + Fluent

1. CRUD 구현
2. Restful API 아키텍처

---



- Controller
    
    ```swift
    class DerekItemsController: RouteCollection{
    
    		//: EndPoint 설정
    		func boot(routes: RoutesBuilder) throws {
    				//:  /derek/title
    				//:  /derek/items
    				let api = routes.grouped("derek").grouped(JsonWebTokenAuthenticator())
    			
    				//: GET
    		    api.get("title", use: getTitle)
    		    api.get("items", use: getItems)
    		
    		    //: POST
    		    api.post("title", use: saveTitle)
    		    api.post("items", use: saveItem)
    		}
    		
    		//: 생략..
    		
    		func saveTitle(req: Request) async throws -> BaseResponseDTO<DerekTitleResponseDTO>{
    		        let titleRequestDTO = try req.content.decode(DerekTitleRequestDTO.self)
    		        let derekTitle = DerekTitle(mainTitle: titleRequestDTO.mainTitle, headerFieldValue: titleRequestDTO.subTitle)
    		        
    		        try await derekTitle.save(on: req.db)
    		        
    		        guard let derekTitleResponseDTO = DerekTitleResponseDTO(derekTitle) else{
    		            return BaseResponseDTO(code: 500, status: "internalServerError", message: "fail", data: nil, errors: nil)
    		        }
    		        return BaseResponseDTO(code: 200, status: "ok", message: "성공", data: derekTitleResponseDTO, errors: nil)
    		    }
    
    //: 생략..
    		
    		func getTitle(req: Request) async throws -> BaseResponseDTO<DerekTitleResponseDTO>{
    		        let derekTitle = try await DerekTitle.query(on: req.db)
    		            .first()
    		            .flatMap(DerekTitleResponseDTO.init)
    
    		        return BaseResponseDTO(code: 200, status: "ok", message: "성공", timestamp: dateToIsoStrFormat(date: Date()), data: derekTitle, errors: nil)
    		    }
    
    //: 생략..
    
    }
    ```

- Middleware
 
    ```swift

    struct JsonWebTokenAuthenticator: AsyncRequestAuthenticator{
	    func authenticate(request: Request) async throws {
        	try request.jwt.verify(as: AuthPayload.self)
    	}
	}
    

	struct AuthPayload : JWTPayload {
    	typealias Payload = AuthPayload
    
	    enum CodingKeys : String, CodingKey {
    	    case subject = "sub"
	        case expiration = "exp"
        	case userId = "uid"
    	}

    	var subject : SubjectClaim
	    var expiration : ExpirationClaim
    	var userId : UUID
    
	    func verify(using signer: JWTSigner) throws {
        	try self.expiration.verifyNotExpired()
    	}
	}

    ```
 
- Migration
    
    ```swift
    class CreateDerekTitleTableMigration : AsyncMigration {
        
        func prepare(on database: Database) async throws {
            try await database.schema("derek_title")
                .id()
                .field("mainTitle", .string, .required)
                .field("subTitle", .string, .required)
                .create()
        }
        
        func revert(on database: Database) async throws {
            try await database.schema("derek_title")
                .delete()
        }
    }
    
    ```
    
- Model
    
    ```swift
    import Foundation
    import Vapor
    import Fluent
    
    final class DerekTitle : Model, Content, Validatable{
        static let schema: String = "derek_title"
        
        @ID(key: .id)
        var id : UUID?
        
        @Field(key: "mainTitle")
        var mainTitle : String
        
        @Field(key: "subTitle")
        var subTitle : String
        
        init() {}
        
        init(id : UUID? = nil, mainTitle: String, headerFieldValue: String) {
            self.id = id
            self.mainTitle = mainTitle
            self.subTitle = headerFieldValue
        }
        
        static func validations(_ validations: inout Validations) {
            validations.add("mainTitle", as : String.self, is: !.empty, customFailureDescription: "값이 없습니다.")
            validations.add("subTitle", as : String.self, is: !.empty, customFailureDescription: "값이 없습니다.")
        }
    }

    ```
    
- Configure
    
    ```swift
    
    public func configure(_ app: Application) async throws {
    //: 생략..
        app.databases.use(.postgres(hostname: "localhost", username: "*******", password: "*******", database: "api*****appdb"),
                          as: .psql)
    //: 생략..
        app.migrations.add(CreateDerekTitleTableMigration())
    //: 생략..
        try app.register(collection: DerekItemsController())
        
        app.jwt.signers.use(.hs256(key: "****_*******"))
        
        try routes(app)
    }
    
    ```
    

### EndPoint 결과
<img width="972" alt="endpoint_result_db" src="https://github.com/Gugoon/DerekToyProject/assets/10485667/7b44ce3b-6dfd-439b-824c-ee794de90999">


<img width="1286" alt="endpoint_result_json" src="https://github.com/Gugoon/DerekToyProject/assets/10485667/1c5d4c96-824a-4ae6-976d-3fc74a53c27a">



---

### [iOS] SwiftUI, Combine

1. MVVM
2. Restful API

---
![iOS_result](https://github.com/Gugoon/DerekToyProject/assets/10485667/6c507d5d-f875-4439-bc40-73b161f2c83b)

- View
    
    ```swift
    import SwiftUI
    
    struct ContentView: View {
        @EnvironmentObject var derekViewModel : DerekViewModel
        
        @State var currentItem: DerekModel?
        @State var showDetailPage: Bool = false
        // Matched Geometry Effect
        @Namespace  var animation
        
        @State var animateView: Bool = false
        @State var animateContent: Bool = false
        @State var scrollOffset: CGFloat = 0
        
        var body: some View {
            ScrollView(.vertical, showsIndicators: false) {
                VStack(spacing: 30) {
                    headerView
                        .padding(.horizontal)
                        .padding(.bottom)
                        .opacity(showDetailPage ? 0 : 1)
                    cardListView
                }
                .padding(.vertical)
            }//: SCOLL_VIEW
            .overlay {
                if let currentItem = currentItem, self.showDetailPage {
                    DetailView(item: currentItem)
                        .ignoresSafeArea(.container, edges: .top)
                }
            }
            .background(alignment: .top) {
                RoundedRectangle(cornerRadius: 15, style: .continuous)
                    .fill(Color(.tertiarySystemBackground))
                    .frame(height: animateView ? nil : 350, alignment: .top)
                    .opacity(animateView ? 1 : 0)
                    .ignoresSafeArea()
            }
            .onAppear{
                Task{
                    await getDataInfos()
                }
            }
        }//: BODY
        
        func getDataInfos() async{
            let derekTitle = await self.derekViewModel.getTitelData()
            if derekTitle.statusCode == .SUCCESS{
                let _ = await self.derekViewModel.getItemslData()
            }else{
                Log("#### derekTitle  \(derekTitle.statusCode) ")
            }
        }
    }
    ```
    
- DataModel
    
    ```swift
    struct DerekItemResponseDTO : Identifiable, Codable{
        let id : UUID
        let appName: String?
        let appDescription: String?
        let appLogo: String?
        let bannerTitle: String?
        let platformTitle: String?
        let artwork: String?
        let appDetailDescription: String?
    }
    
    struct DerekTitleResponseDTO : Codable{
        let mainTitle : String?
        let subTitle : String?
    }
    ```
    
- ViewModel
    
    ```swift
    **class** DerekViewModel : DefaultViewModel {
    			var headerData : DerekTitleResponseDTO?{
    		        didSet{
    		            self.mainTitle = headerData?.mainTitle ?? ""
    		            self.subTitle = headerData?.subTitle ?? ""
    		        }
    		   }		    
    		   var derekItems : [DerekItemResponseDTO]?{        
    //: 생략..
    		   }
    }
    ```
    
- Network - API
    
    ```swift
    func getTitelData() async -> APICallbackDataModel{
    //: 생략..
                if let url = URL(string: baseURL + "/title"){
                    CombineAPI.fetch(request: self.commonHeader(url: url, method: .GET))
                        .sink(receiveCompletion: { completion in
                            switch completion {
                            case .finished:
                                break
                            case .failure(let error):
                                continuation.resume(returning: APICallbackDataModel(statusCode: .ERROR, message: error.localizedDescription))
                            }
                        }, receiveValue: {  data in
                            if let resData = try? JSONDecoder().decode(BaseModel<DerekTitleResponseDTO>.self, from: data){
                                self.headerData = resData.data
                                continuation.resume(returning: APICallbackDataModel(statusCode: .SUCCESS, message: ""))
                            }
                        })
    //: 생략..
                }
    //: 생략..
        }
    ```
    

---

### [Android] Kotlin, Jetpack Compose, Hilt, Retrofit, OkHttp3

1. MVVM
2. Restful API

---

![android_result](https://github.com/Gugoon/DerekToyProject/assets/10485667/eab27691-bb8c-44f9-866a-6a900f6d2133)

- View
    
    ```kotlin
    @Composable
    fun MainView(derekViewModel: DerekViewModel = hiltViewModel()) {
    
    //: 생략..
    
        MaxSizeBox(
            modifier = Modifier
                .padding(bottom = 16.dp)
        ) {
            MaxWidthBox {
                LazyColumn(
                    modifier = Modifier,
                    state = scrollState
                ) {
                    item {
                        Spacer(modifier = Modifier.height(70.dp))
                    }
                    derekViewModel.derekItemsState.value.baseData?.data?.let { itemList ->
                        itemsIndexed(itemList) { index, item ->
                            CardView(
                                modifier = Modifier,
                                item = item,
                                isMovedCardView = isMovedCardView,
                                isShowDetailView = isShowDetailView,
                                selectedIdx = selectedIdx,
                                itemIdx = index,
                                listOffset = offset.value,
                                preOffset = preOffset
                            )
                        }
                    }
                }//: LAZY_COLUM
    
                AnimatedVisibility(
                    visible = isShowHeader.value,
                    enter = fadeIn(animationSpec = tween(500)),
                    exit = fadeOut(animationSpec = tween(500))
                ) {
                    Row(
                        modifier = Modifier
                            .background(Color.White)
                            .padding(16.dp)
                            .height(60.dp),
                        verticalAlignment = Alignment.CenterVertically
                    ) {
                        derekViewModel.derekTitleState.value.baseData?.data?.let {derekTitle ->
                            Column(modifier = Modifier) {
                                Text(
                                    text = "${derekTitle.subTitle}",
                                    fontWeight = FontWeight.SemiBold,
                                    textAlign = TextAlign.Left,
                                    color = Color.Gray
                                )
    
                                Text(
                                    text = "${derekTitle.mainTitle}",
                                    fontWeight = FontWeight.Bold,
                                    textAlign = TextAlign.Left,
                                    color = Color.Black,
                                    fontSize = 24.sp,
                                )
                            }//: COLUMN
                            Spacer(modifier = Modifier.weight(1f))
                            Image(
                                painter = painterResource(id = R.drawable.face_24px),
                                contentDescription = "face_24px",
                                modifier = Modifier
                                    .padding(start = 48.dp, end = 16.dp)
                                    .width(40.dp)
                                    .height(40.dp)
                                    .background(Color.Blue)
                            ) //: IMAGE
                        }
                    }//: ROW
                }
            } //: BOX
    
    //: 생략..
    
         }//: BOX
    //: 생략..
    }
    ```
    
- DataModel
    
    ```kotlin
    data class DerekTitleResponseDTO(val mainTitle: String?, val subTitle: String?)
    
    data class DerekItemResponseDTO(
        val id: String?,
        val appName: String?,
        val appDescription: String?,
        val appLogo: String?,
        val bannerTitle: String?,
        val platformTitle: String?,
        val artwork: String?,
        val appDetailDescription: String?
    )
    ```
    
- ViewModel
    
    ```kotlin
    class DerekViewModel @Inject constructor(
        private val derekRepository: DerekApiRepository
    ) : BaseViewModel() {
    
        init {
            launch {
                getDerekTitleData()
                getDerekItemsData()
            }
        }
        private suspend fun getDerekTitleData() {
            with(derekTitleState) {
                value = derekRepository.getDerekTitle()
                value.baseData.let { baseResponse ->
                    baseResponse?.let { data ->
                        if (data.isSuccessful() && data.toString().isNotEmpty()) {
                            isLoading.value = false
                        } else if (data.retryAuth()) { // 401, 403
    
                        } else if (data.conflictException()) {
                            exceptionMsg.value = data.message.toString()
                        } else {
                            exceptionMsg.value = data.message.toString()
                        }
                    }
                }
            }
        }
    //: 생략..
    }
    ```
    
- Network- API
    
    ```kotlin
    @Singleton
    interface DerekApi {
        @GET(value = "derek/title")
        suspend fun getDerekTitle(): BaseResponse<DerekTitleResponseDTO>
    
        @GET(value = "derek/items")
        suspend fun getDerekItems(): BaseResponse<Array<DerekItemResponseDTO>>
    }
    ```
    
- DI
    
    ```kotlin
    object AppModule {
    
    //: 생략..
    
        @Provides
        @Singleton
        fun provideDerekApi(): DerekApi {
            return Retrofit.Builder().baseUrl(BASE_URL)
                .client(
                    provideOkHttpClientAuth(
                        appInterceptor = AppInterceptor(),
                        domainInterceptor = AuthDomainInterceptor()
                    )
                )
                .addConverterFactory(GsonConverterFactory.create())
                .build().create(DerekApi::class.java)
        }
    
    //: 생략..
    
    }
    ```



