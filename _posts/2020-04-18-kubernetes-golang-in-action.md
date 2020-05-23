---
title: Kubernetes 源码学习时golang的实现套路
subtitle: golang 在kubernetes中的实现套路
layout: post
tags: [golang,kuberntes]
---



go在kuberntes实现过程当中的总结。

## 异常处理

runtime统一处理异常：

- error处理
- crash处理



```go
var (
   // ReallyCrash controls the behavior of HandleCrash and now defaults
   // true. It's still exposed so components can optionally set to false
   // to restore prior behavior.
   ReallyCrash = true
)

// PanicHandlers is a list of functions which will be invoked when a panic happens.
var PanicHandlers = []func(interface{}){logPanic}

// HandleCrash simply catches a crash and logs an error. Meant to be called via
// defer.  Additional context-specific handlers can be provided, and will be
// called in case of panic.  HandleCrash actually crashes, after calling the
// handlers and logging the panic message.
//
// E.g., you can provide one or more additional handlers for something like shutting down go routines gracefully.
func HandleCrash(additionalHandlers ...func(interface{})) {
   if r := recover(); r != nil {
      for _, fn := range PanicHandlers {
         fn(r)
      }
      for _, fn := range additionalHandlers {
         fn(r)
      }
      if ReallyCrash {
         // Actually proceed to panic.
         panic(r)
      }
   }
}

// logPanic logs the caller tree when a panic occurs (except in the special case of http.ErrAbortHandler).
func logPanic(r interface{}) {
   if r == http.ErrAbortHandler {
      // honor the http.ErrAbortHandler sentinel panic value:
      //   ErrAbortHandler is a sentinel panic value to abort a handler.
      //   While any panic from ServeHTTP aborts the response to the client,
      //   panicking with ErrAbortHandler also suppresses logging of a stack trace to the server's error log.
      return
   }

   // Same as stdlib http server code. Manually allocate stack trace buffer size
   // to prevent excessively large logs
   const size = 64 << 10
   stacktrace := make([]byte, size)
   stacktrace = stacktrace[:runtime.Stack(stacktrace, false)]
   if _, ok := r.(string); ok {
      klog.Errorf("Observed a panic: %s\n%s", r, stacktrace)
   } else {
      klog.Errorf("Observed a panic: %#v (%v)\n%s", r, r, stacktrace)
   }
}

// ErrorHandlers is a list of functions which will be invoked when an unreturnable
// error occurs.
// 当一个error发生时，这些func会被调用
var ErrorHandlers = []func(error){
   logError,
   (&rudimentaryErrorBackoff{
      lastErrorTime: time.Now(),
      // 1ms was the number folks were able to stomach as a global rate limit.
      // If you need to log errors more than 1000 times a second you
      // should probably consider fixing your code instead. :)
      minPeriod: time.Millisecond,
   }).OnError,
}

// HandlerError is a method to invoke when a non-user facing piece of code cannot
// return an error and needs to indicate it has been ignored. Invoking this method
// is preferable to logging the error - the default behavior is to log but the
// errors may be sent to a remote server for analysis.
func HandleError(err error) {
   // 当有错误发生时，调用注册好的Error处理函数
   if err == nil {
      return
   }

   for _, fn := range ErrorHandlers {
      fn(err)
   }
}

// logError prints an error with the call stack of the location it was reported
func logError(err error) {
   klog.ErrorDepth(2, err)
}

//原始错误回滚
type rudimentaryErrorBackoff struct {
   minPeriod time.Duration // immutable
   // TODO(lavalamp): use the clock for testability. Need to move that
   // package for that to be accessible here.
   lastErrorTimeLock sync.Mutex
   lastErrorTime     time.Time
}

// OnError will block if it is called more often than the embedded period time.
// This will prevent overly tight hot error loops.
func (r *rudimentaryErrorBackoff) OnError(error) {
   r.lastErrorTimeLock.Lock()
   defer r.lastErrorTimeLock.Unlock()
   d := time.Since(r.lastErrorTime)
   if d < r.minPeriod {
      // If the time moves backwards for any reason, do nothing
      time.Sleep(r.minPeriod - d)
   }
   r.lastErrorTime = time.Now()
}

// GetCaller returns the caller of the function that calls it.
func GetCaller() string {
   var pc [1]uintptr
   runtime.Callers(3, pc[:])
   f := runtime.FuncForPC(pc[0])
   if f == nil {
      return fmt.Sprintf("Unable to find caller")
   }
   return f.Name()
}

// RecoverFromPanic replaces the specified error with an error containing the
// original error, and  the call tree when a panic occurs. This enables error
// handlers to handle errors and panics the same way.
func RecoverFromPanic(err *error) {
   if r := recover(); r != nil {
      // Same as stdlib http server code. Manually allocate stack trace buffer size
      // to prevent excessively large logs
      const size = 64 << 10
      stacktrace := make([]byte, size)
      stacktrace = stacktrace[:runtime.Stack(stacktrace, false)]

      *err = fmt.Errorf(
         "recovered from panic %q. (err=%v) Call stack:\n%s",
         r,
         *err,
         stacktrace)
   }
}

// Must panics on non-nil errors.  Useful to handling programmer level errors.
func Must(err error) {
   if err != nil {
      panic(err)
   }
}
```

