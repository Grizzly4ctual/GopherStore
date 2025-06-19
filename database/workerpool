package database

import (
	"container/list"
	"sync"
	"sync/atomic"
	"time"
)

// WorkerPool manages a fixed-size pool of goroutines to execute submitted tasks.
// It limits concurrency, queues excess tasks, and gracefully handles shutdowns.
type WorkerPool struct {
	maxWorkers   int               // Maximum concurrent workers allowed.
	taskQueue    chan func()       // Channel for incoming tasks.
	workerQueue  chan func()       // Channel for workers to fetch tasks.
	stoppedChan  chan struct{}     // Closed when the pool has fully stopped.
	stopSignal   chan struct{}     // Closed to signal all workers to halt.
	waitingQueue list.List         // FIFO queue for tasks waiting for a worker.
	stopLock     sync.Mutex        // Ensures mutual exclusion during shutdown.
	stopOnce     sync.Once         // Guarantees Stop logic runs only once.
	stopped      bool              // Indicates if the pool is shutting down.
	waiting      int32             // Atomic count of waiting tasks.
	wait         bool              // If true, waits for all queued tasks before stopping.
}

// idleTimeout defines how long a worker can be idle before being retired.
var idleTimeout time.Duration = 2 * time.Second

// NewPool initializes a WorkerPool with a maximum number of workers.
// If maxWorkers is less than 1, defaults to 1 for safety.
func NewPool(maxWorkers int) *WorkerPool {
	if maxWorkers < 1 {
		maxWorkers = 1
	}

	pool := &WorkerPool{
		maxWorkers:  maxWorkers,
		taskQueue:   make(chan func()),
		workerQueue: make(chan func()),
		stopSignal:  make(chan struct{}),
		stoppedChan: make(chan struct{}),
	}
	// Start the dispatch loop in a separate goroutine.
	go pool.dispatch()
	return pool
}

// Submit queues a task for asynchronous execution.
// If the pool is full, the task may be queued internally.
func (p *WorkerPool) Submit(task func()) {
	if task != nil {
		p.taskQueue <- task
	}
}

// SubmitWait submits a task and blocks until it completes.
// Useful for synchronous operations needing worker pool guarantees.
func (p *WorkerPool) SubmitWait(task func()) {
	if task == nil {
		return
	}
	doneChan := make(chan struct{})
	p.taskQueue <- func() {
		task()
		close(doneChan)
	}
	<-doneChan
}

// Stop initiates a graceful shutdown of the pool.
// If wait is true, waits for all queued tasks to finish before stopping.
func (p *WorkerPool) Stop() {
	p.stop(false)
}

// stop handles the shutdown sequence, ensuring only one shutdown occurs.
func (p *WorkerPool) stop(wait bool) {
	p.stopOnce.Do(func() {
		// Notify all goroutines to stop.
		close(p.stopSignal)
		p.stopLock.Lock()
		// Mark the pool as stopped to prevent further task submission.
		p.stopped = true
		p.stopLock.Unlock()
		p.wait = wait
		// Close the task queue to prevent new tasks.
		close(p.taskQueue)
	})
	<-p.stoppedChan // Wait until dispatch loop has exited.
}

// dispatch is the main scheduler for the pool.
// It assigns tasks to workers, manages the waiting queue, and handles worker lifecycle.
func (p *WorkerPool) dispatch() {
	defer close(p.stoppedChan)
	timeout := time.NewTimer(idleTimeout)
	var workerCount int
	var idle bool
	var wg sync.WaitGroup

Loop:
	for {
		// If there are waiting tasks, try to process them first.
		if p.waitingQueue.Len() != 0 {
			if !p.processWaitingQueue() {
				break Loop
			}
			continue
		}

		select {
		case task, ok := <-p.taskQueue:
			if !ok {
				break Loop // taskQueue closed, exit loop.
			}
			// Try to assign task to an idle worker.
			select {
			case p.workerQueue <- task:
			default:
				// No idle worker. If we can, start a new worker.
				if workerCount < p.maxWorkers {
					wg.Add(1)
					go worker(task, p.workerQueue, &wg)
					workerCount++
				} else {
					// All workers busy and at max capacity; queue the task.
					p.waitingQueue.PushBack(task)
					atomic.StoreInt32(&p.waiting, int32(p.waitingQueue.Len()))
				}
			}
			idle = false

		case <-timeout.C:
			// On timeout, retire an idle worker if possible.
			if idle && workerCount > 0 {
				if p.killIdleWorker() {
					workerCount--
				}
			}
			idle = true
			timeout.Reset(idleTimeout)
		}
	}
	if p.wait {
		p.runQueuedTasks()
	}
	// Send nil to all workers to signal shutdown.
	for workerCount > 0 {
		p.workerQueue <- nil
		workerCount--
	}
	wg.Wait()         // Wait for all workers to finish.
	timeout.Stop()    // Clean up timer.
}

// worker executes tasks as they are assigned via the workerQueue.
// Receives nil to signal shutdown.
func worker(task func(), workerQueue chan func(), wg *sync.WaitGroup) {
	for task != nil {
		task()
		task = <-workerQueue
	}
	wg.Done()
}

// killIdleWorker attempts to retire an idle worker by sending nil.
// Returns true if a worker was signaled to exit.
func (p *WorkerPool) killIdleWorker() bool {
	select {
	case p.workerQueue <- nil:
		return true
	default:
		return false // No idle workers available.
	}
}

// processWaitingQueue attempts to assign waiting tasks to available workers.
// Returns false if the pool is shutting down.
func (p *WorkerPool) processWaitingQueue() bool {
	select {
	case task, ok := <-p.taskQueue:
		if !ok {
			return false // Pool is stopping.
		}
		p.waitingQueue.PushBack(task)
	case p.workerQueue <- p.waitingQueue.Front().Value.(func()):
		// Assign task to worker and remove from queue.
		front := p.waitingQueue.Front()
		p.waitingQueue.Remove(front)
	}
	atomic.StoreInt32(&p.waiting, int32(p.waitingQueue.Len()))
	return true
}

// runQueuedTasks drains the waiting queue, assigning each task to a worker.
func (p *WorkerPool) runQueuedTasks() {
	for p.waitingQueue.Len() != 0 {
		front := p.waitingQueue.Front()
		p.workerQueue <- p.waitingQueue.Remove(front).(func())
		atomic.StoreInt32(&p.waiting, int32(p.waitingQueue.Len()))
	}
}
