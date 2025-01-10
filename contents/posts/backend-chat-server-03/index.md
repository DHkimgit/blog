---
title: "[KOIN] 채팅 데이터 저장 전략 (2) - 관계형 DB, 몽고디비 저장 구현과 빈 동적 주입"
description: "벡엔드"
date: 2025-01-10
update: 2025-01-10
tags:
  - java
  - spring
series: "Chatting Server"
---

처음 구상했던 채팅 메시지 저장 방법은 레디스에 우선 저장한 뒤 시간이 지난 데이터를 이관하는 것이었다. 이 과정에서 여러 데이터베이스의 특성을 활용하고 시스템의 유연성을 확보하기 위해 멀티 모듈 아키텍처를 구현했다. 특히 저장소 계층을 추상화하여 다양한 데이터베이스를 동적으로 전환할 수 있도록 설계했다.

## 관계형 데이터베이스(Postgres)에 저장 구현

테이블은 다음과 같이 작성했다.

```sql
CREATE TABLE chat_message ( 
	id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY, 
	message_id BIGINT NOT NULL, 
	article_id BIGINT NOT NULL, 
	chat_room_id BIGINT NOT NULL, 
	user_id BIGINT NOT NULL, 
	is_read BOOLEAN NOT NULL DEFAULT FALSE, 
	is_deleted BOOLEAN NOT NULL DEFAULT FALSE, 
	nick_name VARCHAR(50) NOT NULL, 
	contents TEXT NOT NULL, 
	created_at TIMESTAMP NOT NULL
);
```

임시 테스트용 테이블이므로 정규화는 따로 고려하지 않았다.

`message_id` 값엔 벡엔드에서 생성한 tsid 값이 삽입될 것이다.

`article_id` 와 `chat_room_id` 값으로 해당 채팅방의 전체 메시지를 조회하기 때문에 인덱스를 추가했다.

```sql
CREATE INDEX idx_article_chatroom_time 
ON chat_message (article_id, chat_room_id, created_at);
```

`created_at` 을 기준으로 정렬해야 하므로 다중 인덱스를 생성한다.

`module-rdb` 를 생성하고 의존성을 설정한다. 이 모듈은 관계형 데이터베이스와 상호작용허눈 데이터 영속성을 담당하는 독립적인 모듈이다. 

도메인 모듈에만 의존성을 가지도록 설계하여 다른 저장소 모듈들과의 결합도를 낮추었다. 따라서 다른 데이터베이스를 사용하는 모듈을 추가하거나 기존 모듈을 수정하더라도 서로 영향을 주지 않는다.

```gradle
bootJar {
    enabled = false
}

jar {
    enabled = true
}

dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web')
    implementation('org.springframework.boot:spring-boot-starter-data-jpa')
    implementation("org.postgresql:postgresql")

    compileOnly(project(":module-domain"))
}
```

JPA 사용을 위한 라이브러리와 postgres 드라이버를 활용한다,

```java
@Entity
@Builder
@AllArgsConstructor
@NoArgsConstructor(access = PROTECTED)
@Table(name = "chat_message")
public class ChatMessageRdbEntity {

    @Id
    @GeneratedValue(strategy = IDENTITY)
    private Long id;
    private Long messageId;
    private Long articleId;
    private Long chatRoomId;
    private Long userId;
    private Boolean isRead;
    private Boolean isDeleted;
    private String nickName;
    private String contents;
    private LocalDateTime createdAt;

    // 도메인 객체 -> 영속성 엔티티
    public static ChatMessageRdbEntity toRdbEntity(ChatMessageEntity message) {...}
    
    // 영속성 엔티티 -> 도메인 객체
    public ChatMessageEntity toDomain() {...}
}

@Repository("rdb")
@RequiredArgsConstructor
public class ChatMessageRepositoryRdbImpl implements ChatMessageRepository {

    private final ChatMessageJpaRepository chatMessageJpaRepository;

    @Override
    public ChatMessageEntity save(ChatMessageEntity message) {
        ChatMessageRdbEntity messageEntity = ChatMessageRdbEntity.toRdbEntity(message);
        chatMessageJpaRepository.save(messageEntity);
        return message;
    }

    // TODO : ChatMessageRepositoryRdbImpl findByArticleIdAndChatRoomId limit 로직 구현
    @Override
    public List<ChatMessageEntity> findByArticleIdAndChatRoomId(Long articleId, Long chatRoomId, int limit) {
        return chatMessageJpaRepository.findAllByArticleIdAndAndChatRoomId(articleId, chatRoomId).stream()
            .map(ChatMessageRdbEntity::toDomain)
            .toList();
    }
}

public interface ChatMessageJpaRepository extends Repository<ChatMessageRdbEntity, Long> {

    ChatMessageRdbEntity save(ChatMessageRdbEntity entity);

    @Query("SELECT m FROM ChatMessageRdbEntity m WHERE m.articleId= :articleId AND m.chatRoomId= :chatRoomId ORDER BY m.createdAt")
    List<ChatMessageRdbEntity> findAllByArticleIdAndAndChatRoomId(Long articleId, Long chatRoomId);
}
```

메시지 저장과 조회를 위한 간단한 코드를 작성한다. 엔티티를 도메인 객체로 바꿔서 반환해야 한다.

## 몽고디비 저장 구현