## wait：拉取、监听Condition的改变

#### 1. Group ：启动一组goroutines并等待他们完成

实现原理：

- 封装了 `sync.WaitGroup`
- `g.Start`自动将wg加一`wg.Add(1)`，并启动一个goroutine并调用defer语句`defer g.wg.Done()` 将wg的Add释放

```go
//
type Group struct {
   wg sync.WaitGroup
}

func (g *Group) Wait() {
   g.wg.Wait()
}

// StartWithChannel starts f in a new goroutine in the group.
// stopCh is passed to f as an argument. f should stop when stopCh is available.
func (g *Group) StartWithChannel(stopCh <-chan struct{}, f func(stopCh <-chan struct{})) {
   g.Start(func() {
      f(stopCh)
   })
}

// StartWithContext starts f in a new goroutine in the group.
// ctx is passed to f as an argument. f should stop when ctx.Done() is available.
func (g *Group) StartWithContext(ctx context.Context, f func(context.Context)) {
   g.Start(func() {
      f(ctx)
   })
}

// Start starts f in a new goroutine in the group.
func (g *Group) Start(f func()) {
   g.wg.Add(1)
   go func() {
      defer g.wg.Done()
      f()
   }()
}
```





#### 2. Until：周期性调用f函数，直到stop channel 进行close调用关闭channel

Until是一个语法糖，封装了JitterUntil。



```go
// Until is syntactic sugar on top of JitterUntil with zero jitter factor and
// with sliding = true (which means the timer for period starts after the f
// completes).
func Until(f func(), period time.Duration, stopCh <-chan struct{}) {
  //参数：
   JitterUntil(f, period, 0.0, true, stopCh)
}

// UntilWithContext loops until context is done, running f every period.
//
// UntilWithContext is syntactic sugar on top of JitterUntilWithContext
// with zero jitter factor and with sliding = true (which means the timer
// for period starts after the f completes).
func UntilWithContext(ctx context.Context, f func(context.Context), period time.Duration) {
   JitterUntilWithContext(ctx, f, period, 0.0, true)
}


```





### 3. JitterUntil：loops until stop channel is closed, running f every period.



