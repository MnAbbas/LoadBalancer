# Qualitylist
``` go
var qualitylist map[(chan Request)]float64
```
this represents the situation of TimeSeries and it allows us to pick up the best quality service 
`qualitylist` has a key which reperesent `chan Request` and a value which reperesent the lasttime response time

# RegisterInstance
I add a map variable which hold status of TimeService response and their corresponding channel

```go
func (lb *MyLoadBalancer) RegisterInstance(ch chan Request) {
	if qualitylist == nil {
		qualitylist = make(map[(chan Request)]float64)

	}

	qualitylist[ch] = 0
	return
}
```
default value for the first time is 0 means it provide quick response to let request method try it and change with actual response time
# Request method
search through `qualitylist` to find best possible channel to send request 
```go
func (lb *MyLoadBalancer) Request(payload interface{}) chan Response {
	ch := make(chan Response, 1)

	rq := Request{
		Payload: payload,
		RspChan: ch,
	}
	bestchannel, err := lb.PosibleNode()
	if err != nil {
		ch <- err.Error()
		return ch
	}

	start := time.Now()
	bestchannel <- rq

	for {
		select {
		case mrt := <-ch:
			t := time.Now()
			elapsed := t.Sub(start)
			qualitylist[bestchannel] = elapsed.Seconds()
			ch <- mrt
			return ch
		// case <-time.After(4 * time.Second):
		// 	// delete(qualitylist, bestchannel)
		// 	ch <- "TimeOut"
		// 	return ch

		}
	}
}
```
first check the available channels and for the first time will call them and record the duration of getting response and for the next time it will find the shortest time to deal with

# Trade-offs
## First trade-off
I needed to have a variable which both `RegisterInstance` and `Request` could share between themselves
I had two choice 
first :
```go
var qualitylist map[(chan Request)]float64
```
which `RegisterInstance` could add an item to it and `Request` could range the items and find best possible `TimeService`
second :

```go
type MyLoadBalancer struct {
	qualitylist map[(chan Request)]float64
}
```
change `MyLoadBalancer` struct which allows both methods to access to it 

The second one seems more structured and encapsulates data but for `Kill` situation I needed an item which can be assessable within `TimeService.Dead` so I choose the first one

## changing TimeService.Run()

due to killing `TimeService` it was necessary to remove the killed time service from the available array so I had two choice to deal with

first :
handle it within `Request` method, if I didn't get a response from channel then looking for another request channel and delete the current channel from `qualitylist` 
```go
case <-time.After(4 * time.Second):
			delete(qualitylist, bestchannel)
			ch <- "TimeOut"
			return ch
```
second : 
handle it within `TimeService.Dead` and remove from list 
```go
			delete(qualitylist, ts.ReqChan)
```

The first one seems to be feasible because it was possible to kill more than two channel then a response will exceed 5 second and the `TimeOut` will arise 


# Suggestion
I have some suggestion to use in order to make it better

## increase or change the Buffer size
With current code in `Spawn` and via defining `ReqChan` because of channel buffer
```go
		ReqChan:         make(chan Request, 10),
```
it causes `TimeOut` because when 10 requests go for a TimeService it will cause a delay 

### use UnRegisterInstance method
it makes the code more readable if we could use 
```go
type MyLoadBalancer struct{
    qualitylist map[(chan Request)]float64

}

func (lb *MyLoadBalancer) UnRegisterInstance(ch chan Request) {
	delete(lb.qualitylist, ch)
	return
}

```
in order to do that, we must add a new channel, which the change in the status of TimeService status will be informed and base of status (spawn, kill,...) proper action will be taken

```go
type SignalChannel struct {
	Status interface{}
	Req chan Request
}

```

### use Observer Design Pattern

another way which has been used in OOP is Observer Design Pattern which will allow the concrete object or struct to inform other structs that something happened and execute a method or series of the method from another structure,