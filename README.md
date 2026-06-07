# リアルタイムチャット機能 統合実装リファレンス【完全版】

## 本ドキュメントの目的
本ドキュメントは、AWS環境（ECS + RDS）での稼働を前提とした、単一コンテナ構成のリアルタイムチャットシステムの実装リファレンスです。
研修課題やシステム開発における「セッション認証」「セキュリティ対策（XSS防止）」「インフラ制約（ALBタイムアウト防止）」を網羅したベストプラクティスを提供します。

## 1. 導入技術の概要
本システムは、JSP/Servletをベースとしつつ、モダンなWebアプリケーションに不可欠な以下の技術・インフラ要素を組み合わせて構築されています。

| 技術・インフラ要素 | 概要と本システムでの役割 |
| :--- | :--- |
| **WebSocket** | ブラウザとサーバー間で「常時接続の双方向通信」を実現するプロトコル。相手の発言をリロードなしで画面に即時反映（プッシュ通知）させます。 |
| **JSON (Gson)** | サーバー（Java）のオブジェクトを文字列（JSON）に変換し、WebSocket経由でクライアント（JavaScript）へ渡すために、GoogleのGsonライブラリを使用します。 |
| **PostgreSQL (RDS)** | フルマネージドなリレーショナルデータベース。外部キー制約（`ON DELETE CASCADE`）とインデックスを駆使し、データの整合性担保と高速な過去ログ検索を実現します。 |
| **AWS ALB** | サーバーへのアクセスを振り分けるロードバランサー。「60秒間無通信が続くと強制切断する」仕様に対する対策（定期Ping通信）を実装しています。 |
| **AWS ECS** | アプリケーションをコンテナ（Docker等）として実行・管理するサービス。本システムでは構成をシンプルにするため、コンテナ数を「1台（スケールアウトなし）」で稼働させます。 |

---

## 2. システムアーキテクチャ設計（セキュア・ハイブリッド構成）
既存のログイン認証（セッション情報）を活かし、悪意のあるユーザーによる「なりすまし」を完全に防ぐため、**「送信はHTTP」「受信はWebSocket」のハイブリッド構成**を採用しています。

### 通信とセキュリティの処理フロー
1. **【送信】** クライアント（JS）から `ChatServlet` へ、メッセージ本文のみを **HTTP POST** で送信します。
2. **【認証】** `ChatServlet` はリクエストの偽造を防ぐため、サーバー側のセッションから確実なユーザーIDを取得します（なりすまし防止）。
3. **【保存】** 取得したユーザーIDとメッセージ本文をデータベースへ保存します。
4. **【連携】** `ChatServlet` から WebSocket管理クラス (`ChatEndpoint`) へ配信を指示します。
5. **【配信】** `ChatEndpoint` がデータをJSON形式に変換し、全クライアントへ **WebSocketで一斉配信** します。
6. **【更新】** クライアント側でJSONを受け取り、XSS（クロスサイトスクリプティング）を無害化する処理を挟んだ上で、画面を動的に更新します。

---

## 3. 開発ディレクトリ構造
必要となるライブラリ（JAR）の配置箇所を含めたプロジェクト構成です。

```text
[プロジェクト名]/
├── src/main/java/
│   ├── controller/                
│   │   ├── ChatServlet.java       
│   │   └── ChatEndpoint.java      
│   ├── model/                     
│   │   └── ChatService.java       
│   ├── dao/                       
│   │   └── MessageDAO.java        
│   ├── dto/                       
│   │   └── MessageDTO.java        
│   └── util/                      
│       └── DBUtil.java            
│
└── src/main/webapp/
    └── WEB-INF/
        ├── chat.jsp               
        └── lib/                   
            ├── postgresql-x.x.x.jar 
            └── gson-x.x.x.jar       
```

---

## 4. データベース定義（PostgreSQL）
ユーザーアカウント管理テーブル（`account`）が既に存在することを前提としています。

