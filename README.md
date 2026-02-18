# Go Code Challenge Quick Reference

---

## **Priority Queue (Min-Heap)**
```go
type Item struct{ val, priority int }
type PQ []Item

func (h PQ) Len() int            { return len(h) }
func (h PQ) Less(i, j int) bool  { return h[i].priority < h[j].priority } // flip for max-heap
func (h PQ) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *PQ) Push(x any)         { *h = append(*h, x.(Item)) }
func (h *PQ) Pop() any           { old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x }

h := &PQ{}
heap.Init(h)
heap.Push(h, Item{val: 42, priority: 3})
top := heap.Pop(h).(Item)
```

---

## **Queue (slice + head index)**
```go
type Queue[T any] struct {
    data []T
    head int
}

func (q *Queue[T]) Enqueue(v T) { q.data = append(q.data, v) }
func (q *Queue[T]) Dequeue() (T, bool) {
    if q.head >= len(q.data) { var zero T; return zero, false }
    v := q.data[q.head]; q.head++
    if q.head > len(q.data)/2 { q.data = q.data[q.head:]; q.head = 0 } // compact
    return v, true
}
func (q *Queue[T]) Len() int { return len(q.data) - q.head }
```

---

## **Stack**
```go
stack := []int{}
stack = append(stack, v)          // push
v, stack = stack[len(stack)-1], stack[:len(stack)-1] // pop — guard len>0 in real code
```

---

## **BFS**
```go
// Note: queue = queue[1:] never releases head memory; fine for code challenges,
// use the Queue struct above if the graph is huge.
func bfs(graph map[int][]int, start int) {
    visited := map[int]bool{start: true}
    queue := []int{start}
    for len(queue) > 0 {
        node := queue[0]; queue = queue[1:]
        for _, next := range graph[node] {
            if !visited[next] {
                visited[next] = true
                queue = append(queue, next)
            }
        }
    }
}
```

## **DFS (iterative)**
```go
func dfs(graph map[int][]int, start int) {
    visited := map[int]bool{}
    stack := []int{start}
    for len(stack) > 0 {
        node := stack[len(stack)-1]; stack = stack[:len(stack)-1]
        if visited[node] { continue }
        visited[node] = true
        for _, next := range graph[node] {
            stack = append(stack, next)
        }
    }
}
```

## **Cycle Detection (directed graph)**
```go
// 0=unvisited  1=in current path  2=fully processed
func hasCycle(graph map[int][]int) bool {
    state := map[int]int{}
    var dfs func(n int) bool
    dfs = func(n int) bool {
        state[n] = 1
        for _, next := range graph[n] {
            if state[next] == 1 { return true }            // back edge → cycle
            if state[next] == 0 && dfs(next) { return true }
        }
        state[n] = 2
        return false
    }
    for n := range graph {
        if state[n] == 0 && dfs(n) { return true }
    }
    return false
}
// Undirected: same idea, but pass parent and skip it; cycle if you reach a visited non-parent node.
```

---

## **Union-Find**
```go
type UF struct{ parent, rank []int }

func NewUF(n int) *UF {
    p := make([]int, n)
    for i := range p { p[i] = i }
    return &UF{p, make([]int, n)}
}
func (u *UF) Find(x int) int {
    if u.parent[x] != x { u.parent[x] = u.Find(u.parent[x]) } // path compression
    return u.parent[x]
}
func (u *UF) Union(x, y int) bool {
    px, py := u.Find(x), u.Find(y)
    if px == py { return false } // already connected
    if u.rank[px] < u.rank[py] { px, py = py, px }
    u.parent[py] = px
    if u.rank[px] == u.rank[py] { u.rank[px]++ }
    return true
}
```

---

