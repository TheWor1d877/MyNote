多个 goroutine 访问同一份内存时，什么情况下一个 goroutine 的写一定能被另一个 goroutine 看到。它给出的判据就是 happens-before。

