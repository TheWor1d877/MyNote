TCP是字节流协议，没有消息边界
直接conn.Read解析消息会出现粘包或者拆包的问题

发送方：
```go
conn.Write([]byte("msg1"))
conn.Write([]byte("msg2"))
```
接收方：
可能一次 Read() 得到
"msg1msg2"（粘包）
"msg1ms" 和 "g2"（拆包）
所以必须在应用层定义消息边界

## 解决方案：长度字段帧(Length_field)
```text
+----------+------------------+
| 4字节长度 |   N字节 payload  |
+----------+------------------+
```
现读取四字节长度的字节数，这个数字
## GO: bufio.Reader + io.ReadFull 解决问题
```go
func ReadMessage(reader *bufio.Reader) ([]byte, error) {
	lenBuf := make([]byte, 4)
	if _, err := io.ReadFull(reader, lenBuf); err != nil {
		return nil, err
	}
	length := binary.BigEndian.Uint32(lenBuf)
	if length > 10<<20 {
		return nil, errors.New("message too large")
	}

	payload := make([]byte, length)
	if _, err := io.ReadFull(reader, payload); err != nil {
		return nil, err
	}

	return payload, nil
}

func handleConn(conn net.Conn) {
	defer func() {
		err := conn.Close()
		if err != nil {
			log.Fatalln(fmt.Errorf("error closing connection: %v", err))
		}
	}()
	reader := bufio.NewReader(conn)
	for {
		msg, err := ReadMessage(reader)
		if err != nil {
			if err == io.EOF {
				log.Println("connection closed")
			} else {
				log.Printf("read error: %v", err)
			}
			return
		}

		log.Printf("Receive: %v\n", msg)
	}
}
```