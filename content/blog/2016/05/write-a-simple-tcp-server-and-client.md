+++
author = ""
categories = ["Go"]
date = "2016-05-27T17:12:51+08:00"
description = ""
linktitle = ""
title = "用Go写一个简单的TCP server or client 模型"
type = "post"

+++

## 用Go写一个简单的TCP server or client 模型

### 对Conn封装的基本思路

go内置了net包已经很好的封装了socket通讯。然而在实际使用中，由于net/Conn的Read/Write方法是堵塞的原因，必须将其放入单独的goroutine中进行处理。

我们先简单的整理下思路，对于连接(Conn)的处理，我们可以开启2条goroutine进行处理，一条用于堵塞的Read的处理，另一条进行Write的处理。

这里必须指出，其实Write本身就是线程安全的，也就是我们在别的任何地方都可以进行Write，但是Write是堵塞的，所以考虑到这点，我们还是将其放入一个单独的goroutine中进行处理。

这样设计的原因在于Conn是支持同时Read/Write的，这样我们的基本的Conn的模型就成型了。对于服务端或者客户端而言，我们只需要封装对应过来的Conn即可，Conn的读写goroutine进行处理，并将获得的事件抛向外部。

那么我们就按这个思路来实现一个简单的Connection封装，该封装支持线程安全的写，并且支持解包操作。

	package tcpnetwork
	
	import (
		"errors"
		"log"
		"net"
		"time"
	)
	
	const (
		kConnStatus_None = iota
		kConnStatus_Connected
		kConnStatus_Disconnected
	)
	
	const (
		kConnEvent_None = iota
		kConnEvent_Connected
		kConnEvent_Disconnected
		kConnEvent_Data
		kConnEvent_Close
	)
	
	const (
		kConnConf_DefaultSendTimeoutSec = 5
		kConnConf_MaxReadBufferLength   = 0xffff // 0xffff
	)
	
	type Connection struct {
		conn                net.Conn
		status              int
		connId              int
		sendMsgQueue        chan []byte
		sendTimeoutSec      int
		eventQueue          IEventQueue
		streamProtocol      IStreamProtocol
		maxReadBufferLength int
		userdata            interface{}
		from                int
		readTimeoutSec      int
	}
	
	func newConnection(c net.Conn, sendBufferSize int, eq IEventQueue) *Connection {
		return &Connection{
			conn:                c,
			status:              kConnStatus_None,
			connId:              0,
			sendMsgQueue:        make(chan []byte, sendBufferSize),
			sendTimeoutSec:      kConnConf_DefaultSendTimeoutSec,
			maxReadBufferLength: kConnConf_MaxReadBufferLength,
			eventQueue:          eq,
		}
	}
	
	type ConnEvent struct {
		EventType int
		Conn      *Connection
		Data      []byte
	}
	
	func newConnEvent(et int, c *Connection, d []byte) *ConnEvent {
		return &ConnEvent{
			EventType: et,
			Conn:      c,
			Data:      d,
		}
	}
	
	//	directly close, packages in queue will not be sent
	func (this *Connection) close() {
		if kConnStatus_Connected != this.status {
			return
		}
	
		this.conn.Close()
		this.status = kConnStatus_Disconnected
	}
	
	func (this *Connection) Close() {
		if this.status != kConnStatus_Connected {
			return
		}
	
		select {
		case this.sendMsgQueue <- nil:
			{
				//	nothing
			}
		case <-time.After(time.Duration(this.sendTimeoutSec)):
			{
				//	timeout, close the connection
				this.close()
				log.Printf("Con[%d] send message timeout, close it", this.connId)
			}
		}
	}
	
	func (this *Connection) pushEvent(et int, d []byte) {
		if nil == this.eventQueue {
			log.Println("Nil event queue")
			return
		}
		this.eventQueue.Push(newConnEvent(et, this, d))
	}
	
	func (this *Connection) GetStatus() int {
		return this.status
	}
	
	func (this *Connection) setStatus(stat int) {
		this.status = stat
	}
	
	func (this *Connection) GetConnId() int {
		return this.connId
	}
	
	func (this *Connection) SetConnId(id int) {
		this.connId = id
	}
	
	func (this *Connection) GetUserdata() interface{} {
		return this.userdata
	}

	func (this *Connection) SetUserdata(ud interface{}) {
		this.userdata = ud
	}

	func (this *Connection) SetReadTimeoutSec(sec int) {
		this.readTimeoutSec = sec
	}

	func (this *Connection) GetReadTimeoutSec() int {
		return this.readTimeoutSec
	}

	func (this *Connection) setStreamProtocol(sp IStreamProtocol) {
		this.streamProtocol = sp
	}

	func (this *Connection) sendRaw(msg []byte) {
		if this.status != kConnStatus_Connected {
			return
		}
	
		select {
		case this.sendMsgQueue <- msg:
			{
				//	nothing
			}
		case <-time.After(time.Duration(this.sendTimeoutSec)):
			{
				//	timeout, close the connection
				this.close()
				log.Printf("Con[%d] send message timeout, close it", this.connId)
			}
		}
	}

	func (this *Connection) Send(msg []byte, cpy bool) {
		if this.status != kConnStatus_Connected {
			return
		}
	
		buf := msg
		if cpy {
			msgCopy := make([]byte, len(msg))
			copy(msgCopy, msg)
			buf = msgCopy
		}
	
		select {
		case this.sendMsgQueue <- buf:
			{
				//	nothing
			}
		case <-time.After(time.Duration(this.sendTimeoutSec)):
			{
				//	timeout, close the connection
				this.close()
				log.Printf("Con[%d] send message timeout, close it", this.connId)
			}
		}
	}

	//	run a routine to process the connection
	func (this *Connection) run() {
		go this.routineMain()
	}

	func (this *Connection) routineMain() {
		defer func() {
			//	routine end
			log.Printf("Routine of connection[%d] quit", this.connId)
			e := recover()
			if e != nil {
				log.Println("Panic:", e)
			}
	
			//	close the connection
			this.close()
	
			//	free channel
			close(this.sendMsgQueue)
			this.sendMsgQueue = nil
	
			//	post event
			this.pushEvent(kConnEvent_Disconnected, nil)
		}()
	
		if nil == this.streamProtocol {
			log.Println("Nil stream protocol")
			return
		}
		this.streamProtocol.Init()
	
		//	connected
		this.pushEvent(kConnEvent_Connected, nil)
		this.status = kConnStatus_Connected
	
		go this.routineSend()
		this.routineRead()
	}

	func (this *Connection) routineSend() error {
		defer func() {
			log.Println("Connection", this.connId, " send loop return")
		}()
	
		for {
			select {
			case evt, ok := <-this.sendMsgQueue:
				{
					if !ok {
						//	channel closed, quit
						return nil
					}
	
					if nil == evt {
						log.Println("User disconnect")
						this.close()
						return nil
					}
	
					var err error
	
					headerBytes := this.streamProtocol.SerializeHeader(evt)
					if nil != headerBytes {
						//	write header first
						_, err = this.conn.Write(headerBytes)
						if err != nil {
							log.Println("Conn write error:", err)
							return err
						}
					}
	
					_, err = this.conn.Write(evt)
					if err != nil {
						log.Println("Conn write error:", err)
						return err
					}
				}
			}
		}
	
		return nil
	}

	func (this *Connection) routineRead() error {
		//	default buffer
		buf := make([]byte, this.maxReadBufferLength)
	
		for {
			msg, err := this.unpack(buf)
			if err != nil {
				log.Println("Conn read error:", err)
				return err
			}
	
			this.pushEvent(kConnEvent_Data, msg)
		}
	
		return nil
	}

    func (this *Connection) unpack(buf []byte) ([]byte, error) {
		//	read head
		if 0 != this.readTimeoutSec {
			this.conn.SetReadDeadline(time.Now().Add(time.Duration(this.readTimeoutSec) * time.Second))
		}
		headBuf := buf[:this.streamProtocol.GetHeaderLength()]
		_, err := this.conn.Read(headBuf)
		if err != nil {
			return nil, err
		}
	
		//	check length
		packetLength := this.streamProtocol.UnserializeHeader(headBuf)
		if packetLength > this.maxReadBufferLength ||
			0 == packetLength {
			return nil, errors.New("The stream data is too long")
		}
	
		//	read body
		if 0 != this.readTimeoutSec {
			this.conn.SetReadDeadline(time.Now().Add(time.Duration(this.readTimeoutSec) * time.Second))
		}
		bodyLength := packetLength - this.streamProtocol.GetHeaderLength()
		_, err = this.conn.Read(buf[:bodyLength])
		if err != nil {
			return nil, err
		}

		//	ok
		msg := make([]byte, bodyLength)
		copy(msg, buf[:bodyLength])
		if 0 != this.readTimeoutSec {
			this.conn.SetReadDeadline(time.Time{})
		}

		return msg, nil
	}