```sql
CREATE TABLE chat_detail (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id INT NOT NULL,
    chat_body TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- 外部キー制約（親アカウント削除時に連動して履歴も削除）
    FOREIGN KEY (user_id) 
        REFERENCES account(id) 
        ON DELETE CASCADE 
        ON UPDATE CASCADE
);

-- 連動削除のパフォーマンス低下を防ぎ、検索を高速化するためのインデックス
CREATE INDEX idx_chat_detail_user_id ON chat_detail(user_id);
```

---

## 5. 全ソースコード一式および詳細解説

### ① 【DTO】 dto/MessageDTO.java

```java
package dto;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class MessageDTO {
    private Integer userId; 
    private String chatBody;
    private String timestamp;

    public MessageDTO() {}

    public MessageDTO(Integer userId, String chatBody) {
        this.userId = userId;
        this.chatBody = chatBody;
        this.timestamp = LocalDateTime.now().format(DateTimeFormatter.ofPattern("HH:mm"));
    }

    public MessageDTO(Integer userId, String chatBody, String timestamp) {
        this.userId = userId;
        this.chatBody = chatBody;
        this.timestamp = timestamp;
    }

    public Integer getUserId() { return userId; }
    public String getChatBody() { return chatBody; }
    public String getTimestamp() { return timestamp; }
}
```

**【詳細解説】**
* **役割:** データの運搬（Data Transfer Object）および、GsonライブラリによるJSON変換のベースとなるクラスです。このクラスのフィールド名が、そのままJavaScript側で受け取るJSONのキー名になります。
* **日時のフォーマット処理:** チャット画面では「14:30」のような時刻のみの表示が一般的なため、コンストラクタ内で `LocalDateTime` を用いて、インスタンス生成時に自動で `HH:mm` 形式の文字列としてタイムスタンプを保持する設計にしています。

---

### ② 【Util】 util/DBUtil.java

```java
package util;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class DBUtil {
    private static final String URL = "jdbc:postgresql://localhost:5432/ご自身のDB名";
    private static final String USER = "postgres";
    private static final String PASS = "パスワード";

    static {
        try {
            Class.forName("org.postgresql.Driver");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }

    public static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USER, PASS);
    }
}
```

**【詳細解説】**
* **役割:** データベースへの接続情報と、コネクション取得の共通処理を隠蔽するユーティリティクラスです。
* **staticイニシャライザの活用:** `Class.forName("org.postgresql.Driver")` は、クラスがメモリにロードされた最初の1回だけ実行されればよいため、`static { ... }` ブロック内に記述しています。これにより、毎回接続するたびにドライバをロードする無駄を省いています。

---

### ③ 【DAO】 dao/MessageDAO.java

```java
package dao;

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.List;

import dto.MessageDTO;
import util.DBUtil;

public class MessageDAO {

    public void insert(MessageDTO msg) {
        String sql = "INSERT INTO chat_detail (user_id, chat_body) VALUES (?, ?)";
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            
            pstmt.setInt(1, msg.getUserId());
            pstmt.setString(2, msg.getChatBody());
            pstmt.executeUpdate();
            
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    public List<MessageDTO> getLatest50() {
        List<MessageDTO> list = new ArrayList<>();
        String sql = "SELECT user_id, chat_body, created_at FROM ("
                   + "  SELECT user_id, chat_body, created_at FROM chat_detail ORDER BY id DESC LIMIT 50"
                   + ") AS sub ORDER BY created_at ASC";
        
        try (Connection conn = DBUtil.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql);
             ResultSet rs = pstmt.executeQuery()) {
             
            while (rs.next()) {
                Integer userId = rs.getInt("user_id");
                String chatBody = rs.getString("chat_body");
                LocalDateTime createdAt = rs.getTimestamp("created_at").toLocalDateTime();
                String timestamp = createdAt.format(DateTimeFormatter.ofPattern("HH:mm"));
                
                list.add(new MessageDTO(userId, chatBody, timestamp));
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return list;
    }
}
```