## **LRU Cache**
```go
type LRU struct {
    cap  int
    data map[int]*list.Element
    list *list.List // front = most recent
}
type kv struct{ k, v int }

func NewLRU(cap int) *LRU { return &LRU{cap, map[int]*list.Element{}, list.New()} }

func (c *LRU) Get(k int) (int, bool) {
    if el, ok := c.data[k]; ok {
        c.list.MoveToFront(el)
        return el.Value.(*kv).v, true
    }
    return 0, false
}
func (c *LRU) Put(k, v int) {
    if el, ok := c.data[k]; ok {
        el.Value.(*kv).v = v; c.list.MoveToFront(el); return
    }
    if c.list.Len() == c.cap {
        back := c.list.Back(); c.list.Remove(back)
        delete(c.data, back.Value.(*kv).k)
    }
    c.data[k] = c.list.PushFront(&kv{k, v})
}
```

---

## **Mutex — Safe Map**
```go
type SafeMap struct {
    mu   sync.Mutex
    data map[string]int
}

func (m *SafeMap) Inc(key string) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.data[key]++
}
func (m *SafeMap) Get(key string) int {
    m.mu.Lock()
    defer m.mu.Unlock()
    return m.data[key]
}
// Use sync.RWMutex and RLock/RUnlock when reads dominate.
```

---

## **Producer-Consumer (fan-out)**
```go
func produce(ch chan<- int, n int) {
    for i := range n { ch <- i }
    close(ch)
}
func consume(ch <-chan int, wg *sync.WaitGroup) {
    defer wg.Done()
    for v := range ch { _ = v /* process */ }
}

ch := make(chan int, 32)
var wg sync.WaitGroup
go produce(ch, 1000)
for range 4 { wg.Add(1); go consume(ch, &wg) }
wg.Wait()
```

---

## **Ordered Concurrent Results**
```go
// Each goroutine writes to its own index — no mutex needed.
results := make([]Result, len(jobs))
var wg sync.WaitGroup
for i, job := range jobs {
    wg.Add(1)
    go func() { // Go 1.22+: loop vars are per-iteration, no need to capture
        defer wg.Done()
        results[i] = process(job)
    }()
}
wg.Wait()
```

---

## **Semaphore (bounded concurrency)**
```go
const maxConcurrent = 8
sem := make(chan struct{}, maxConcurrent)

for _, job := range jobs {
    sem <- struct{}{}
    go func() {
        defer func() { <-sem }()
        process(job)
    }()
}
for range maxConcurrent { sem <- struct{}{} } // drain = wait for all
```

---

## **Select: Timeout / Cancellation**
```go
select {
case v := <-dataCh:
    handle(v)
case <-time.After(5 * time.Second):
    // timed out — OK for one-shot selects
    // in a loop, use time.NewTimer or time.NewTicker to avoid timer leaks
case <-ctx.Done():
    // cancelled/deadline exceeded
}
```

---

## **sync.Once (singleton)**
```go
var (
    once     sync.Once
    instance *Service
)
func GetService() *Service {
    once.Do(func() { instance = &Service{} })
    return instance
}
```

---

## **errgroup — Concurrent Tasks with Error Propagation**
```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]string, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]string, len(urls))

    for i, url := range urls {
        g.Go(func() error {
            req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
            if err != nil { return err }
            resp, err := http.DefaultClient.Do(req)
            if err != nil { return err }
            defer resp.Body.Close()
            body, err := io.ReadAll(resp.Body)
            results[i] = string(body)
            return err
        })
    }
    if err := g.Wait(); err != nil { return nil, err }
    return results, nil
}
// First error cancels ctx → other goroutines see ctx.Done() and can bail out.
// g.SetLimit(n) caps concurrency (replaces manual semaphore).
```

---

## **Pipeline (channel chaining)**
```go
func generate(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done(): return
            }
        }
    }()
    return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-ctx.Done(): return
            }
        }
    }()
    return out
}

// Usage:
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
for v := range square(ctx, generate(ctx, 1, 2, 3, 4)) {
    fmt.Println(v)
}
```

---

