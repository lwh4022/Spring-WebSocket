# Web socket + Spring boot 사용법

## 서버

### 1. build.gradle에 Dependency 추가
```
implementation 'org.springframework.boot:spring-boot-starter-websocket'
```

### 2. MessageBroker 활성화
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer{

	@Override
	public void configureMessageBroker(MessageBrokerRegistry registry) {
    // topic으로 시작되는 메시지가 메시지 브로커로 라우팅되도록 설정
		registry.enableSimpleBroker("/topic");
    //  /app으로 시작되는 메시지가 메시지 핸들 메소드로 연결
		registry.setApplicationDestinationPrefixes("/app");
	}
	
	@Override
	public void registerStompEndpoints(StompEndpointRegistry registry) {
  
    // 웹소켓 엔드포인트 명시
    // withSockJS()는 웹소켓을 사용할 수 없는 브라우저나 장애가 있을 때 fallback 옵션을 사용할 수 있음
		registry.addEndpoint("/ws").withSockJS();
	}
}
```

### 3. WebSocketEventListener 구현 (@Component 사용)
```java
  @Autowired
  private SimpMessageSendingOperations messagingTemplate;

  // 연결 성공시
  @EventListener
  public void handleWebSocketConnectListener(SessionConnectedEvent event) {
 	logger.info("Received a new web socket connection");
  }
	
  // 연결 실패시
  @EventListener
  public void handleWebSocketDisconnectListner(SessionDisconnectEvent event) {
  	StompHeaderAccessor headerAccessor = StompHeaderAccessor.wrap(event.getMessage());
	logger.info(new String(event.getMessage().getHeaders().toString()));
		
	String username = (String) headerAccessor.getSessionAttributes().get("username");
	if(username != null) {
		logger.info("User Disconnected : " + username);
		
		ChatMessage chatMessage = new ChatMessage();
		chatMessage.setType(MessageType.LEAVE);
		chatMessage.setSender(username);
    
     //메시지 전송 하기
		messagingTemplate.convertAndSend("/topic/public", chatMessage);
		}
  }
```

### 4. Controller 구현
```java
  //Message-handling Method 엔드포인트
  @MessageMapping("/chat.sendMessage")
  // Queue 이름
  @SendTo("/topic/public")
  public ChatMessage greeting(@Payload ChatMessage chatMessage) throws Exception {
	return chatMessage;
  }
	
  @MessageMapping("/chat.addUser")
  @SendTo("/topic/public")
  public ChatMessage addUser(@Payload ChatMessage chatMessage, SimpMessageHeaderAccessor headerAccessor) {
	headerAccessor.getSessionAttributes().put("username", chatMessage.getSender());
	return chatMessage;
  }
```

### 클라이언트
1. Connect
```javascript
function connect(event) {
    username = document.querySelector('#name').value.trim();

    if(username) {
        usernamePage.classList.add('hidden');
        chatPage.classList.remove('hidden');
        
        // 소켓 연결
        var socket = new SockJS('/ws');
        
        // STOMP에 소켓 연결
        stompClient = Stomp.over(socket);
        
        // 서버에 연결
        stompClient.connect({}, onConnected, onError);
        
        // 콘솔 창에 로그 미표시, 기본은 표시
        stompClient.debug = function(str){};
        
        // 서버에서 소켓이 끊어졌을 시
        stompClient.ws.onclose = function(e){
        	console.log(e);
        	onError('error');
        }
    }
    event.preventDefault();
}
```

2. 채널 연결
```javascript
function onConnected() {
    // 소켓 큐에 연결
    stompClient.subscribe('/topic/public', onMessageReceived);

    // 메시지 보내기
    // send(엔드포인트, 헤더, 바디)
    stompClient.send("/app/chat.addUser",
        {},
        JSON.stringify({sender: username, type: 'JOIN'})
    )

    connectingElement.classList.add('hidden');
}
```

3. 메시지 보내기
```javascript
function sendMessage(event) {
    var messageContent = messageInput.value.trim();
    if(messageContent && stompClient) {
        var chatMessage = {
            sender: username,
            content: messageInput.value,
            type: 'CHAT'
        };
        // send(엔드포인트, 헤더, 바디)
        stompClient.send("/app/chat.sendMessage", {}, JSON.stringify(chatMessage));
        messageInput.value = '';
    }
    event.preventDefault();
}
```