```go

// JitterUntil loops until stop channel is closed, running f every period.
//
// If jitterFactor is positive, the period is jittered before every run of f.
// If jitterFactor is not positive, the period is unchanged and not jittered.
//
// If sliding is true, the period is computed after f runs. If it is false then
// period includes the runtime for f.
//
// Close stopCh to stop. f may not be invoked if stop channel is already
// closed. Pass NeverStop to if you don't want it stop.
func JitterUntil(f func(), period time.Duration, jitterFactor float64, sliding bool, stopCh <-chan struct{}) {
	var t *time.Timer
	var sawTimeout bool

	for {
		select {
		case <-stopCh:
			return
		default:
		}

		jitteredPeriod := period
		if jitterFactor > 0.0 {
			//基于base的周期时间，利用抖动生成随机的时间 jitteredPeriod（抖动的周期时间）
			jitteredPeriod = Jitter(period, jitterFactor)
		}

		//是否滑动：重置或者重用定时器
		if !sliding {
			t = resetOrReuseTimer(t, jitteredPeriod, sawTimeout)
		}

		//匿名函数，调用核心逻辑函数
		func() {
			defer runtime.HandleCrash()
			f()
		}()

		if sliding {
			t = resetOrReuseTimer(t, jitteredPeriod, sawTimeout)
		}

		// NOTE: b/c there is no priority selection in golang
		// it is possible for this to race, meaning we could
		// trigger t.C and stopCh, and t.C select falls through.
		// In order to mitigate we re-check stopCh at the beginning
		// of every loop to prevent extra executions of f().
		select {
		case <-stopCh:
			return
		case <-t.C:
			sawTimeout = true
		}
	}
}

// JitterUntilWithContext loops until context is done, running f every period.
//
// If jitterFactor is positive, the period is jittered before every run of f.
// If jitterFactor is not positive, the period is unchanged and not jittered.
//
// If sliding is true, the period is computed after f runs. If it is false then
// period includes the runtime for f.
//
// Cancel context to stop. f may not be invoked if context is already expired.
func JitterUntilWithContext(ctx context.Context, f func(context.Context), period time.Duration, jitterFactor float64, sliding bool) {
	JitterUntil(func() { f(ctx) }, period, jitterFactor, sliding, ctx.Done())
}

// Jitter returns a time.Duration between duration and duration + maxFactor *
// duration.
//
// This allows clients to avoid converging on periodic behavior. If maxFactor
// is 0.0, a suggested default value will be chosen.
func Jitter(duration time.Duration, maxFactor float64) time.Duration {
	if maxFactor <= 0.0 {
		maxFactor = 1.0
	}
	wait := duration + time.Duration(rand.Float64()*maxFactor*float64(duration))
	return wait
}

```





### 4. ConditionFunc：returns true if the condition is satisfied, or an error



```go
// if the loop should be aborted.
type ConditionFunc func() (done bool, err error)
```

### 5. Backoff ： 

