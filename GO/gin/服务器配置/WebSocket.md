WebSocket 是一种在单个 TCP 连接上提供全双工（双向实时）通信的网络协议。

- 工作原理
客户端先发一个 特殊的 HTTP 请求（带 Upgrade: websocket 头）。
服务器同意后，将 HTTP 连接“升级”为 WebSocket 连接。
之后，双方通过这个 TCP 连接直接收发数据帧（Frame），不再有 HTTP 头！

| 特性 | 说明 |
| :--- | :--- |
| 基于 TCP | 可靠、有序、不丢包。 |
| 轻量级帧结构 | 数据以“帧”（Frame）为单位传输，支持分片、压缩、掩码。 |
| 同源策略 & 安全 | 受浏览器同源策略限制；可通过 `wss://`（WebSocket Secure）加密。 |
| 与 HTTP 兼容 | 握手阶段是标准 HTTP，可穿透防火墙和代理（只要它们支持 Upgrade）。 |

## 一个带有心跳机制的ws server示例
```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"sync"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/gorilla/websocket"
)

var (
	mutex       = sync.RWMutex{}
	connections = make(map[*websocket.Conn]struct{})
)
var upgrader = websocket.Upgrader{
	CheckOrigin: func(r *http.Request) bool {
		origin := r.Header.Get("Origin")
		fmt.Println(origin)
		return true
	},
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
}

func broadcastMessage(messageType int, data []byte) {
	mutex.Lock()
	defer mutex.Unlock()
	for conn := range connections {
		if err := conn.WriteMessage(messageType, data); err != nil {
			log.Println("write:", err)
		}
	}
}

func setupWebSocketRoutes(r *gin.Engine) {
	r.GET("/ws", func(c *gin.Context) {
		log.Println("Received request for /ws")
		log.Printf("Headers: %+v\n", c.Request.Header)

		if !websocket.IsWebSocketUpgrade(c.Request) {
			log.Println("Not a WebSocket upgrade request!")
			c.AbortWithStatus(400)
			return
		}

		// Step 1: 升级 HTTP 到 WebSocket
		conn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
		if err != nil {
			c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{"error": "Failed to upgrade connection"})
			return
		}
		mutex.Lock()
		connections[conn] = struct{}{}
		mutex.Unlock()

		log.Printf("New WebSocket connection. Total: %d", len(connections))

		c.Done()

		go handleWebsocketConnection(conn) //
	})
}

func handleWebsocketConnection(conn *websocket.Conn) {
	defer func() {
		mutex.Lock()
		delete(connections, conn)
		mutex.Unlock()
		conn.Close()
		log.Printf("WebSocket connection closed. Total: %d", len(connections))
	}()
	ticker := time.NewTicker(time.Second * 10)
	defer ticker.Stop()

	conn.SetPongHandler(func(string) error {
		log.Printf("pong received")
		return nil
	})

	//pongWaitTime := time.Minute
	//conn.SetReadDeadline(time.Now().Add(pongWaitTime))
	//conn.SetWriteDeadline(time.Now().Add(pongWaitTime / 2))
	go func() {
		for {
			select {
			case <-ticker.C:
				if err := conn.WriteMessage(websocket.PingMessage, nil); err != nil {
					log.Printf("Ping error: %v", err)
					return
				}
			}
		}
	}()
	for {
		messageType, message, err := conn.ReadMessage()
		if err != nil {
			// 连接已断开（正常或异常）
			if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
				log.Printf("Unexpected close: %v", err)
			}
			break
		}

		log.Printf("Received: %s", message)

		// 示例：回显 + 广播
		response := []byte("Echo: " + string(message))
		if err := conn.WriteMessage(messageType, response); err != nil {
			log.Printf("Write error: %v", err)
			break
		}

		broadcastMessage(messageType, message)
	}
}

func main() {
	gin.SetMode(gin.ReleaseMode)
	r := gin.New()
	setupWebSocketRoutes(r)
	r.GET("/health", func(c *gin.Context) {
		mutex.RLock()
		count := len(connections)
		mutex.RUnlock()
		c.JSON(200, gin.H{
			"status": "ok",
			"count":  count,
		})
	})
	r.Run(":8080")
}

```