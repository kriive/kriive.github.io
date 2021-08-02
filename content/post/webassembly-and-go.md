---
title: "WebAssembly and Go"
date: 2021-07-29T23:24:49+02:00
---
This semester in university I had one professor that distributed lectures and other learning material as PDF files.

> Sounds perfect! I can use them and take notes directly on my iPad!
> 
> -- <cite>Me</cite>

Nope, because these PDFs are not writable and protected by a password, rendering iPad note taking impossible.
I mean, why would anyone lock a PDF file of a lecture in the first place?

After a brief Google search, I learned that PDFs like mine can be unlocked by a high number of tools ([qpdf](https://github.com/qpdf/qpdf), [pdftk](https://www.pdflabs.com/tools/pdftk-the-pdf-toolkit/), ...) **without knowing the password**. 
However, two Google searches later, I discovered a wonderful PDF toolkit called [pdfcpu](https://github.com/pdfcpu/pdfcpu), which offers a vast number of tools and it's written in Go!

Why don't we write a Web Application that unlocks PDFs? And while we're at it, I'd like to try to WebAssemblify[^1] pdfcpu to do the heavy lifting.

![Here's a cat picture to lift your mood!](/cat.webp 'A cat to lift your mood')

This is the plan:
1. Write a Go function that takes a PDF in input, decrypts it with pdfcpu and returns the unencrypted PDF, and compile it to WebAssembly.
2. Embed it in a webpage, write the glue code in JS.
3. Profit!

As a plus, processing is done client-side, so people are happy and the GDPR police doesn't bite me. Sounds good?

# Step 1: write the Go function
## First things first, say hello to WebAssembly

Make a new directory and `go mod init github.com/<your-username>/pdf-wasm` it.
We will create two subdirectories and copy the Go WebAssembly runtime to `cmd/server/assets`.
The Go WebAssembly runtime will provide some functions to allow JS and Go to communicate and share memory.
```bash
# The -p flag creates any parent directory if needed
# See man mkdir for more options
mkdir -p cmd/wasm cmd/server/assets

# Install the Go WebAssembly runtime
cp "$(go env GOROOT)/misc/wasm/wasm_exec.js" ./cmd/server/assets/
```
Let's write some code!
Open a new file under `cmd/wasm/main.go` and type the following Hello, World!

```go
// cmd/wasm/main.go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello from Go WebAssembly!")
}
```
Now we're gonna build it! WebAssembly is just another target, if you're familiar with cross-compiling Go projects, WebAssembly isn't much different.

```bash
GOOS=js GOARCH=wasm go build -o ./cmd/server/assets/wasm/pdf.wasm cmd/wasm/main.go
```

Now, we are going to create the webpage that our browser will execute! Create a new file under `cmd/server/assets/index.html`.

This page loads the WebAssembly runtime script that we just copied to `cmd/server/assets`, then it will load and execute our Hello WebAssembly.
As suggested [in the Go WebAssembly Github Wiki](https://github.com/golang/go/wiki/WebAssembly#getting-started), we included a simple [polyfill](https://github.com/golang/go/blob/b2fcfc1a50fbd46556f7075f7f1fbf600b5c9e5d/misc/wasm/wasm_exec.html#L17-L22)[^polyfill] that makes the `WebAssembly.instantiateStreaming` function available even on browsers that do not support it natively.
```html
<-- cmd/server/assets/index.html -->
<html>

<head>
    <meta charset="utf-8" />
    <script src="wasm_exec.js"></script>
    <script>
        if (!WebAssembly.instantiateStreaming) { // polyfill
            WebAssembly.instantiateStreaming = async (resp, importObject) => {
                const source = await (await resp).arrayBuffer();
                return await WebAssembly.instantiate(source, importObject);
            };
        }
        const go = new Go();
        WebAssembly.instantiateStreaming(fetch("wasm/pdf.wasm"), go.importObject).then((result) => {
            go.run(result.instance);
        });
    </script>
</head>
</html>
```
The last bit that we need is a static server that will serve our files! I wanted to try the new [Go 1.16 embed feature](https://pkg.go.dev/embed), so I prepared a little server that leveraged it.
I'm not gonna dive into details here, but this server tries to serve files compressed[^compress]. We won't need this feature right away.
Create a new file under `cmd/server/main.go` and paste the following:

```go
// cmd/server/main.go
package main

import (
	"embed"
	"fmt"
	"io/fs"
	"log"
	"net/http"
	"path"
	"strings"

	"github.com/vearutop/statigz"
	"github.com/vearutop/statigz/brotli"
)

//go:embed assets
var st embed.FS

func main() {
	// Auto-index
	withIndexHTML := func(h http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if strings.HasSuffix(r.URL.Path, "/") || len(r.URL.Path) == 0 {
				newpath := path.Join(r.URL.Path, "index.html")
				r.URL.Path = newpath
			}
			h.ServeHTTP(w, r)
		})
	}

	// Retrieve sub directory.
	sub, err := fs.Sub(st, "assets")
	if err != nil {
		log.Fatal(err)
	}

	if err = http.ListenAndServe("0.0.0.0:9090", withIndexHTML(statigz.FileServer(sub.(fs.ReadDirFS), brotli.AddEncoding))); err != nil {
		fmt.Println("Failed to start server", err)
		return
	}
}
```
Run `go mod tidy`, it should pick up the dependencies and download 'em. Now we're ready for our Hello, World!

Run `go run ./cmd/server`, open your browser, go to [http://localhost:9090/](http://localhost:9090/ "Hello to you too!"), open your developer tools and select the console window.

![Hello, World! WebAssembly](/console_hello_world.webp "This is what you should see")

## Let's unlock them PDFs

Now that we learned how to compile and load a Go WebAssembly application, we need to actually write code that helps us towards our goal.
We need to write a function that takes an array of bytes as input (our PDF), decrypts it using pdfcpu and returns an array of bytes that represents the decrypted PDF.

We instruct pdfcpu not to load configurations from file, since it wouldn't make much sense in a WebAssembly environment.
I then copied what [pdfcpu authors did in their decrypt command](https://github.com/pdfcpu/pdfcpu/blob/3c08a45b5cf14f05aca9a253f351f68a78c8da56/pkg/api/crypto.go#L34-L42).

Open `cmd/wasm/main.go` and replace the old Hello, WebAssembly code with the following.
```go
// cmd/wasm/main.go
package main

import (
	"bytes"

	"github.com/pdfcpu/pdfcpu/pkg/api"
	"github.com/pdfcpu/pdfcpu/pkg/pdfcpu"
)

// DecryptPDF decrypts a PDF and returns the unencrypted file.
// It uses pdfcpu, it's a bit of a hack, because I don't know 
// about API stability of the package and this could break in a update.
// Who knows? 🤷 
func DecryptPDF(input []byte) ([]byte, error) {
	r := bytes.NewReader(input)
	var w bytes.Buffer

	// Workaround: avoid pdfcpu to complain for missing config
	pdfcpu.ConfigPath = "disable"

	conf := pdfcpu.NewDefaultConfiguration()
	conf.Cmd = pdfcpu.DECRYPT

	if err := api.Optimize(r, &w, conf); err != nil {
		return nil, err
	}

	return w.Bytes(), nil
}
```

Use `go mod tidy` and let the Go dependency system pick up the `pdfcpu` package.

Now we need some sort of interoperability with JS. Enter the package `syscall/js`!
Beware that this package is an [exempt from the Go compatibility promise](https://pkg.go.dev/syscall/js#pkg-overview) and it's highly experimental, 
so the used API can change at any point in time.

We need to write a JS wrapper that internally calls our `DecryptPDF` and glues JS and Go together, 
since we cannot call Go directly (at least, that's my understanding of it).

We will write a JS function that returns a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise), to allow an idiomatical approach to both the happy and sad paths.
Add the pdfWrapper function in `cmd/wasm/main.go`.

```go
// cmd/wasm/main.go
package main

import (
	"bytes"
	"syscall/js"

	"github.com/pdfcpu/pdfcpu/pkg/api"
	"github.com/pdfcpu/pdfcpu/pkg/pdfcpu"
)

func DecryptPDF(input []byte) ([]byte, error) {
	// ...
}

// Add this function!
func pdfWrapper() js.Func {
	return js.FuncOf(func(this js.Value, args []js.Value) interface{} {
		if len(args) != 1 {
			return "Invalid no of arguments passed"
		}

		// Outer function has one argument, which is the input locked PDF file 
		pdfBuf := args[0]

		handler := js.FuncOf(func(this js.Value, args []js.Value) interface{} {
			// This handler function will be wrapped inside a Promise, see below
			// The first argument is a function called if everything's ok,
			// the second one is a function called when something's gone wrong
			resolve := args[0]
			reject := args[1]

			// Execute the decrypt function in a separate goroutine
			go func() {
				inBuf := make([]byte, pdfBuf.Get("byteLength").Int())

				// Copy the pdfBuf (containing the locked PDF as an array)
				// to the Go WebAssembly runtime
				js.CopyBytesToGo(inBuf, pdfBuf)

				// Do our little magic trick and call our decryption function
				output, err := DecryptPDF(inBuf)
				if err != nil {
					// Create a new Error wrapping the error message
					// returned by DecryptPDF
					errorConstructor := js.Global().Get("Error")
					errorObject := errorConstructor.New(err.Error())

					// Invoke the sad path, calling the reject function
					reject.Invoke(errorObject)
				} else {
					// Create a new Uint8Array with the proper length
					dst := js.Global().Get("Uint8Array").New(len(output))

					// Copy the decrypted output from Go runtime to JS
					js.CopyBytesToJS(dst, output)

					// We're done, call the happy path
					resolve.Invoke(dst)
				}
			}()

			return nil
		})

		// Create the Promise
		promiseConstructor := js.Global().Get("Promise")

		// https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
		// A new Promise can be created from a function with two parameters
		// In our case: resolve (happy path) and reject (sad path, an error occurred)
		return promiseConstructor.New(handler)
	})
}
```
We're pretty close to having a functioning WebAssembly application. We need two other things: a main function and a webpage that contains it all.
The main function is pretty straightforward: we need to signal the JS runtime that the JS function `decryptPDF()` internally executes `pdfWrapper()`. Then we need a channel that listens indefinitely to avoid closing the WebAssembly application too soon, or else the exported Go functions will become unavailable.

```go
import (
	"bytes"
	"fmt"
	"syscall/js"

	"github.com/pdfcpu/pdfcpu/pkg/api"
	"github.com/pdfcpu/pdfcpu/pkg/pdfcpu"
)

func DecryptPDF(input []byte) ([]byte, error) {
	// ...
}

func pdfWrapper() js.Func {
	// ...
}

func main() {
	fmt.Println("Hello from Go WebAssembly!")
	js.Global().Set("decryptPDF", pdfWrapper())
	<-make(chan bool)
}
```

# Step 2, write a (ugly) frontend!
We're set, we have every piece in place, we only need to glue them together in a HTML page!
I apologize in advance because my frontend skills are even worse than my backend's, so have a little mercy.

We're going to add a couple of components in our main HTML page and a script that handles the file IO and boom, we're done!
When our processing's done, the Download link will colorize and we can download the decrypted file.

So, let's edit our `./cmd/server/assets/index.html`.
Before the closing `</html>` tag add the following:
```html
<-- ./cmd/server/assets/index.html -->
<html>
<head>
...
</head>
<body>
    <input type="file" accept=".pdf" />
    <div id="result"><a id="link" target="_blank" download="file.txt">Download</a></div>
</body>
<script>
    document.querySelector('input').addEventListener('change', async (event) => {
        const buffer = await event.target.files[0].arrayBuffer();

        // Add a suffix to the soon-to-be-downloaded file
        const name = event.target.files[0].name ?
            event.target.files[0].name.replace(new RegExp(".pdf" + '$'), '-decrypted.pdf') :
            "file.pdf";

        var file = new Uint8Array(buffer);

        var data = [];
        try {
            data.push(await decryptPDF(file));
        } catch (err) {
            console.error('caught error from WASM:', err);
            return;
        }

        var properties = { type: 'application/pdf' }; // Specify the file's mime-type.
        try {
            // Specify the filename using the File constructor, but ...
            const nameInDownload = name;
            file = new File(data, name, properties);
        } catch (e) {
            // ... fall back to the Blob constructor if that isn't supported.
            file = new Blob(data, properties);
        }

        var url = URL.createObjectURL(file);
        document.getElementById('link').href = url;
	// Change the download name to our *-decrypted.pdf name
        document.getElementById('link').download = name;
    }, false)
</script>
</html>
```

# Step 3: profit!
Recompile the WASM (this is important!) and re-run the server.
```sh
GOOS=js GOARCH=wasm go build -o ./cmd/server/assets/wasm/pdf.wasm cmd/wasm/main.go
// and run the server!
go run ./cmd/server
```

Go to [http://localhost:9090](http://localhost:9090/) and we're done!

## Conclusions
We learned how to make Go and JS interact and how to pwn (sort of) write-locked PDFs without knowing the password.
Here's the complete code packed in a single repository: [kriive/decrypt-wasm](https://github.com/kriive/decrypt-wasm).

If you wanna use the web app: [https://kriive.github.io/decrypt-wasm](https://kriive.github.io/decrypt-wasm).

Feel free to fork the project and (hopefully) implement a better frontend! PRs welcome!

[^1]: *WebAssembly*: a technology that allows you to execute some binaries on a virtual machine inside your web page. It's a standard and it's implemented by [almost every major browser](https://developer.mozilla.org/en-US/docs/WebAssembly#browser_compatibility). Even Safari on iOS!

[^polyfill]: *Polyfill*: code that implements a feature on web browsers that do not support the feature [(Wikipedia definition)](https://en.wikipedia.org/wiki/Polyfill_(programming)).

[^compress]: We will need to manually generate the compressed files, the package searches for [brotli](https://en.wikipedia.org/wiki/Brotli) or [gzipped](https://en.wikipedia.org/wiki/Gzip) compressed files.