몽고디비는 살짝 더 까다로웠다. 스프링 데이터 JPA를 사용하고 싶었는데 엔티티와 매핑이 잘 되지 않는 문제가 생겼다. `spring-data-mongodb` 를 깊게 공부해보지 않아서 생긴 문제라 추후에 좀 더 자세히 알아보려고 한다.

또한 스프링 데이터 JPA는 UPDATE를 지원하지 않기 때문에 MongoTemplate을 이용하여 리포지토리를 직접 구현했다.

```gradle
bootJar {
    enabled = false
}

jar {
    enabled = true
}

dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web')
    implementation('org.springframework.boot:spring-boot-starter-data-jpa')
    implementation('org.springframework.boot:spring-boot-starter-data-mongodb')

    compileOnly(project(":module-domain"))
}
```

```java
@Getter
@Document(collection = "chat_room")
public class ChatMessageMongoEntity {

    @Id
    @Field("_id")
    private ObjectId id;

    @Field("article_id")
    private Long articleId;

    @Field("chatroom_id")
    private Long chatRoomId;

    @Field("messages")
    private List<Message> messageList;

    @Builder
    public ChatMessageMongoEntity(Long articleId, Long chatRoomId) {
        this.articleId = articleId;
        this.chatRoomId = chatRoomId;
        this.messageList = new ArrayList<>();
    }

    public void addMessage(Message message) {
        this.messageList.add(message);
    }

    @Getter
    @Builder
    @AllArgsConstructor
    @NoArgsConstructor(access = AccessLevel.PROTECTED)
    public static class Message {

        @Field("tsid")
        private String tsid;

        @Field("user_id")
        private Long userId;

        @Field("is_read")
        private Boolean isRead;

        @Field("is_deleted")
        private Boolean isDeleted;

        @Field("nickname")
        private String nickName;

        @Field("contents")
        private String contents;

        @Field("created_at")
        private LocalDateTime createdAt;

        public static Message fromDomain(ChatMessageEntity domainMessage) {...}

    public static ChatMessageMongoEntity createNewDocument(Long articleId, Long chatRoomId) {...}
}
```

RDB의 엔티티와 마찬가지로 몽고디비도 영속성 객체를 정의해주어야 한다. @Document 어노테이션을 통해 몽고디비 컬렉션과 매핑이 가능하다.

```java
@Repository("mongoRepository")
@RequiredArgsConstructor
public class ChatMessageRepositoryMongoImpl implements ChatMessageRepository {

    private final MongoTemplate mongoTemplate;

    @Override
    public ChatMessageEntity save(ChatMessageEntity message) {
        // articleId와 chatRoomId로 기존 문서 조회
        Query query = Query.query(Criteria.where("article_id").is(message.getArticleId())
            .and("chatroom_id").is(message.getChatRoomId()));
        Update update = new Update();

        // 새로운 메시지를 ChatMessageMongoEntity.Message로 변환
        ChatMessageMongoEntity.Message newMessage = ChatMessageMongoEntity.Message.fromDomain(message);

        // messages 필드에 새 메시지를 추가
        update.push("messages", newMessage);

        // 기존 문서가 없으면 새로 생성
        update.setOnInsert("article_id", message.getArticleId());
        update.setOnInsert("chatroom_id", message.getChatRoomId());

        // upsert(있으면 업데이트, 없으면 삽입)
        mongoTemplate.upsert(query, update, ChatMessageMongoEntity.class);

        return message;
    }
```

스프링 데이터 JPA의 인터페이스를 사용할 수 없다면 저장하는 로직을 직접 구현해야 한다.

Query 객체와 Update 객체, MongoTemplate을 이용했다. 몽고디비 저장 로직을 fastapi 개발할 때 사용해존 적이 있었는데 그것과 약간 비슷한 것 같다.

## repository 빈 동적 주입
```java
@Component
@RequiredArgsConstructor
public class MessageAppender {

    private final Map<String, ChatMessageRepository> repositoryMap;

    public void append(ChatMessageEntity message, String storageType) {
        ChatMessageRepository chatMessageRepository = repositoryMap.get(storageType);
        chatMessageRepository.save(message);
    }
}
```

도메인 모듈의 `ChatMessageRepository` 에 대한 구현체는 각각의 스토리지 모듈에서 빈으로 등록된다.

- **module-storage-mongo** : ChatMessageRepositoryMongoImpl
- **module-storage-rdb** : ChatMessageRepositoryRdbImpl
- **module-storage-redis** : ChatMessageRepositoryRedisImpl

이 구조를 통해 쉽게 새로운 저장소 추가가 가능했다. 새로운 모듈은 단순히 `ChatMessageRepository`만 구현하면 된다. 또한 각 저장소 구현체를 독립적으로 테스트 할 수 있다.

이 빈들을 동적으로 주입받아 다형성을 구현하기 위해 Map 자료구조를 사용했다. 또한 각각의 빈은 고유 식별자를 가지기 때문에 원하는 리포지토리에 동적으로 연결해서 사용할 수 있다.

```java
public void saveMessage(Long articleId, Long chatRoomId, ChatMessageCommand message) {
    ChatMessageEntity newMessage = ChatMessageEntity.create(articleId, chatRoomId, message);

    messageHelper.generateMessageId(newMessage);

    messageAppender.append(newMessage, "redis");
    messageAppender.append(newMessage, "rdb");
    messageAppender.append(newMessage, "mongoRepository");
}
```


