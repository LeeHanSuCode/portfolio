# 이한수 포트폴리오
>캐치 프레이즈

</br>

## :pushpin: Intro
(여기에 자기 소개)

</br>

## :pushpin: Contact
- 이메일: dlsdn857758@gmail.com
- 블로그: https://velog.io/@dhfl0710
- 깃헙: https://github.com/LeeHanSuCode

</br>

--------------------------------------------------------------
# :pushpin: goQuality
>게시판 프로젝트 
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
  
#### `Front-end`
  - html
  - javascript
</br>

## 3. ERD 설계
![20220603_195928](https://user-images.githubusercontent.com/101684811/171841579-972eac4f-430b-44fd-b017-6a82828b6ca1.png)


## 4. 핵심 기능
이 서비스의 핵심 기능은 컨텐츠 등록 기능입니다.  
이 단순한 기능의 흐름을 보면, 서비스가 어떻게 동작하는지 알 수 있습니다.  

</br>

## 5. 핵심 트러블 슈팅
### 5.1. 엔티티 조회시 연관 관계 엔티티는 따로 쿼리를 날려 조회하는 문제.

- 회원 엔티티 조회시 , 연관 관계로 있는 게시글을 한번에 가져오지 않고
 쿼리를 2번 날려 조회해오는 것을 확인하였습니다.

<details>
<summary><b>해결</b></summary>
<div markdown="1">

~~~java

    @Query("select m from Member m left join fetch m.boardList where m.id=:id")
    public Optional<Member> findByFetchId(@Param("id") Long id);
  ~~~

fetch join을 활용하여 한번에 조회할 수 있도록 해결하였습니다.  

</div>
</details>

### 5.2. 회원 삭제시 수 많은 쿼리 전송.
  
  -회원 삭제시 회원이 작성한 게시글과 댓글을 삭제해야 했습니다.
   또한 , 게시글마다 있는 댓글과 파일 또한 삭제가 필요했습니다.
  
  

<details>
<summary><b>기존 코드</b></summary>
<div markdown="1">

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
  
 
 게시글을 삭제할 때마다 그와 연관된 댓글과 파일들의 수만큼 delete 쿼리가 날라가는 문제가 발생하였습니다.
 이는 spring data jpa가 기본으로 제공하는 delete를 이용하여 삭제한 것이 원인이 되어 , 
 JPA 벌크 연산을 이용하여 문제를 해결하였습니다.

 <details>
<summary><b>해결 코드</b></summary>
<div markdown="1">
  
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
</br>

## 6. 그 외 트러블 슈팅
<summary>생성자 자동 주입시 순환 참조</summary>
<details>
<div markdown="1">
-원인 : Service 계층들끼리 서로 참조하고 있어 문제가 발생. 
-해결 : 단방향 참조로 변경 . Service -> Repository 계층만 참조하도록 전체 구조 변경.
</div>
</details>

<summary>파일 다운로드</summary>
<details>
<div markdown="1">
-원인 : CONTENT_DISPOSITION 헤더의 부재.
-해결 : ResponseEntity를 사용하여 응답에  contentDisposition = "attachment; filename 추가하여 해결
</div>
</details>


</br>