## **Fan-In (merge multiple channels)**
```go
func merge[T any](ctx context.Context, channels ...<-chan T) <-chan T {
    out := make(chan T)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for v := range ch {
                select {
                case out <- v:
                case <-ctx.Done(): return
                }
            }
        }()
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

---

## **Rate Limiter (token bucket)**
```go
// stdlib rate limiter — simple and production-grade
import "golang.org/x/time/rate"

limiter := rate.NewLimiter(rate.Every(100*time.Millisecond), 10) // 10 rps, burst 10

func handleRequest(ctx context.Context) error {
    if err := limiter.Wait(ctx); err != nil { return err } // blocks until token available
    return doWork()
}

// Or non-blocking:
if !limiter.Allow() { return ErrRateLimited }
```

---

## **Context Patterns**
```go
// Timeout — auto-cancels after duration
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel() // always defer cancel to release resources

// Passing values (use sparingly — prefer function args)
type ctxKey string
ctx = context.WithValue(ctx, ctxKey("requestID"), "abc-123")
rid := ctx.Value(ctxKey("requestID")).(string)

// Checking cancellation in a long loop
for i := range hugeSlice {
    select {
    case <-ctx.Done():
        return ctx.Err() // context.Canceled or context.DeadlineExceeded
    default:
    }
    process(hugeSlice[i])
}
```

---

## **sync.Map (high-contention, many keys, rare writes)**
```go
var m sync.Map
m.Store("key", 42)
if v, ok := m.Load("key"); ok { fmt.Println(v.(int)) }

m.LoadOrStore("key2", 99) // atomic get-or-set
m.Range(func(k, v any) bool {
    fmt.Println(k, v)
    return true // return false to stop
})
// Prefer SafeMap with Mutex for most cases; sync.Map wins when:
//   • keys are stable (few deletes/adds)
//   • goroutines access mostly disjoint key sets
```

---

## **Atomic Operations**
```go
var counter atomic.Int64
counter.Add(1)         // thread-safe increment
v := counter.Load()    // read

var flag atomic.Bool
flag.Store(true)
if flag.Load() { /* ... */ }

// atomic.Value for copy-on-write config reloads
var config atomic.Value // stores *Config
config.Store(&Config{Timeout: 5 * time.Second})
cfg := config.Load().(*Config)
```

---

## **Mini Redis (TTL store)**
```go
type Redis struct {
    mu   sync.Mutex
    data map[string]entry
}
type entry struct {
    val     string
    expires time.Time // zero value = no expiry
}

func (r *Redis) Set(k, v string, ttl time.Duration) {
    r.mu.Lock(); defer r.mu.Unlock()
    exp := time.Time{}
    if ttl > 0 { exp = time.Now().Add(ttl) }
    r.data[k] = entry{v, exp}
}
func (r *Redis) Get(k string) (string, bool) {
    r.mu.Lock(); defer r.mu.Unlock()
    e, ok := r.data[k]
    if !ok { return "", false }
    if !e.expires.IsZero() && time.Now().After(e.expires) {
        delete(r.data, k); return "", false
    }
    return e.val, true
}
func (r *Redis) Del(k string) { r.mu.Lock(); defer r.mu.Unlock(); delete(r.data, k) }

// Background eviction loop — use Ticker, NOT time.After in a loop (leaks timers).
func (r *Redis) StartEviction(ctx context.Context, interval time.Duration) {
    go func() {
        ticker := time.NewTicker(interval)
        defer ticker.Stop()
        for {
            select {
            case <-ctx.Done(): return
            case <-ticker.C:
                now := time.Now()
                r.mu.Lock()
                for k, e := range r.data {
                    if !e.expires.IsZero() && now.After(e.expires) { delete(r.data, k) }
                }
                r.mu.Unlock()
            }
        }
    }()
}
```

---

## **Binary Search (stdlib)**
```go
// Returns (index where target would be, found)
idx, found := slices.BinarySearch(sorted, target)

