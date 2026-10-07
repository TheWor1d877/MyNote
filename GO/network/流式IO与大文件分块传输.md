**实现高效、低内存的大文件上传/下载，避免 OOM（Out-Of-Memory），支撑云原生存储的核心能力**

## 不能直接ioutil.ReadAll
会造成OOM，
并且延迟高，必须等待全部读完才能继续，
如果网络中断，必须要全部重传

## 解决办法
***固定缓冲区 +  分块思想***
内存占用小
支持断点续传
网络可并行
#### 代码逻辑
client.go
```go
func uploadFile(conn net.Conn, filepath string, fileID string) {
	file, err := os.OpenFile(filepath, os.O_CREATE|os.O_RDWR, 0666)
	if err != nil {
		log.Fatal(err)
	}
	defer file.Close()

	const chunkSize = 64 * 1024
	buffer := make([]byte, chunkSize)
	offset := 0
	for {
		n, err := file.Read(buffer[offset:])
		if err != nil {
			break
		}
		chunk := &Chunk{
			FileID: fileID,
			Offset: int64(offset),
			Size:   int64(n),
			EOF:    err == io.EOF,
		}
		hdr, _ := json.Marshal(chunk)
		message := append(hdr, '\n')
		message = append(message, buffer[:n]...)

		sendMsg(conn, message)
		offset += n
		if err == io.EOF {
			break
		}
	}
}

// 服务端发送
func downloadFile(conn net.Conn, fileID string) {
    file, _ := os.Open(fileID)
    defer file.Close()

    buf := make([]byte, 4*1024*1024)
    for {
        n, err := file.Read(buf)
        if n > 0 {
            sendMsg(conn, buf[:n]) // 直接发数据（可加 header）
        }
        if err == io.EOF {
            break
        }
    }
}
```

server.go
```go
type Chunk struct {
	FileID string `json:"file_id"`
	Offset int64  `json:"offset"`
	Size   int64  `json:"size"`
	EOF    bool   `json:"eof"`
}

func handleUpload(conn net.Conn) {
	buf := make([]byte, 1024*1024*4) // 4MB Buffer
	for {
		//读length
		if _, err := io.ReadFull(conn, buf); err != nil {
			break
		}
		msgLen := binary.BigEndian.Uint64(buf[:4])

		//读完整消息
		if _, err := io.ReadFull(conn, buf[4:4+msgLen]); err != nil {
			break
		}
		payLoad := buf[4 : 4+msgLen]

		//解析header + data
		var chunk Chunk
		hdrEnd := bytes.IndexByte(payLoad, '\n')
		err := json.Unmarshal(payLoad[hdrEnd+1:], &chunk)
		if err != nil {
			break
		}
		data := payLoad[hdrEnd+1:]

		// 写入本地文件
		f, _ := os.OpenFile(chunk.FileID, os.O_CREATE|os.O_RDWR, 0666)
		_, err = f.Seek(chunk.Offset, 0)
		if err != nil {
			return
		}
		_, err = f.Write(data)
		if err != nil {
			return
		}
		err = f.Close()
		if err != nil {
			return
		}
		if chunk.EOF {
			log.Println("EOF")
			break
		}
	}
}

```

