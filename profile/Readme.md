### Chat App (Didn't think of a name yet)

-   [System Design Overview](#system-design-overview)
    -   [Technology Stack](#technology-stack)
    -   [System Components \& Interaction](#system-components--interaction)
        -   [1. WebSocket (Socket.IO)](#1-websocket-socketio)
        -   [2. JWT (JSON Web Token)](#2-jwt-json-web-token)
        -   [3. SQL (PostgreSQL)](#3-sql-postgresql)
        -   [4. SQLite (for Client-side Storage)](#4-sqlite-for-client-side-storage)
    -   [System Flow](#system-flow)
        -   [1. User Login \& Authentication](#1-user-login--authentication)
        -   [2. Real-Time Chat (with WebSocket)](#2-real-time-chat-with-websocket)
        -   [3. Message Storage (SQLite on Client Side)](#3-message-storage-sqlite-on-client-side)
        -   [4. File Upload and Sharing](#4-file-upload-and-sharing)
        -   [5. Data Flow](#5-data-flow)
    -   [Database Design](#database-design)
    -   [Microservices Breakdown](#microservices-breakdown)
    -   [Final Architecture](#final-architecture)
    -   [Deployment \& Scaling Considerations](#deployment--scaling-considerations)

## System Design Overview

### Technology Stack

1. _WebSocket (Socket.IO):_ Used for real-time communication, ensuring that the chat messages are delivered instantly.

2. _JWT (JSON Web Token):_ For user authentication and secure communication between services. It will help with both user login and maintaining authenticated sessions.

3. _SQL (PostgreSQL):_ For storing user profiles, authentication details, and user metadata (such as the permanent username).

4. _SQLite:_ For message storage on the client-side, providing a local database to persist chat history during a session. This is especially useful for offline functionality.

### System Components & Interaction

#### 1. WebSocket (Socket.IO)

-   _Role_: Handles real-time message exchange between the client and server.

-   _Operation_:
    Once the user logs in (authenticated) or connects as a guest, the client will establish a WebSocket connection with the Chat Service.
    The server will validate the JWT token sent from the client, or for guest users, a temporary session ID will be used.
    Chat messages are sent via WebSocket events (e.g., sendMessage, receiveMessage).
    For guest users, random user matching can be done by the server, ensuring that they are connected with other random users.
    Authenticated users will have the ability to send private messages and access their chat history.

#### 2. JWT (JSON Web Token)

-   _Role_: Ensures secure authentication and authorization of users.

-   _Operation_:
    The user logs in via email/password or OAuth (Google, GitHub).
    If successful, a JWT token is generated containing the user’s unique identifier (like the userId).
    The client stores this JWT in the local storage (or cookies) and includes it in the WebSocket handshake when initiating the connection to the Chat Service.
    The Chat Service validates the JWT on every connection attempt to ensure the user is authenticated.
    This also helps protect endpoints related to file sharing and user profile management (so only authenticated users can upload files or manage their profiles).

#### 3. SQL (PostgreSQL)

-   _Role_: Storing user profile and authentication information (username, email, password, etc.).

-   _Operation_:
    When a user registers or logs in, the system will check the SQL database for existing credentials (for authentication).
    Store the user's profile information such as their permanent username, profile settings, and other metadata.
    SQL will be used for user-related operations like registration, login, and password management.
    Data like chat preferences and notification settings can also be stored here.

#### 4. SQLite (for Client-side Storage)

-   _Role_: Local storage of chat messages during a session, even when the user is offline.

-   _Operation_:
    The SQLite database on the client-side will store chat messages, which will be synced with the server once the connection is re-established.
    This helps provide a smooth offline experience, where the client can view past chat messages without requiring an active connection.
    The SQLite storage will only handle chat messages for the current session, meaning historical messages from previous sessions won’t be stored on the client.

### System Flow

#### 1. User Login & Authentication

-   User logs in via email/password or OAuth (Google, GitHub).
-   Auth Service validates the credentials.
-   On successful authentication, a JWT token is generated and returned to the client.
-   The client stores the JWT token and includes it in the WebSocket connection request to authenticate and establish a connection.

#### 2. Real-Time Chat (with WebSocket)

-   The client connects to the Chat Service via WebSocket using the JWT token for authentication (authenticated users) or temporary session ID for guest users.
-   The client sends and receives messages using WebSocket events (sendMessage, receiveMessage).
-   The Chat Service sends the message to the appropriate destination (random guest matching for guests, specific user for private messages).
-   Authenticated users can join private rooms or chat with specific users, while guest users can only chat with other random guests.

#### 3. Message Storage (SQLite on Client Side)

-   Messages exchanged during a session are stored in SQLite on the client side.
-   When the user is offline, chat messages are saved locally.
-   On reconnecting, the client can sync these locally stored messages with the Chat Service or display them as needed.
-   SQLite storage ensures that users can access their messages even if they temporarily lose their network connection.

#### 4. File Upload and Sharing

-   Authenticated users can upload files through the File Service.
-   The File Service stores files securely (AWS S3, GridFS, or another cloud storage solution).
-   The file’s URL or access link is shared via WebSocket in the chat.
-   Only authenticated users can upload and share files, ensuring that guests are restricted from doing so.

#### 5. Data Flow

-   **Authentication:**
    User provides credentials → Auth Service → Validate credentials → Generate JWT token → Return token to client.
-   **Chat:**
    Client connects via WebSocket → Auth Service validates JWT token (if authenticated) → Chat Service establishes WebSocket connection → User sends/receives messages in real-time.
-   **File Upload:**
    Authenticated user uploads file → File Service stores the file → Share link via WebSocket.

### Database Design

-   SQL Database (User Profile):

    -   _Users_:

        ```
        userId (Primary Key)
        email
        passwordHash
        username (Permanent, unique)
        createdAt
        updatedAt
        ```

-   SQLite Database (Chat Messages):

    -   _Messages_:
        ```
        messageId (Primary Key)
        senderId (Foreign Key to User)
        receiverId (Foreign Key to User or Guest ID)
        messageText
        messageType (text, file, etc.)
        timestamp
        sessionId (For guest users to link messages to specific sessions)
        ```

### Microservices Breakdown

-   _API Gateway:_ Routes user requests to the appropriate services (Auth, Chat, File).
-   _Authentication Service:_ Handles user login, token generation, validation.
-   _Chat Service:_ Manages real-time messaging using WebSockets.
-   _File Service:_ Handles file uploads and sharing between users.
-   _Database:_ PostgreSQL/MySQL for user data, SQLite for client-side message storage.

### Final Architecture

```
+---------------------+        +-------------------+        +------------------+
|  Frontend (Client)  |<------>|    API Gateway    |<------>|  Authentication  |
+---------------------+        +-------------------+        |     Service      |
                               |     (Routing)     |        +------------------+
                               +-------------------+        | (JWT validation) |
                                         |                  +------------------+
                               +-------------------+
                               |   Chat Service    |
                               |   (WebSockets)    |
                               +-------------------+
                                         |
                               +-------------------+
                               |   File Service    |
                               |   (File Storage)  |
                               +-------------------+
                                         |
                               +-------------------+
                               |   SQL Database    |
                               |  (User Profiles)  |
                               +-------------------+
                                         |
                               +-------------------+
                               |   SQLite (Client) |
                               |  (Message Storage)|
                               +-------------------+

```

### Deployment & Scaling Considerations

-   _Docker:_ Containerize each service for ease of deployment.
-   _Kubernetes:_ Use Kubernetes to orchestrate and scale microservices efficiently.
-   _Horizontal Scaling:_ WebSocket servers can scale horizontally to handle large numbers of concurrent connections.
-   _Load Balancer:_ Use a load balancer to evenly distribute WebSocket connections across available servers.
