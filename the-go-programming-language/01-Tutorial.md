## 1. Hello World

```go
package helloworld

import "fmt"

func helloWorld()  {
	fmt.Println("Hello, World!")
}
```



## 2. Command Line

```go
import (
	"fmt"
	"os"
)

// 0 一般是运行的脚本本身
func echo2() {
	s, sep := "", ""
	for _, arg := range os.Args[1:] {
		s += sep + arg
		sep = " "
	}
	fmt.Println(s)
}
```



## 3. File I/O

```go
import (
	"fmt"
	"io/ioutil"
	"os"
	"strings"
)

func dup3() {
	counts := make(map[string]int)
	for _, filename := range os.Args[4:] {
		data, err := ioutil.ReadFile(filename)
		if err != nil {
			fmt.Fprintf(os.Stderr, "dup3: %v\n", err)
			continue
		}
		for _, line := range strings.Split(string(data), "\n") {
			counts[line]++
		}
	}
	for line, n := range counts {
		if n > 1 {
			fmt.Printf("%d\t%s\n", n, line)
		}
	}
}
```



## 4. Cocurrently

```go
import (
	"fmt"
	"io"
	"io/ioutil"
	"net/http"
	"strings"
	"time"
)

func fetchAll() {
	start := time.Now()
	ch := make(chan string)
	data, _ := ioutil.ReadFile("urls.txt")
	for _, url := range strings.Split(string(data), "\n") {
		go fetch(url, ch) // start a goroutine
	}
	for range strings.Split(string(data), "\n") {
		fmt.Println(<-ch) // receive from channel ch
	}
	fmt.Printf("%.2fs elapsed\n", time.Since(start).Seconds())
}

func fetch(url string, ch chan<- string) {
	start := time.Now()
	resp, err := http.Get(url)
	if err != nil {
		ch <- fmt.Sprint(err) // send to channel ch
		return
	}

	nbytes, err := io.Copy(ioutil.Discard, resp.Body)
	resp.Body.Close() // don't leak resources
	if err != nil {
		ch <- fmt.Sprintf("while reading %s: %v", url, err)
		return
	}
	secs := time.Since(start).Seconds()
	ch <- fmt.Sprintf("%.2fs  %7d  %s", secs, nbytes, url)
}

```



## 5. Web Server

```go
import (
	"fmt"
	"log"
	"net/http"
)

func server3() {
	http.HandleFunc("/", handler)
	log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

//!+handler
// handler echoes the HTTP request.
func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "%s %s %s\n", r.Method, r.URL, r.Proto)
	for k, v := range r.Header {
		fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
	}
	fmt.Fprintf(w, "Host = %q\n", r.Host)
	fmt.Fprintf(w, "RemoteAddr = %q\n", r.RemoteAddr)
	if err := r.ParseForm(); err != nil {
		log.Print(err)
	}
	for k, v := range r.Form {
		fmt.Fprintf(w, "Form[%q] = %q\n", k, v)
	}
}
```



## 6. Format 输出

| 示例       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| %d         | Decimal integer                                              |
| %x, %o, %b | Integer in hexademical, octal, binary                        |
| %f, %g, %e | floating-point number: 3.141593 3.141592653589793 3.141593e+00 |
| %t         | Boolean                                                      |
| %c         | Rune                                                         |
| %s         | string                                                       |
| %q         | quot ed str ing "abc" or rune 'c'                            |
| %v         | any value in a natural format                                |
| %T         | type of any value                                            |
| %%         | literal percent sig n (no operand)                           |

