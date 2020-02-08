---
title: "Fan In/Out Pattern In Golang"
date: 2020-02-07T19:01:09+05:30
draft: false
tags: ["golang", "fanin", "fanout", "golang channels", "concurrency", "parallelism"]
---

Golang is absolutely gorgeous when it comes to concurrency. I was trying to implement fan in/out pattern in it. Following is the result.

```golang
package main

import "fmt"

func main() {
	var numOfWorkers int
	// How many workers do you want?
	fmt.Printf("Enter the number of workers to spawn : ")
	fmt.Scanf("%d\n", &numOfWorkers)

	c := make(chan int)
	outChanArr := make([]chan map[int]int, numOfWorkers)
	done := make(chan bool)
	collateChan := make(chan map[int]int)

	// generate some values
	go genFunc(c)

	for s := 0; s < numOfWorkers; s++ {
		// calculate the factorials
		outChanArr[s] = fact(c, done, s)
	}

	// collate results from all channels
	collateFunc(done, collateChan, outChanArr...)

	go func() {
		for s := 0; s < numOfWorkers; s++ {
			<-done // syncronise the goroutines
		}
		close(collateChan)
	}()

	for i := range collateChan {
		fmt.Println(i) // Print the result
	}
}

func genFunc(c chan<- int) {
	for i := 0; i < 100; i++ {
		for j := 0; j < 20; j++ {
			c <- j
		}
	}
	close(c)
}

func fact(cin <-chan int, done chan bool, routineName int) chan map[int]int {
	cout := make(chan map[int]int)
	go func() {
		counter := 0
		for i := range cin {
			out := 1
			for j := i; j > 1; j-- {
				out *= j
			}
			cout <- map[int]int{i: out}
			counter++
		}
		fmt.Println("Goroutine fact ", routineName, "processed", counter, "items")
		close(cout)
		fmt.Println("Goroutine fact", routineName, "is finished")
	}()
	return cout
}

func collateFunc(done chan bool, collateChan chan map[int]int, c ...chan map[int]int) {
	for idx, ci := range c {
		go func(ci chan map[int]int, idx int) {
			counter := 0
			for i := range ci {
				collateChan <- i
				counter++
			}
			fmt.Println("Goroutine consume ", idx, "consumed", counter, "items")
			done <- true
		}(ci, idx)
	}
}
```

As you can see here, in golang we don't have to worry about locking if we use channels. Channels themselves act as 
syncronization agent. On the other hand we can use something called `Waitgroups` to achive the syncronization.

The first part is to specify the number of workers we want to crunch some numbers. `numOfWorkers` stores that value.
`genFunc` is the goroutine here that generates the dummy values, and pushes it on a channel `c`. This task happens in
separate thread, as we have called the `genFunc` with keyword `go`. And the `genFunc` is off and running.
```golang
go genFunc(c) // generate some values
```
We can consider `c` as a conveyor belt on which a
little gopher called `genFunc` is putting some stuff to be processed.

The for loop calls the `fact` function `numOfWorkers` times. It effectively spawns the `numOfWorkers` goroutines parallelly, 
which process the `int` items on channel `c`. Each of one these `fact` goroutines creates a new `cout` channel and puts the 
result on it.
```golang
for s := 0; s < numOfWorkers; s++ {
	// calculate the factorials
	outChanArr[s] = fact(c, done, s)
}
```
If we say we that want 4 workers, 4 goroutines will be spawned, and the channel `c` will be passed down. It is as if 4 gophers
are taking off the items on conveyor belt `c`. This process is called `fan out`. Once each of the goroutines process the items,
the output is put on `cout` channel. These channels are appended to `outChanArr`.

Then we collate the `outChanArr` in `collateFunc` and put it on `collateChan` for displaying it on "standard out". It is as if gophers
 called `collateFunc0, collateFunc1, collateFunc2` are gathering the output from conveyer belts `outChanArr[0], outChanArr[1], outChanArr[2]...` 
 and so on, and putting it on single conveyor belt `outChanArr`. This process is called fan `fan in`.
 These gophers are signalling whether they are done with their tasks on `done`
 channel.

 We are checking whether all of the goroutines are finished with their tasks with anonymous goroutine, which blocks on `done` channel. Once, all
 of the goroutines are done we are closing `collateChan` channel.

 ```golang
go func() {
	for s := 0; s < numOfWorkers; s++ {
		<-done // syncronise the goroutines
	}
	close(collateChan)
}()
 ```

 The last line is just printing the output from channel `collateChan`.

I ran the above code, and the output was,

```bash
pranav@ubuntu:~/goworkspace/src/github.com/pranav93/gopher/fan$ go run main.go
Enter the number of workers to spawn : 4
map[0:1]
map[3:6]
.
.
.
map[14:87178291200]
map[15:1307674368000]
map[18:6402373705728000]
Goroutine fact  0 processed 501 items
Goroutine fact 0 is finished
Goroutine consume  0 consumed 501 items
Goroutine fact  1 processed 497 items
Goroutine fact 1 is finished
Goroutine consume  1 consumed 497 items
Goroutine fact  2 processed 495 items
Goroutine fact 2 is finished
Goroutine consume  2 consumed 495 items
Goroutine fact  3 processed 507 items
Goroutine fact 3 is finished
Goroutine consume  3 consumed 507 items
map[19:121645100408832000]
```

The 2000 items are spread across 4 goroutines almost evenly.