**【詳細解説】**
* **役割:** データベースに対する具体的なSQLの実行（登録・抽出）を担当します。
* **サブクエリを用いたソートテクニック:** `getLatest50()` メソッドのSQLに注目してください。単純に `ORDER BY id DESC LIMIT 50` とすると、最新の50件は取得できますが「新しい発言が上」に来てしまいます。チャットは「新しい発言が下」に来る必要があるため、一度サブクエリで最新50件を切り出した後、外側のクエリで `ORDER BY created_at ASC`（古い順）に並び直す処理を行っています。
* **Try-with-resources文:** `try ( ... )` の構文を使用することで、処理完了後や例外発生時に `Connection` や `PreparedStatement` などのリソースが自動的にクローズ（解放）され、メモリリークやコネクション枯渇を防いでいます。

---

### ④ 【Model】 model/ChatService.java

```java
package model;

import java.util.List;
import dao.MessageDAO;
import dto.MessageDTO;

public class ChatService {
    private static final ChatService instance = new ChatService();
    private final MessageDAO messageDAO = new MessageDAO();

    private ChatService() {}

    public static ChatService getInstance() {
        return instance;
    }

    public void saveMessage(MessageDTO msg) {
        messageDAO.insert(msg);
    }

    public List<MessageDTO> getLogList() {
        return messageDAO.getLatest50();
    }
}
```

**【詳細解説】**
* **役割:** Controller（Servlet/Endpoint）とDAOの間に立つビジネスロジック層です。今回は機能がシンプルですが、将来的に「NGワードのチェック機能」などを追加する場合は、このクラスにロジックを実装します。
* **シングルトンパターンの採用:** 状態を持たないロジッククラスであるため、リクエストのたびに `new` してメモリを消費しないよう、システム内でインスタンスが1つだけ生成される「シングルトンパターン」を採用しています（`getInstance()` メソッドによる呼び出し）。

---

### ⑤ 【Controller: HTTP窓口】 controller/ChatServlet.java

```java
package controller;

import java.io.IOException;

import dto.MessageDTO;
import model.ChatService;

import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import jakarta.servlet.http.HttpSession;

@WebServlet("/ChatServlet")
public class ChatServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) 
            throws ServletException, IOException {
        
        HttpSession session = request.getSession(false);
        if (session == null || session.getAttribute("user") == null) {
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "ログインが必要です");
            return;
        }
        
        request.getRequestDispatcher("/WEB-INF/chat.jsp").forward(request, response);
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) 
            throws ServletException, IOException {
        
        request.setCharacterEncoding("UTF-8");

        HttpSession session = request.getSession(false);
        if (session == null || session.getAttribute("user") == null) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED); 
            return;
        }

        Integer userId = null;
        try {
            userId = Integer.parseInt(String.valueOf(session.getAttribute("user")));
        } catch (NumberFormatException e) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return;
        }

        String chatBody = request.getParameter("chatBody");

        // 空文字および長すぎる文字（スパム・メモリ枯渇対策）のブロック
        if (chatBody == null || chatBody.trim().isEmpty() || chatBody.length() > 500) {
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            return;
        }

        MessageDTO msgDto = new MessageDTO(userId, chatBody.trim());
        ChatService.getInstance().saveMessage(msgDto);

        ChatEndpoint.broadcast(msgDto);

        response.setStatus(HttpServletResponse.SC_OK);
    }
}
```

**【詳細解説】**
* **役割:** 画面の表示要求（GET）と、メッセージの送信要求（POST）を受け付けるHTTP窓口です。
* **サーバー側セッションによる「なりすまし防止」:** POST処理において、クライアントから送信元ユーザーIDを受け取っていません。クライアントからのPOSTデータは悪意あるユーザーによって改ざんされる可能性があるため、**必ずサーバー側で管理している確実なセッション情報（`session.getAttribute("user")`）**から送信元を特定し、なりすましを根本から防いでいます。
* **バリデーション（防御的プログラミング）:** `chatBody.length() > 500` のように文字数制限を設けることで、大量のテキストを送りつけてサーバーのメモリを枯渇させる攻撃や、スパムを防止しています。
* **ハイブリッド構成のブリッジ:** DBへの保存完了後、WebSocketを管理する `ChatEndpoint.broadcast(msgDto)` を呼び出し、HTTPで受け取ったデータをWebSocket側へ受け渡す橋渡しを行っています。