```go
// Backoff holds parameters applied to a Backoff function.
type Backoff struct {
   // The initial duration.
   Duration time.Duration
   // Duration is multiplied by factor each iteration, if factor is not zero
   // and the limits imposed by Steps and Cap have not been reached.
   // Should not be negative.
   // The jitter does not contribute to the updates to the duration parameter.
   Factor float64
   // The sleep at each iteration is the duration plus an additional
   // amount chosen uniformly at random from the interval between
   // zero and `jitter*duration`.
   Jitter float64
   // The remaining number of iterations in which the duration
   // parameter may change (but progress can be stopped earlier by
   // hitting the cap). If not positive, the duration is not
   // changed. Used for exponential backoff in combination with
   // Factor and Cap.
   Steps int
   // A limit on revised values of the duration parameter. If a
   // multiplication by the factor parameter would make the duration
   // exceed the cap then the duration is set to the cap and the
   // steps parameter is set to zero.
   Cap time.Duration
}

// Step (1) returns an amount of time to sleep determined by the
// original Duration and Jitter and (2) mutates the provided Backoff
// to update its Steps and Duration.
func (b *Backoff) Step() time.Duration {
   if b.Steps < 1 {
      if b.Jitter > 0 {
         return Jitter(b.Duration, b.Jitter)
      }
      return b.Duration
   }
   b.Steps--

   duration := b.Duration

   // calculate the next step
   if b.Factor != 0 {
      b.Duration = time.Duration(float64(b.Duration) * b.Factor)
      if b.Cap > 0 && b.Duration > b.Cap {
         b.Duration = b.Cap
         b.Steps = 0
      }
   }

   if b.Jitter > 0 {
      duration = Jitter(duration, b.Jitter)
   }
   return duration
}

// contextForChannel derives a child context from a parent channel.
//
// The derived context's Done channel is closed when the returned cancel function
// is called or when the parent channel is closed, whichever happens first.
//
// Note the caller must *always* call the CancelFunc, otherwise resources may be leaked.
func contextForChannel(parentCh <-chan struct{}) (context.Context, context.CancelFunc) {
   ctx, cancel := context.WithCancel(context.Background())

   go func() {
      select {
      case <-parentCh:
         cancel()
      case <-ctx.Done():
      }
   }()
   return ctx, cancel
}

// ExponentialBackoff repeats a condition check with exponential backoff.
//
// It repeatedly checks the condition and then sleeps, using `backoff.Step()`
// to determine the length of the sleep and adjust Duration and Steps.
// Stops and returns as soon as:
// 1. the condition check returns true or an error,
// 2. `backoff.Steps` checks of the condition have been done, or
// 3. a sleep truncated by the cap on duration has been completed.
// In case (1) the returned error is what the condition function returned.
// In all other cases, ErrWaitTimeout is returned.
func ExponentialBackoff(backoff Backoff, condition ConditionFunc) error {
   for backoff.Steps > 0 {
      if ok, err := condition(); err != nil || ok {
         return err
      }
      if backoff.Steps == 1 {
         break
      }
      time.Sleep(backoff.Step())
   }
   return ErrWaitTimeout
}


```



### 6. Poll：超时或者条件满足

 Poll tries a condition func until it returns true, an error, or the timeout is reached.

```go
// Poll always waits the interval before the run of 'condition'.
// 'condition' will always be invoked at least once.
//
// Some intervals may be missed if the condition takes too long or the time
// window is too short.
//
// If you want to Poll something forever, see PollInfinite.
func Poll(interval, timeout time.Duration, condition ConditionFunc) error {
   return pollInternal(poller(interval, timeout), condition)
}

func pollInternal(wait WaitFunc, condition ConditionFunc) error {
   done := make(chan struct{})
   defer close(done)
   return WaitFor(wait, condition, done)
}

// PollImmediate tries a condition func until it returns true, an error, or the timeout
// is reached.
//
// PollImmediate always checks 'condition' before waiting for the interval. 'condition'
// will always be invoked at least once.
//
// Some intervals may be missed if the condition takes too long or the time
// window is too short.
//
// If you want to immediately Poll something forever, see PollImmediateInfinite.
func PollImmediate(interval, timeout time.Duration, condition ConditionFunc) error {
   return pollImmediateInternal(poller(interval, timeout), condition)
}

func pollImmediateInternal(wait WaitFunc, condition ConditionFunc) error {
   done, err := condition()
   if err != nil {
      return err
   }
   if done {
      return nil
   }
   return pollInternal(wait, condition)
}

// PollInfinite tries a condition func until it returns true or an error
//
// PollInfinite always waits the interval before the run of 'condition'.
//
// Some intervals may be missed if the condition takes too long or the time
// window is too short.
func PollInfinite(interval time.Duration, condition ConditionFunc) error {
   done := make(chan struct{})
   defer close(done)
   return PollUntil(interval, condition, done)
}

// PollImmediateInfinite tries a condition func until it returns true or an error
//
// PollImmediateInfinite runs the 'condition' before waiting for the interval.
//
// Some intervals may be missed if the condition takes too long or the time
// window is too short.
func PollImmediateInfinite(interval time.Duration, condition ConditionFunc) error {
   done, err := condition()
   if err != nil {
      return err
   }
   if done {
      return nil
   }
   return PollInfinite(interval, condition)
}

// PollUntil tries a condition func until it returns true, an error or stopCh is
// closed.
//
// PollUntil always waits interval before the first run of 'condition'.
// 'condition' will always be invoked at least once.
func PollUntil(interval time.Duration, condition ConditionFunc, stopCh <-chan struct{}) error {
   ctx, cancel := contextForChannel(stopCh)
   defer cancel()
   return WaitFor(poller(interval, 0), condition, ctx.Done())
}

// PollImmediateUntil tries a condition func until it returns true, an error or stopCh is closed.
//
// PollImmediateUntil runs the 'condition' before waiting for the interval.
// 'condition' will always be invoked at least once.
func PollImmediateUntil(interval time.Duration, condition ConditionFunc, stopCh <-chan struct{}) error {
   done, err := condition()
   if err != nil {
      return err
   }
   if done {
      return nil
   }
   select {
   case <-stopCh:
      return ErrWaitTimeout
   default:
      return PollUntil(interval, condition, stopCh)
   }
}
```