这就是简单的Conn封装，在抛出事件这部，定义了一个interface来接收事件。

	package tcpnetwork
	
	type IEventQueue interface {
		Push(*ConnEvent)
		Pop() *ConnEvent
	}
	
	type IStreamProtocol interface {
		//	Init
		Init()
		//	get the header length of the stream
		GetHeaderLength() int
		//	read the header length of the stream
		UnserializeHeader([]byte) int
		//	format header
		SerializeHeader([]byte) []byte
	}
	
	type IEventHandler interface {
		OnConnected(evt *ConnEvent)
		OnDisconnected(evt *ConnEvent)
		OnRecv(evt *ConnEvent)
	}

我们只要在实现对应的方法，就可以接收事件和读取事件了。

### Server/Client 端的实现

#### Server

我们已经封装好了Conn，那么接下来的工作将会简单很多。我们来封装一个TCPNetwork的结构。

对于Server端来说，它基本的步骤就是

* 监听端口
* 开启accept线程
* 得到Conn，并生成Connection，开启Read/Write线程

第一步很简单，net包已封装

	ls, err := net.Listen("tcp", addr)
	if nil != err {
		return err
	}
	
	//	accept
	this.listener = ls
	go this.acceptRoutine()
	return nil

我们在listen成功后，开启了一个goroutine来不断的进行死循环来等待连接的接入。

	func (this *TCPNetwork) acceptRoutine() {
		for {
			conn, err := this.listener.Accept()
			if err != nil {
				log.Println("accept routine quit.error:", err)
				return
			}
	
			//	process conn event
			connection := this.createConn(conn)
			connection.SetReadTimeoutSec(this.readTimeoutSec)
			connection.from = 0
			connection.run()
		}
	}