---

### ⑥ 【Controller: WebSocket配信】 controller/ChatEndpoint.java

```java
package controller;

import java.io.IOException;
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;

import com.google.gson.Gson;

import dto.MessageDTO;
import model.ChatService;

import jakarta.websocket.OnClose;
import jakarta.websocket.OnError;
import jakarta.websocket.OnMessage;
import jakarta.websocket.OnOpen;
import jakarta.websocket.Session;
import jakarta.websocket.server.ServerEndpoint;

@ServerEndpoint("/chatEndpoint")
public class ChatEndpoint {

    private static final Set<Session> clients = Collections.synchronizedSet(new HashSet<>());
    private static final Gson gson = new Gson();

    @OnOpen
    public void onOpen(Session session) {
        clients.add(session);
        try {
            for (MessageDTO pastMsg : ChatService.getInstance().getLogList()) {
                session.getBasicRemote().sendText(gson.toJson(pastMsg));
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @OnMessage
    public void onMessage(String message, Session session) {
        if ("ping".equals(message)) {
            // 受信するだけでコネクションが維持されるため、特別な処理は不要
        }
    }

    @OnClose
    public void onClose(Session session) {
        clients.remove(session);
    }

    @OnError
    public void onError(Throwable throwable) {
        throwable.printStackTrace();
    }

    public static void broadcast(MessageDTO msgDto) {
        String jsonResult = gson.toJson(msgDto);
        
        synchronized (clients) {
            for (Session client : clients) {
                if (client.isOpen()) {
                    try {
                        client.getBasicRemote().sendText(jsonResult);
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}
```

**【詳細解説】**
* **役割:** WebSocketプロトコルによるクライアントとの常時接続管理と、プッシュ通知（一斉配信）を担当します。
* **スレッドセーフなコレクション管理:** 複数のユーザーが同時に接続・切断を行うため、接続状態を保持する `clients` リストには `Collections.synchronizedSet` を使用し、排他制御（スレッドセーフ）を担保してデータ不整合によるクラッシュを防いでいます。
* **初期データ（過去ログ）の送信:** `@OnOpen` アノテーションが付与されたメソッドは、クライアントが接続した瞬間に実行されます。ここでDAOから過去ログを取得し、接続してきたユーザーの画面に初期データを送り込んでいます。
* **AWS ALBのタイムアウト回避:** `@OnMessage` にて、クライアントから送られてくる "ping" という文字列を受け流す処理を実装しています。AWSのALB（ロードバランサー）は仕様上「60秒間無通信状態のWebSocket接続を強制切断」するため、サーバー側で定期的な通信を受信して接続をアクティブな状態に維持しています。

---