### 7.WaitFor：continually checks 'fn' as driven by 'wait'.

```go


// WaitFunc creates a channel that receives an item every time a test
// should be executed and is closed when the last test should be invoked.
type WaitFunc func(done <-chan struct{}) <-chan struct{}

// WaitFor continually checks 'fn' as driven by 'wait'.
//
// WaitFor gets a channel from 'wait()'', and then invokes 'fn' once for every value
// placed on the channel and once more when the channel is closed. If the channel is closed
// and 'fn' returns false without error, WaitFor returns ErrWaitTimeout.
//
// If 'fn' returns an error the loop ends and that error is returned. If
// 'fn' returns true the loop ends and nil is returned.
//
// ErrWaitTimeout will be returned if the 'done' channel is closed without fn ever
// returning true.
//
// When the done channel is closed, because the golang `select` statement is
// "uniform pseudo-random", the `fn` might still run one or multiple time,
// though eventually `WaitFor` will return.
func WaitFor(wait WaitFunc, fn ConditionFunc, done <-chan struct{}) error {
   stopCh := make(chan struct{})
   defer close(stopCh)
   c := wait(stopCh)
   for {
      select {
      case _, open := <-c:
         ok, err := fn()
         if err != nil {
            return err
         }
         if ok {
            return nil
         }
         if !open {
            return ErrWaitTimeout
         }
      case <-done:
         return ErrWaitTimeout
      }
   }
}

// poller returns a WaitFunc that will send to the channel every interval until
// timeout has elapsed and then closes the channel.
//
// Over very short intervals you may receive no ticks before the channel is
// closed. A timeout of 0 is interpreted as an infinity, and in such a case
// it would be the caller's responsibility to close the done channel.
// Failure to do so would result in a leaked goroutine.
//
// Output ticks are not buffered. If the channel is not ready to receive an
// item, the tick is skipped.
func poller(interval, timeout time.Duration) WaitFunc {
   return WaitFunc(func(done <-chan struct{}) <-chan struct{} {
      ch := make(chan struct{})

      go func() {
         defer close(ch)

         tick := time.NewTicker(interval)
         defer tick.Stop()

         var after <-chan time.Time
         if timeout != 0 {
            // time.After is more convenient, but it
            // potentially leaves timers around much longer
            // than necessary if we exit early.
            timer := time.NewTimer(timeout)
            after = timer.C
            defer timer.Stop()
         }

         for {
            select {
            case <-tick.C:
               // If the consumer isn't ready for this signal drop it and
               // check the other channels.
               select {
               case ch <- struct{}{}:
               default:
               }
            case <-after:
               return
            case <-done:
               return
            }
         }
      }()

      return ch
   })
}

// resetOrReuseTimer avoids allocating a new timer if one is already in use.
// Not safe for multiple threads.
func resetOrReuseTimer(t *time.Timer, d time.Duration, sawTimeout bool) *time.Timer {
   if t == nil {
      return time.NewTimer(d)
   }
   if !t.Stop() && !sawTimeout {
      <-t.C
   }
   t.Reset(d)
   return t
}
```



