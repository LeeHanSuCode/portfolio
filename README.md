# 이한수 포트폴리오

## :pushpin: Intro
- 새로운 지식을 습득하고 활용하는 과정에서 즐거움을 느낍니다.
- 기술의 원리를 파악하려고 노력하고, 정리하는 습관이 있습니다.
- 예상치 못한 문제가 발생했을 때 혹은 더 나은 서비스를 위해 항상 학습합니다.
- 다양한 관점에서 바라보고 접근하기 위해 모임을 활용하여, 사람들과 함께 학습하고 공유합니다.
</br>

## :pushpin: Contact
- 이메일: dlsdn857758@gmail.com
- 블로그: https://velog.io/@dhfl0710
- 깃헙: https://github.com/LeeHanSuCode

</br>

--------------------------------------------------------------
# :pushpin: goQuality
>게시판 프로젝트 1
>(https://github.com/LeeHanSuCode/board-study) 

</br>

## 1. 제작 기간 & 참여 인원
- 2022년 5월 1일 ~ 5월 30일
- 개인 프로젝트

</br>

## 2. 사용 기술
#### `Back-end`
  - Java 11
  - Spring Boot 2.6.7
  - Gradle
  - Spring Data JPA
  - QueryDSL
  - Oracle 18c
  
#### `Front-d`
  - html
  - javascript
</br>

## 3. ERD 설계
![20220603_195928](https://user-images.githubusercontent.com/101684811/171841579-972eac4f-430b-44fd-b017-6a82828b6ca1.png)

## 4. 핵심 기능
이 서비스의 핵심 기능은 컨텐츠 등록 기능입니다. 

### 4-1.게시판 프로젝트1
- thymeleaf를 이용하여 view를 만들었습니다.
- 댓글 기능은 비동기 통신을 고려하여 Ajax를 이용하여 구현하였습니다.
- 게시글 등록시 파일 업로드와 게시글 보기에서 다운로드 기능을 구현하였습니다.
- JPA를 도입하여 , ORM 기술을 사용해보았고 , QueryDsl을 이용하여 동적 쿼리 기능을 구현하였습니다.

### 4-2.게시판 프로젝트2
- 앞선 프로젝트와 ERD와 기능은 같습니다.
  단,  REST API 방식으로 구현하였습니다. 
- 전체적으로 단위 테스트와 통합 테스트를 진행하였습니다.
- test와 함께 문서화까지 관리가능한 Spring Rest Docs를 도입하여 응답 값에 대한 문서화를 구현하였습니다.

- 예외

</br>

## 5. 핵심 트러블 슈팅
### 5.1 게시판 프로젝트1 에서의 핵심 트러블 슈팅

#### 1) 엔티티 조회시 연관 관계 엔티티는 따로 쿼리를 날려 조회하는 문제.

<details>
<summary><b>해결</b></summary>
<div markdown="1">

- 회원 엔티티 조회시 , 연관 관계로 있는 게시글을 한번에 가져오지 않고
 쿼리를 2번 날려 조회해오는 것을 확인하였습니다.

~~~java

    @Query("select m from Member m left join fetch m.boardList where m.id=:id")
    public Optional<Member> findByFetchId(@Param("id") Long id);
  ~~~

fetch join을 활용하여 한번에 조회할 수 있도록 해결하였습니다.  

</div>
</details>

#### 2) 회원 삭제시 수 많은 쿼리 전송.

<details>
<summary><b>기존 코드</b></summary>
<div markdown="1">

  -회원 삭제시 회원이 작성한 게시글과 댓글을 삭제해야 했습니다.
   또한 , 게시글마다 있는 댓글과 파일 또한 삭제가 필요했습니다.

//MemberService
~~~java

    //회원 삭제 작업
    @Transactional
    public void removeMember(Long id){
        Member member = memberRepository.findByFetchId(id)
                .orElseThrow(() -> new MemberException("존재하지 않는 회원 입니다."));

        //회원이 작성한 게시글을 삭제
        for(Board b :  member.getBoardList()){
            deletedByMember(b);			
            boardRepository.delete(b);
        }

        memberRepository.delete(member);			
    }

  
  //게시글과 연관된 파일과 댓글 삭제.
 private void deletedByMember(Board board){			
        //게시글 삭제
        if(board.getFileStores().size()>0){
            for(FileStore f : board.getFileStores()){
                fileStoreRepository.delete(f);
            }
        }

        //댓글 삭제
        if(board.getComments().size() > 0){
            for(Comments c : board.getComments()){
                commentsRepository.delete(c);       
            }
        }
    }
~~~
  
  
</div>
</details>
  
 
 

 <details>
<summary><b>해결 코드</b></summary>
<div markdown="1">
  
  게시글을 삭제할 때마다 그와 연관된 댓글과 파일들의 수만큼 delete 쿼리가 날라가는 문제가 발생하였습니다.
 이는 spring data jpa가 기본으로 제공하는 delete를 이용하여 삭제한 것이 원인이 되어 , 
 JPA 벌크 연산을 이용하여 문제를 해결하였습니다.
  
  //MemberService
  ~~~java
    @Transactional
    public void removeMember(Long id){
        Member member = memberRepository.findByFetchId(id)
                .orElseThrow(() -> new MemberException("존재하지 않는 회원 입니다."));
        
  
        //회원이 작성한 게시글을 삭제
        for(Board b :  member.getBoardList()){
            deletedByBoard(b);
            boardRepository.delete(b);
        }

        //회원이 작성한 댓글 삭제
        deletedByMember(member);

        memberRepository.delete(member);
    }

  
  
    //삭제되는 게시글과 연관된 파일과 댓글 삭제
    private void deletedByBoard(Board board){
        //게시글 삭제
        if(board.getFileStores().size()>0){
            fileStoreRepository.deletedByBoard(board);
        }

        //댓글 삭제
        if(board.getComments().size() > 0){
            commentsRepository.deletedByBoard(board);
        }
    }

  
    //삭제되는 회원과 연관된 댓글 삭제
    private void deletedByMember(Member member){
        if(member.getCommentsList().size() > 0){
            commentsRepository.deletedByMember(member);
        }
    }
  ~~~
  
  
  //FileStoreRepository
  ~~~java
  
    //게시글에 있는 파일 삭제
    @Modifying
    @Query("delete from FileStore f where f.board = :board")
    public int deletedByBoard(@Param("board") Board board);
  
  ~~~
  
  
 //CommentesRepository
  ~~~java
  
     //회원이 작성한 댓글 삭제
    @Modifying
    @Query("delete from Comments c where c.member =:member")
    public int deletedByMember(@Param("member")Member member);
  
    //게시글에 작성된 댓글 삭제
    @Modifying
    @Query("delete from Comments c where c.board =:board")
    public int deletedByBoard(@Param("board")Board board);
  ~~~
  
  </div>
</details>

### 5.2 게시판 프로젝트2(Restful) 에서의 핵심 트러블 슈팅

#### 1) API Validation 예외처리

<details>
<summary><b>기존 코드</b></summary>
<div markdown="1">

//MemberService
~~~java
//controller

@PostMapping
public ResponseEntity<UpdateMemberDto> join(@RequestBody @Valid JoinMemberDto joinMemberDto){
	

        
        Member joinMember = memberService.join(joinMemberDto);

        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(joinMember.getId())
                .toUri();


        return ResponseEntity.created(location).body(
                UpdateMemberDto.builder()
                        .id(joinMember.getId())
                        .userId(joinMember.getUserId())
                        .username(joinMember.getUsername())
                        .email(joinMember.getEmail())
                        .tel(joinMember.getTel())
                        .build());
}
~~~

~~~java
//회원 가입 검증용 DTO

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor(access = AccessLevel.PRIVATE)
@Builder
public class JoinMemberDto {
    private Long id;

    @NotBlank
    @Size(min = 2 , max = 4)
    private String username;

    @NotBlank
    @Pattern(regexp = "[a-zA-Z0-9]{8,20}")
    @Size(min = 8 , max = 20)
    private String userId;

    @NotBlank
    @Pattern(regexp = "^(?=.*[A-Za-z])(?=.*\\d)(?=.*[~!@#$%^&*()+|=])[A-Za-z\\d~!@#$%^&*()+|=]{8,16}$")
    @Size(min = 8,max = 16)
    private String password;

    private String password2;

    @Email
    private String email;

    private String tel;

    private LocalDateTime createdDate;

}

~~~

~~~java
//전반적인 예외처리 담당 클래스

@Slf4j
@RestController
@ControllerAdvice
public class ApiExceptionController extends ResponseEntityExceptionHandler {
	
	 @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers, HttpStatus status, WebRequest request) {

        Map<String, Object> body = new LinkedHashMap<>();
        
        body.put("timestamp", occurExceptionTime());
        body.put("status", status.value());
        body.put("path",request.getDescription(false));

        List<Map> fieldErrors = ex.getBindingResult().getFieldErrors()
                .stream().map(
                        fe ->{
                            HashMap errorInfo = new HashMap();
                            
                            errorInfo.put("rejectedValue" , fe.getRejectedValue());
                            errorInfo.put("fieldName" , fe.getField());
                            errorInfo.put("message" , fe.getDefaultMessage());

                            return errorInfo;
                        }
                ).collect(Collectors.toList());


        body.put("fieldErrors", fieldErrors);

        return new ResponseEntity<>(body,status);
    }

}


  //에러 발생한 시간 반환(format)
    private String occurExceptionTime() {
        return LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    }

~~~
  
  ~~~java
  //출력결과(postman)
     "timestamp": "2022-07-05 02:29:13",
    "status": "BAD_REQUEST",
    "path": "uri=/members",
    "fieldErrors": 
       {
            "rejectedValue": "hslee",                 //rejectedValue와 fieldName의 중복 문제.
            "fieldName": "userId",                    //fieldErrors 내부에서 다시 내부로 들어가 fieldName값을 확인해야만 어떠한 필드의 문제인지 파악가능하다는 문제.
            "message": "크기가 8에서 20 사이여야 합니다"
        },
        {
            "rejectedValue": "hslee",
            "fieldName": "userId",
            "message": "\"[a-zA-Z0-9]{8,20}\"와 일치해야 합니다"
        }
  ~~~
    
    #### 문제
  - ResponseEntityExceptionHandler를 상속하여 , handleMethodArgumentNotValid 메소드를 재정의하여 사용하였습니다.
   BeanValidation에 의한 유효성 검증은 잘되었으나 , 필드 2개이상의 값을 비교하여 처리해야 하는 ObjectError까지 처리할 수는 없었습니다.
   
  - 반환 데이터 형식에 문제가 있어 , 예외 정보는 내부를 확인해야 어떠한 필드의 데이터인지 알 수 있었으며 
   같은 필드에 여러 검증 문제가 발생하였을 경우 각기 예외 메세지가 다르다 보니 , 중복데이터가 발생하는 문제가 있었습니다.
   
   아래의 출력처럼 표현하고 싶었습니다.
  
  ~~~java
    "timestamp": "2022-07-05 02:29:13",
    "status": "BAD_REQUEST",
    "path": "uri=/members",
    "fieldErrors": 
      "userId" :{
            "rejectedValue": "hslee",
            "fieldName": "userId",
            "message": [
                  "userId은 8 ~ 20글자 사이로 입력해 주세요.",
                   "영어와 숫자로만 구성해주세요."
                   ]
        }
  ~~~
  
</div>
</details>
  

 <details>
<summary><b>해결 코드</b></summary>
<div markdown="1">
  
  ~~~java
      //controller
       @PostMapping
    public ResponseEntity<UpdateMemberDto> join(@RequestBody @Valid JoinMemberDto joinMemberDto ,BindingResult bindingResult){

        if(!joinMemberDto.getPassword().equals(joinMemberDto.getPassword2())){
            bindingResult.rejectValue("password","NotEquals","비밀번호가 일치하지 않습니다");
        }

        if(bindingResult.hasErrors()){
            throw new ValidationNotFieldMatchedException(bindingResult);
        }

        Member joinMember = memberService.join(joinMemberDto);

       URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(joinMember.getId())
                .toUri();


        return ResponseEntity.created(location).body(
                UpdateMemberDto.builder()
                        .id(joinMember.getId())
                        .userId(joinMember.getUserId())
                        .username(joinMember.getUsername())
                        .email(joinMember.getEmail())
                        .tel(joinMember.getTel())
                        .build());
    }
 

  ~~~    
 - BindingResult를 파라미터로 사용하였습니다.
   대신 , 재정의한 handleMethodArgumentNotValid 메소드가 호출되지 않아 새로운 custom예외를 만들어 예외가 있을 경우 ,호출되도록 처리하였습니다.
 
 - @ControllerAdvice에서 잘못 입력된 값을 꺼내올 수 있게 하기 위해서 , password불일치 예외를 rejectValue로 등록하였습니다.
 
 ~~~java
 //custom예외
 public class ValidationNotFieldMatchedException extends RuntimeException{

    private BindingResult bindingResult;

    public ValidationNotFieldMatchedException(BindingResult bindingResult){
        this.bindingResult = bindingResult;
    }

    public BindingResult getBindingResult() {
        return bindingResult;
    }
}
 ~~~
 - @ControllerAdvice에서 BindingResult를 사용하기 위해 , 해당 예외의 생성자로 주입받아 사용하였습니다.
 
 ~~~java
 //예외 정보를 담아줄 클래스
@Getter
@Builder
public class ValidationErrorResponse {

    private List<String> messages;
    private String fieldName;
    private String rejectedValue;
}

 ~~~
 - 반환 데이터인 json의 계층 구조를 표현할 때, Map을 연달아 사용하기에 코드의 가독성이 우려되어 객체를 따로 생성하였습니다.
  또한 , 중복된 필드의 경우 메세지를 같은 객체에 담아주기 위해 message는 List를 이용하였습니다.
 
 
 ~~~java
 //전반적인 예외처리 담당 클래스
 @Slf4j
@RestController
@ControllerAdvice
public class ApiExceptionController extends ResponseEntityExceptionHandler {
	
  @ExceptionHandler
    public ResponseEntity<Object> handleValidationNotFieldMatchedException(
            ValidationNotFieldMatchedException ex, WebRequest request) {

        Map<String, Object> body = new LinkedHashMap<>();
        body.put("timestamp", occurExceptionTime());
        body.put("status",HttpStatus.BAD_REQUEST);
        body.put("path",request.getDescription(false));

          Map<String ,ValidationErrorResponse> filedErrorsInfo = new HashMap<>();


          ex.getBindingResult().getFieldErrors()
                  .stream().forEach(fe -> {

                                if(filedErrorsInfo.containsKey(fe.getField())){

                                    filedErrorsInfo.get(fe.getField()).getMessages().add(getMessageSource(fe));

                                }else{
                                    ValidationErrorResponse validationErrorResponse = ValidationErrorResponse.builder()
                                            .fieldName(fe.getField())
                                            .rejectedValue(getRejectedValue(fe))
                                            .messages(new ArrayList<>())
                                            .build();

                                    validationErrorResponse.getMessages().add(getMessageSource(fe));

                                    filedErrorsInfo.put(fe.getField() , validationErrorResponse);
                                }
                          });

        body.put("fieldErrors", filedErrorsInfo);

        return new ResponseEntity<>(body,HttpStatus.BAD_REQUEST);
    }


     //거절된 값을 얻어온다.
    private String getRejectedValue(FieldError fe) {
        String rejectedValue = null;

        if(fe.getRejectedValue() == null){
            rejectedValue = "값이 들어오지 않음";
        }else{
            rejectedValue = fe.getRejectedValue().toString();
        }
        return rejectedValue;
    }

    //error 메세지를 얻어온다.
    private String getMessageSource(FieldError fe) {
        return Arrays.stream(Objects.requireNonNull(fe.getCodes()))
                .map(c -> {
                    try {
                        Object[] argument = fe.getArguments();
                        return messageSource.getMessage(c, argument, null);
                    } catch (NoSuchMessageException e) {
                        return null;
                    }
                }).filter(Objects::nonNull)
                .findFirst()
                .orElse(fe.getDefaultMessage());
    }
 ~~~
 - 중복되는 메세지와 재사용성을 고려하여 , MessageResolver가 생성해주는 code값을 가지고 , MessageSource를 이용하였습니다.
 
 - null값이 들어간 경우 , rejectedValue로 값을 꺼내올 때 NPE가 발생할 수 있으므로 따로  getRejectedValue 라는 메소드를 구현하여 처리하였습니다.

  </div>
</details>

#### 2) QueryDsl에서 동적쿼리 정렬 조건 처리

 <details>
<summary><b>기존 코드</b></summary>
<div markdown="1">

- Admin 페이지를 개발하던 도중 queryDsl을 이용한 동적쿼리를 작성하였습니다.
   pageable 객체에서 정렬 조건을 가져오고 싶었습니다. 또한, 유효하지 않은 정렬 조건이 넘어왔을 경우에는 default값으로 Board의 식별자인
   id값을 이용하여 내림차순 정렬을 시켜주고자 하였습니다.

~~~java
//AdminBoardRepositoryImpl
    @Override
    public Page<Board> findByCond(Pageable pageable, SearchConditionDto searchConditionDto) {

        List<Board> content = jpaQueryFactory
                .selectFrom(board)
                .join(board.member, member).fetchJoin()
                .where(
                        userIdCond(searchConditionDto.getUserId()),
                        subjectCond(searchConditionDto.getSubject())
                ).offset(pageable.getOffset())
                .limit(pageable.getPageSize())             
                .fetch();



        //countQuery
        Long count = jpaQueryFactory
                .select(board.count())
                .from(board)
                .where(
                        userIdCond(searchConditionDto.getUserId()),
                        subjectCond(searchConditionDto.getSubject())
                ).fetchOne();

        return new PageImpl<>(content,pageable,count);
    }


     //검색 조건
     //사용자 id를 포함하는 게시글만 가져온다.
    private BooleanExpression userIdCond(String userId){
        return StringUtils.hasText(userId) ? board.member.userId.contains(userId) : null;
    }
    //검색 조건
    //제목을 포함하는 게시글만 가져온다.
    private BooleanExpression subjectCond(String subject){
        return StringUtils.hasText(subject) ? board.subject.contains(subject) : null;
    }
~~~

</div>
</details>


 <details>
<summary><b>해결 코드</b></summary>
<div markdown="1">
 - orderby() 메소드가 받는 파라미터의 타입을 확인하니 , OrderSpecifier라는 클래스였습니다.
   그리하여 , OrderSpecifier 클래스의 인스턴스를 제가 원하는 정렬 조건에 맞게 만든 후에 생성하여 이용하였습니다.
   
   ~~~java
   //AdminBoardRepositoryImpl
   
    @Override
    public Page<Board> findByCond(Pageable pageable, SearchConditionDto searchConditionDto) {

        List<OrderSpecifier> allOrderSpecifiers = getAllOrderSpecifiers(pageable);	


        List<Board> content = jpaQueryFactory
                .selectFrom(board)
                .join(board.member, member).fetchJoin()
                .where(
                        userIdCond(searchConditionDto.getUserId()),
                        subjectCond(searchConditionDto.getSubject())
                ).offset(pageable.getOffset())
                .limit(pageable.getPageSize())
                .orderBy(allOrderSpecifiers.stream().toArray(OrderSpecifier[]::new))
                .fetch();



        //countQuery
        Long count = jpaQueryFactory
                .select(board.count())
                .from(board)
                .where(
                        userIdCond(searchConditionDto.getUserId()),
                        subjectCond(searchConditionDto.getSubject())
                ).fetchOne();

        return new PageImpl<>(content,pageable,count);
    }
    
    
         //검색 조건
     //사용자 id를 포함하는 게시글만 가져온다.
    private BooleanExpression userIdCond(String userId){
        return StringUtils.hasText(userId) ? board.member.userId.contains(userId) : null;
    }
    //검색 조건
    //제목을 포함하는 게시글만 가져온다.
    private BooleanExpression subjectCond(String subject){
        return StringUtils.hasText(subject) ? board.subject.contains(subject) : null;
    }
    
   
   //추가된 코드(메소드구현)
     //정렬 조건 들을 생성.
    private List<OrderSpecifier> getAllOrderSpecifiers(Pageable pageable){

        List<OrderSpecifier> orderBys= new ArrayList<>();



        if(!pageable.getSort().isEmpty()){
            Sort sort = pageable.getSort();
            List<Sort.Order> sortOrder = sort.get().collect(Collectors.toList());

            Order direction = sortOrder.get(0).getDirection().isAscending() ? Order.ASC : Order.DESC;


            if(sortOrder.get(0).getProperty().equals("createdDate")){
                OrderSpecifier<?> orderByCreatedDate = QueryDslOrderUtilCustom.getSortedColumn(direction , board,"createdDate");
                orderBys.add(orderByCreatedDate);
            }

            if(sortOrder.get(0).getProperty().equals("readCount")){
                OrderSpecifier<?> orderByReadCount = QueryDslOrderUtilCustom.getSortedColumn(direction , board,"readCount");
                orderBys.add(orderByReadCount);
            }


            OrderSpecifier<?> orderById = QueryDslOrderUtilCustom.getSortedColumn(Order.DESC , board,"id");
            orderBys.add(orderById);

        }

        return orderBys;
    }
   ~~~
   
   - getAllOrderSpecifiers 해당 메소드는 pageable를 주입받아 , 내부적으로 따로 커스텀한 QueryDslOrderUtilCustom.getSortedColumn 를 이용하여 
     알맞는 정렬 조건을 가져옵니다.
     
   
   ~~~java
 //QueryDslOrderUtilCustom  
   
public class QueryDslOrderUtilCustom {

    public static OrderSpecifier<?> getSortedColumn(Order order , Path<?> parent , String fieldName){
        Path<Object> fieldPath = Expressions.path(Object.class, parent , fieldName);

        return new OrderSpecifier(order , fieldPath);
    }

}
   
   ~~~
</div>
</details>

## 6. 그 외 트러블 슈팅
<details>
<summary>생성자 자동 주입시 순환 참조</summary>
<div markdown="1">
  
-원인 : Service 계층들끼리 서로 참조하고 있어 문제가 발생. </br>
-해결 : 단방향 참조로 변경 . Service -> Repository 계층만 참조하도록 전체 구조 변경.
  
</div>
</details>


<details>
<summary>파일 다운로드</summary>
<div markdown="1">
  
-원인 : CONTENT_DISPOSITION 헤더의 부재.</br>
-해결 : ResponseEntity를 사용하여 응답에  contentDisposition = "attachment; filename 추가하여 해결
  
</div>
</details>

<details>
<summary>롬복 @Builder 생성자 문제</summary>
<div markdown="1">
  
-원인 : @Builder는 생성자가 없을 경우 , 모든 파라미터를 받는 생성자를 생성해줍니다.
        반면 ,JPA를 이용할 때 기본 생성자가 필요하여 @NoArgsConstructor 사용하였습니다. 이것이 원인이 되어
	@Builder는 생성자가 이미 있다고 판단하여 모든 멤버 변수를 갖는 생성자를 생성해주지 않았습니다.</br>
-해결 : @AllArgsConstructor를 이용하여 직접 생성해줌으로써 해결하였습니다.
</div>
</details>

<details>
<summary>롬복 @Builder - Jpa컬렉션 객체 초기화의 NPE 문제</summary>
<div markdown="1">
  
-원인 : @Builder는 클래스 레벨에 붙였을 때 , 모든 멤버변수를 이용하여 생성자를 생성합니다.
        이때 , JPA의 양방향 관계로 설정되어 컬렉션 필드는 기존에 초기화 해두었지만 , 이 또한 클래스가 가지고 있는 필드중 하나로 여겨져 따로 초기화 해주지 않을 경우
	Null이 주입되어 NPE가 발생하였습니다.
    </br>
-해결 : 초기화할 필드들만 따로 모아 생성자를 작성한 뒤 , 생성자 위에 @Builder 어노테이션을 붙여 , 양방향 연관관계로써 사용되는 컬렉션 필드에 NULL이 주입되는것을 방지하였습니다.
</div>
</details>


<details>
<summary>일대다 fetch join</summary>
<div markdown="1">
  
-원인 : 회원을 중심으로 게시글을 가져올 때 , 일 대 다 관계에서 fetch join을 이용하여 조회하는데 , 이 때 row 수의 문제가 발생하였습니다.</br>
-해결 : 다행히 회원을 통한 게시글의 조회였고 , 컬렉션 페치 조인도 1개만 사용하였기에 "distinct"를 넣어 해결 하였습니다.
  
</div>
</details>



<details>
<summary>HttpMediaTypeNotAcceptableException 예외</summary>
<div markdown="1">
  
-원인 : 객체를 Json으로 반환하는데 예외가 발생하였습니다.</br>
-해결 : Jackson이 Json으로 객체를 변환할 때 내부적으로 ObjectMapping API를 사용하여 객체를 변환합니다.
        그 변환 과정에서 Jackson 라이브러리는 Getter/Setter 프로퍼티를 기준으로 동작한다는 걸 알고 , 
        내부 클래스에 @Getter 어노테이션을 추가하여 해결하였습니다.

  
</div>
</details>

</br>