### ⑦ 【View】 src/main/webapp/WEB-INF/chat.jsp

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>システム統合用 ハイブリッドチャット</title>
<style>
    * { box-sizing: border-box; }
    body { font-family: sans-serif; background-color: #eef2f5; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; }
    .chat-wrapper { width: 100%; max-width: 500px; height: 600px; background: white; border-radius: 12px; display: flex; flex-direction: column; overflow: hidden; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
    .chat-header { background: #075e54; color: white; padding: 15px; text-align: center; font-weight: bold; }
    #chatArea { flex-grow: 1; padding: 20px; background: #efeae2; overflow-y: auto; }
    .msg-block { margin-bottom: 12px; clear: both; }
    .msg-bubble { background: white; padding: 10px; border-radius: 8px; display: inline-block; max-width: 85%; word-break: break-all; }
    .msg-user { font-size: 0.75rem; color: #128c7e; font-weight: bold; margin-bottom: 3px; }
    .msg-text { font-size: 0.95rem; color: #333; }
    .msg-time { font-size: 0.65rem; color: #999; text-align: right; margin-top: 4px; display: block; }
    .my-message { text-align: right; }
    .my-message .msg-bubble { background: #d9fdd3; text-align: left; }
    .chat-footer { padding: 15px; background: #f0f2f5; display: flex; gap: 10px; }
    input { padding: 10px; border: 1px solid #ccc; border-radius: 6px; }
    #chatBodyInput { flex-grow: 1; }
    button { padding: 10px 20px; background: #075e54; color: white; border: none; border-radius: 6px; cursor: pointer; font-weight: bold; }
</style>
</head>
<body>

<div class="chat-wrapper">
    <div class="chat-header">ハイブリッドチャット</div>
    <div id="chatArea"></div>
    <div class="chat-footer">
        <input type="text" id="chatBodyInput" placeholder="メッセージを入力..." autocomplete="off">
        <button onclick="sendMessage()">送信</button>
    </div>
</div>

<script>
    // 【要変更】コンテキストルートに合わせてプロジェクト名を指定してください
    const CONTEXT_PATH = "/YourProjectName"; 
    
    const wsUrl = "ws://" + window.location.host + CONTEXT_PATH + "/chatEndpoint";
    const socket = new WebSocket(wsUrl);

    // JSP経由でセッションから自身のIDを取得
    const myUserId = parseInt("<%= session.getAttribute("user") %>", 10);

    socket.onopen = function() {
        // ALBの仕様（60秒の無通信で切断）を回避するため、50秒ごとに死活監視用データを送信する
        setInterval(() => {
            if (socket.readyState === WebSocket.OPEN) {
                socket.send("ping");
            }
        }, 50000); 
    };

    socket.onmessage = function(event) {
        const chatArea = document.getElementById("chatArea");
        const msg = JSON.parse(event.data); 

        const blockDiv = document.createElement("div");
        blockDiv.className = "msg-block" + (msg.userId === myUserId ? " my-message" : "");

        const bubbleDiv = document.createElement("div");
        bubbleDiv.className = "msg-bubble";

        const userDiv = document.createElement("div");
        userDiv.className = "msg-user";
        userDiv.textContent = "User:" + msg.userId;

        const textDiv = document.createElement("div");
        textDiv.className = "msg-text";
        textDiv.textContent = msg.chatBody;

        const timeSpan = document.createElement("span");
        timeSpan.className = "msg-time";
        timeSpan.textContent = msg.timestamp;

        bubbleDiv.appendChild(userDiv);
        bubbleDiv.appendChild(textDiv);
        bubbleDiv.appendChild(timeSpan);
        
        blockDiv.appendChild(bubbleDiv);
        chatArea.appendChild(blockDiv);
        
        // 常に最新メッセージが表示されるよう最下部にスクロール
        chatArea.scrollTop = chatArea.scrollHeight;
    };

    function sendMessage() {
        const msgInput = document.getElementById("chatBodyInput");
        const chatBody = msgInput.value.trim();
        
        if (chatBody === "") return;

        const params = new URLSearchParams();
        params.append("chatBody", chatBody);

        fetch(CONTEXT_PATH + "/ChatServlet", {
            method: "POST",
            headers: {
                "Content-Type": "application/x-www-form-urlencoded"
            },
            body: params
        }).then(response => {
            if(response.ok) {
                msgInput.value = "";
                msgInput.focus();
            } else if(response.status === 400) {
                alert("文字数が制限を超過しているか、不正な値です。");
            } else if(response.status === 401) {
                alert("セッションが切れました。再度ログインしてください。");
                window.location.reload();
            } else {
                console.error("送信に失敗しました");
            }
        });
    }

    // Enterキーでの送信対応
    document.getElementById("chatBodyInput").addEventListener("keypress", function(e) {
        if (e.key === "Enter") sendMessage();
    });
</script>
</body>
</html>
```