得到了Connection后，我们只需要让它的处理routine跑起来即可。然而我们需要对应的Connection抛过来的事件，于是我们在TCPNetwork中实现IEventQueue的2个方法，这样我们的TCPNetwork就可以接收对应的事件的抛入，也可以读取，大家读到这里也就知道了，最适合实现这个的就是golang的神器：channel。

我们来为TCPNetwork定义一个chan *ConnEvent，并实现接口的方法。

	func (this *TCPNetwork) Push(evt *ConnEvent) {
		if nil == this.eventQueue {
			return
		}
		this.eventQueue <- evt
	}
	
	func (this *TCPNetwork) Pop() *ConnEvent {
		evt, ok := <-this.eventQueue
		if !ok {
			//	event queue already closed
			this.eventQueue = nil
			return nil
		}
	
		return evt
	}

其实基本的逻辑已经完成了，对于我们来说，只需要关心eventQueue里的内容就行了，这就属于上层逻辑处理了。这样一个简单的TCPNetwork就封装好了。

#### Client

Client端基本没什么好说的，net包Connect成功后会获得一个Conn，然后对于这个Conn的处理其实和Server端一样了。

	func (this *TCPNetwork) Connect(addr string) error {
		conn, err := net.Dial("tcp", addr)
		if nil != err {
			return err
		}
	
		connection := this.createConn(conn)
		connection.from = 1
		connection.run()
	
		return nil
	}

#### 使用方法

对于外部来说，使用很简单，这里贴上一个简单的echo server example.

	package main
	
	import (
		"log"
	
		"github.com/sryanyuan/tcpnetwork"
	)
	
	type TSShutdown struct {
		network *tcpnetwork.TCPNetwork
	}
	
	func NewTSShutdown() *TSShutdown {
		t := &TSShutdown{}
		t.network = tcpnetwork.NewTCPNetwork(1024, tcpnetwork.NewStreamProtocol4())
		return t
	}
	
	func (this *TSShutdown) OnConnected(evt *tcpnetwork.ConnEvent) {
		log.Println("connected ", evt.Conn.GetConnId())
	}
	
	func (this *TSShutdown) OnDisconnected(evt *tcpnetwork.ConnEvent) {
		log.Println("disconnected ", evt.Conn.GetConnId())
	}
	
	func (this *TSShutdown) OnRecv(evt *tcpnetwork.ConnEvent) {
		log.Println("recv ", evt.Conn.GetConnId(), evt.Data)
	
		evt.Conn.Send(evt.Data, false)
	}
	
	func (this *TSShutdown) Run() {
		err := this.network.Listen("127.0.0.1:2222")

		if err != nil {
			log.Println(err)
			return
		}
	
		this.network.ServeWithHandler(this)
		log.Println("done")
	}


	package main
	
	import (
		"fmt"
		"log"
	)
	
	func main() {
		defer func() {
			e := recover()
			if e != nil {
				log.Println(e)
			}
			var inp int
			fmt.Scanln(&inp)
		}()
		tsshutdown := NewTSShutdown()
		tsshutdown.Run()
	}



### 总结

我们其实已经实现了一个简单的tcp封装了，支持server/client连接，至于其中心跳的细节、封包解包的细节等等，这里就不多介绍了，可以通过阅读源码来理解。

该封装可以见我的Github [TCPNetwork](https://github.com/sryanyuan/tcpnetwork)