// Custom comparator (e.g., struct field):
idx, found = slices.BinarySearchFunc(items, targetKey, func(a Item, k int) int {
    return cmp.Compare(a.key, k)
})
```

---

## **Sliding Window (fixed size)**
```go
func maxSum(nums []int, k int) int {
    sum := 0
    for _, v := range nums[:k] { sum += v }
    best := sum
    for i := k; i < len(nums); i++ {
        sum += nums[i] - nums[i-k]
        best = max(best, sum)
    }
    return best
}
```

## **Sliding Window (variable — longest substring without repeat)**
```go
func lengthOfLongestSubstring(s string) int {
    last := map[byte]int{}
    best, left := 0, 0
    for right := range len(s) {
        if i, ok := last[s[right]]; ok && i >= left { left = i + 1 }
        last[s[right]] = right
        best = max(best, right-left+1)
    }
    return best
}
```

---

## **Two Pointers (sorted two-sum)**
```go
func twoSum(nums []int, target int) (int, int) {
    l, r := 0, len(nums)-1
    for l < r {
        switch s := nums[l] + nums[r]; {
        case s == target: return l, r
        case s < target:  l++
        default:          r--
        }
    }
    return -1, -1
}
```

---

## **Trie**
```go
type Trie struct {
    children [26]*Trie
    end      bool
}
func (t *Trie) Insert(word string) {
    cur := t
    for _, c := range word {
        i := c - 'a'
        if cur.children[i] == nil { cur.children[i] = &Trie{} }
        cur = cur.children[i]
    }
    cur.end = true
}
func (t *Trie) Search(word string) bool {
    cur := t
    for _, c := range word {
        i := c - 'a'
        if cur.children[i] == nil { return false }
        cur = cur.children[i]
    }
    return cur.end
}
// StartsWith: same as Search but return true at the end regardless of cur.end
```

---

## **Simple Web Server (static files + API)**
```go
func main() {
    mux := http.NewServeMux()

    // Serve static HTML/JS/CSS from ./static directory
    mux.Handle("GET /static/", http.StripPrefix("/static/",
        http.FileServer(http.Dir("static"))))

    // JSON API endpoint
    mux.HandleFunc("GET /api/health", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
    })

    // Path parameter (Go 1.22+ enhanced routing)
    mux.HandleFunc("GET /api/users/{id}", func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")
        fmt.Fprintf(w, "user %s", id)
    })

    // Middleware pattern
    handler := logMiddleware(mux)

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      handler,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
    }
    log.Fatal(srv.ListenAndServe())
}

func logMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}
```

---

## **Graceful Shutdown**
```go
func main() {
    srv := &http.Server{Addr: ":8080", Handler: mux}

    go func() { log.Fatal(srv.ListenAndServe()) }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Println("shutting down...")

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("forced shutdown:", err)
    }
}
```

---

## **Stdlib Tips & Patterns**

### Parsing
```go
// strings.Cut — cleaner than Split for key=value, header parsing
key, value, ok := strings.Cut("host:8080", ":")  // "host", "8080", true

// strconv essentials
n, err := strconv.Atoi("42")
s := strconv.Itoa(42)
f, err := strconv.ParseFloat("3.14", 64)

// bufio.Scanner — line-by-line or custom split
scanner := bufio.NewScanner(reader)
scanner.Split(bufio.ScanWords) // or ScanLines (default), ScanBytes
for scanner.Scan() {
    word := scanner.Text()
}

// Quick JSON marshal/unmarshal
data, _ := json.Marshal(myStruct)
json.Unmarshal(data, &myStruct)
// For streams: json.NewDecoder(r).Decode(&v) / json.NewEncoder(w).Encode(v)

// fmt.Sscanf for quick structured parsing
var x, y int
fmt.Sscanf("point 3 7", "point %d %d", &x, &y)
```
