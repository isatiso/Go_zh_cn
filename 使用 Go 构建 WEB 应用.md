# 起步

现在，你需要一个 FreeBSD，Linux，OS X，或者 Windows 机器来运行 Go。我们会使用 `$` 来表示命令提示符。

安装 Go 的部分请参见 [Installation Instructions](https://go-zh.org/doc/install)。

在你的 GOPATH 目录中创建一个新的文件夹，并且 `cd` 进入这个文件夹：

```bash
$ mkdir gowiki
$ cd gowiki
```

创建一个叫做 `wiki.go` 的文件，用你的编辑器打开它，然后添加下面的内容：

```go
package main

import (
	"fmt"
	"io/ioutil"
)
```

我们已经从 Go 标准库中加载了 `"fmt"` 和 `"ioutil"` 包。稍后，我们实现其他功能的时候，会加载更多的包。

# 数据结构

下面我们开始定义数据结构。一个 wiki 一系列互相联系的页面组成，每一页有一个 title（标题）和一个 body （页面内容）组成。这里，我们定义 `Page` 结构体，他有两个字段分别代表 title 和 body。

```go
type Page struct {
    Title string
    Body  []byte
}
```

类型 `[]byte` 意思是一个 `byte` 切片。（查看切片部分: [usage and internals](https://go-zh.org/doc/articles/slices_usage_and_internals.html) 以获取更多细节）`Body` 元素是一个 `[]byte` 而不是 `string` 是因为 `io` 库期望的类型是这样，下面你就会看到。

`Page`结构体描述了页面数据在内存中的存储方式。但是永久存储呢？我们可以通过一个 `save` 方法来指定页面的地址：

```go
func (p *Page) save() error {
    filename := p.Title + ".txt"
    return ioutil.WriteFile(filename, p.Body, 0600)
}
```

这个方法实现了这样的功能：这个方法的名字叫做 `save`，作为接受者的 `p` 是一个 `Page` 的指针。它不需要参数，并且返回一个 `error` 类型的值。

这个方法会将 `Page` 的 `body` 存储为一个文本文件。简单起见，我们会用 `Title` 作为文件名。

`save` 方法返回一个 `error` 值是因为这个就是 `WriteFile`（一个将 `byte` 切片写入文件的标准库函数）的返回值类型。`save` 方法返回 `error` 值，这样就可以在写文件出现任何错误时处理它。如果没出任何问题，`Page.save()` 会返回 `nil`（指针，接口，以及一些其他类型的零值）。

八进制整形字面量 `0600`，作为第三个参数传递给 `WriteFile`，意味着将会创建一个只供当前用户拥有读写权限的文件。（查看 Unix 手册 `open(2)` 获取更多细节）

除了要保存页面之外，我们还需要加载页面：

```go
func loadPage(title string) *Page {
    filename := title + ".txt"
    body, _ := ioutil.ReadFile(filename)
    return &Page{Title: title, Body: body}
}
```

函数 `loadPage` 通过 `title` 参数构造文件名，读取文件内容，并存入一个新的变量 `body` 中，然后返回一个指向用对应的 `title` 和 `body` 构造的 `Page` 字面量的指针。

函数可以返回多个值，标准库函数 `io.ReadFile` 返回 `[]byte` 和 `error`。在 `loadPage` 中，错误没有进行处理；（`_`）表示空白标识符，它用来将不需要的 错误值丢弃（in essence, assigning the value to nothing）。

但是如果 `ReadFile` 遇到了错误呢？比如说，文件不存在。我们不应该忽视这类错误，所以让我们修改这个函数，让他返回 `*Page` 和 `error`。

```go
func loadPage(title string) (*Page, error) {
    filename := title + ".txt"
    body, err := ioutil.ReadFile(filename)
    if err != nil {
        return nil, err
    }
    return &Page{Title: title, Body: body}, nil
}
```

这个函数的调用者现在检查了第二个参数；如果它是 `nil` ，那么就说明成功加载了一个 `Page` 。否则它会是一个 `error` ，从而让调用者进行处理（查看 [language specification](https://go-zh.org/ref/spec#Errors) 获取更多细节)。

现在我们有了一个简单的数据结构可以用来存储和加载文件。下面我们来写一个 `main` 函数来测试我们刚刚写的数据结构：

```go
func main() {
    p1 := &Page{Title: "TestPage", Body: []byte("This is a sample Page.")}
    p1.save()
    p2, _ := loadPage("TestPage")
    fmt.Println(string(p2.Body))
}
```
编译执行这段代码之后，一个叫做 `TestPage.txt` 的文件被创建出来，包含了 `p1`  的内容。之后，这个文件的内容会被读取到结构体 `p2` 中，并且它的 `Body` 元素会被打印在屏幕上。

你可以像下面这样编译和执行程序：

```bash
$ go build wiki.go
$ ./wiki
This is a sample page.
```

如果你使用 Windows 系統，你需要使用 `wiki` 不带 `./` 来运行程序。

[点击这里查看目前为止所写的代码](https://go-zh.org/doc/articles/wiki/part1.go)

# 介绍 `net/http` 

这是一个 WEB 服务器的最小化实现：


```go
package main

import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```

`main` 函数调用 `http.HandleFunc`，告诉 `http` 包的 `handler` 处理所有对根目录 `"/"` 的请求。

之后又调用了 `http.ListenAndServe`，指定它在所有接口上应该监听 8080 端口（`":8080"`）。（目前不用担心它的第二个参数是 `nil` ）这个函数会一直阻塞到程序终止。

函数 `handler` 的类型是 `http.HandlerFunc`。他需要 `http.ResponseWriter` 和一个 `http.Request` 作为它的参数。

通过向 `http.ResponseWriter` 写入数据构造 HTTP 服务的响应，并发送给客户端。

 `http.Request` 用来表示一个客户端请求的一个数据结构。`r.URL.Path` 是请求 URL 的路径部分。后面的 `[1:]` 意思是创建一个 `Path` 的切片（从第一个字符一直到最后一个）。这个操作也就是去掉了路径开头的 `"/"`。

如果你运行这个程序，并且接受到下面的 URL：

```bash
http://localhost:8080/monkeys
```

程序就会返回一个包含下列信息的页面：

```bash
Hi there, I love monkeys!
```



# 使用 `net/http` 托管 wiki 页面

使用 `net/http` 包就需要先加载它：

```go
import (
	"fmt"
	"io/ioutil"
	"net/http"
)
```

下面创建一个 `handler`，`viewHandler` 会允许用户查看一个 wiki 页面。它会处理 URL 的前缀，`"/view/"。

```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, _ := loadPage(title)
    fmt.Fprintf(w, "<h1>%s</h1><div>%s</div>", p.Title, p.Body)
}
```

首先，这个函数通过 `r.URL.Path` 提取页面标题。`Path` 已经通过 `[len("/view/"):]` 重新切片，来去掉请求路径前缀的`"/view/"` 部分。这就是为什么路径总是以 `"/view/"` 开头，但它却不是页面标题的一部分。

这个函数之后又加载了页面数据，并将页面格式化为一个简单的 HTML 字符串，写入 `w` （`http.ResponseWriter`）中。

再说一遍，要注意使用 `_` 来忽略 `loadPage` 返回的 `error` 值。这是为了简化起见，并不是一个好的做法，稍后我们会继续关注这里。

想要使用这个 handler，我们重写了 `main` 函数，使用 `viewHandler` 初始化 `http` ，处理任何在 `/view/` 路径下的请求。

```go
func main() {
    http.HandleFunc("/view/", viewHandler)
    http.ListenAndServe(":8080", nil)
}
```

[点击这里查看目前为止我们写的代码](https://go-zh.org/doc/articles/wiki/part2.go)。

下面我们创建一些页面数据（比如 `test.txt` ），编译我们的代码，然后尝试伺服一个 wiki 页面。

在你的编辑器中打开 `test.txt` 文件，写入 `Hello world` 并保存。

```bash
$ go build wiki.go
$ ./wiki
```

如果你使用 Windows 系統，你需要使用 `wiki` 不带 `./` 来运行程序。

这个服务运行起来后，访问 http://localhost:8080/view/test 会看到一个标题为 `"test"` 的页面，内容是 `"Hello world"`。



# 编辑页面

一个不能编辑的 wiki 不是一个真 wiki。下面我们就创建两个新的处理器：一个叫 `editHandler` ，用来显示一个编辑页面表单，另一个叫做 ``saveHandler` ` 用来保存通过表单提交的数据。

首先我们来将它们添加到 `main()` ：

```go
func main() {
    http.HandleFunc("/view/", viewHandler)
    http.HandleFunc("/edit/", editHandler)
    http.HandleFunc("/save/", saveHandler)
    http.ListenAndServe(":8080", nil)
}
```

函数 `editHandler` 加载了页面（或者，如果不存在的话，就创建一个空的 `Page` 结构体），然后显示一个 HTML 表单。

```go
func editHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/edit/"):]
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    fmt.Fprintf(w, "<h1>Editing %s</h1>"+
        "<form action=\"/save/%s\" method=\"POST\">"+
        "<textarea name=\"body\">%s</textarea><br>"+
        "<input type=\"submit\" value=\"Save\">"+
        "</form>",
        p.Title, p.Title, p.Body)
}
```

这个函数可以正常运行，但是所有 HTML 都是硬编码的，这太丑了。我们当然还有更好的方法。

# 内置包 `html/template` 

`html/template` 包是 Go 标准库的一部分。可以用它来将 HTML 作为一个分离的文件，允许更改 HTML 的同时不影响 Go 代码。

首先，我们必须将 `html/template` 添加到加载列表中。后面就不会再用 `fmt` 了，所以我们要移除它。

```go
import (
	"html/template"
	"io/ioutil"
	"net/http"
)
```

下面我们来创建一个包含 HTML 表单的模板文件。打开一个新文件，并命名为 `edit.html`，向其中添加下列内容：

```HTML
<h1>Editing {{.Title}}</h1>

<form action="/save/{{.Title}}" method="POST">
<div><textarea name="body" rows="20" cols="80">{{printf "%s" .Body}}</textarea></div>
<div><input type="submit" value="Save"></div>
</form>
```

修改 `editHandler` 来使用模板，取代硬编码的 HTML：

```go
func editHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/edit/"):]
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    t, _ := template.ParseFiles("edit.html")
    t.Execute(w, p)
}
```

函数 `template.ParseFiles` 会读取 `edit.html` 文件，并返回一个 `*template.Template `。

方法 `t.Execute` 会运行模板，将生成的 HTML 写入 `http.RespnseWriter`。`.Title` 和 `.Body` 这样的点号标识符，就是引用了 `p.Title` 和 `p.Body`。

模板指令通过双大括号包围。`printf "%s" .Body` 指令是一个函数调用，将 `.Body` 作为一个字符串，而不是字节流输出，调用 `fmt.Printf` 也是一样的效果。`html/template` 包保证了只有安全，美观的 HTML可以执行模板渲染。比如，它会自动转移大于符号 （`>`），替换为 `&gt;`，以此保证用户数据不会破坏 HTML 文档。

由于我们现在使用了模板工具，下面我们来创建一个 `viewHandler` 中使用的模板 `view.html`：

```HTML
<h1>{{.Title}}</h1>

<p>[<a href="/edit/{{.Title}}">edit</a>]</p>

<div>{{printf "%s" .Body}}</div>
```

相应的修改 `viewHandler` ：

```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, _ := loadPage(title)
    t, _ := template.ParseFiles("view.html")
    t.Execute(w, p)
}
```

需要注意的是，我们在两个处理器中使用了几乎相同的模板代码。下面我们将模板代码提取到单独的函数中来删除重复代码：

```go
func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
    t, _ := template.ParseFiles(tmpl + ".html")
    t.Execute(w, p)
}
```

然后修改处理器来使用这个函数：

```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, _ := loadPage(title)
    renderTemplate(w, "view", p)
}
```

```go
func editHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/edit/"):]
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    renderTemplate(w, "edit", p)
}
```


如果我们注释掉我们在 `main` 中未实现的 `save` 处理程序的注册，我们可以再次构建和测试我们的程序。

[点击这里查看我们到现在为止写的代码](https://go-zh.org/doc/articles/wiki/part3.go).



# 处理不存在的页面

What if you visit /view/APageThatDoesntExist? You'll see a page containing HTML. This is because it ignores the error return value from loadPage and continues to try and fill out the template with no data. Instead, if the requested Page doesn't exist, it should redirect the client to the edit Page so the content may be created:

func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, err := loadPage(title)
    if err != nil {
        http.Redirect(w, r, "/edit/"+title, http.StatusFound)
        return
    }
    renderTemplate(w, "view", p)
}
The http.Redirect function adds an HTTP status code of http.StatusFound (302) and a Location header to the HTTP response.

Saving Pages
The function saveHandler will handle the submission of forms located on the edit pages. After uncommenting the related line in main, let's implement the handler:

func saveHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/save/"):]
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    p.save()
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
The page title (provided in the URL) and the form's only field, Body, are stored in a new Page. The save() method is then called to write the data to a file, and the client is redirected to the /view/ page.

The value returned by FormValue is of type string. We must convert that value to []byte before it will fit into the Page struct. We use []byte(body) to perform the conversion.

Error handling
There are several places in our program where errors are being ignored. This is bad practice, not least because when an error does occur the program will have unintended behavior. A better solution is to handle the errors and return an error message to the user. That way if something does go wrong, the server will function exactly how we want and the user can be notified.

First, let's handle the errors in renderTemplate:

func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
    t, err := template.ParseFiles(tmpl + ".html")
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    err = t.Execute(w, p)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
}
The http.Error function sends a specified HTTP response code (in this case "Internal Server Error") and error message. Already the decision to put this in a separate function is paying off.

Now let's fix up saveHandler:

func saveHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/save/"):]
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    err := p.save()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
Any errors that occur during p.save() will be reported to the user.

Template caching
There is an inefficiency in this code: renderTemplate calls ParseFiles every time a page is rendered. A better approach would be to call ParseFiles once at program initialization, parsing all templates into a single *Template. Then we can use the ExecuteTemplate method to render a specific template.

First we create a global variable named templates, and initialize it with ParseFiles.

var templates = template.Must(template.ParseFiles("edit.html", "view.html"))
The function template.Must is a convenience wrapper that panics when passed a non-nil error value, and otherwise returns the *Template unaltered. A panic is appropriate here; if the templates can't be loaded the only sensible thing to do is exit the program.

The ParseFiles function takes any number of string arguments that identify our template files, and parses those files into templates that are named after the base file name. If we were to add more templates to our program, we would add their names to the ParseFiles call's arguments.

We then modify the renderTemplate function to call the templates.ExecuteTemplate method with the name of the appropriate template:

func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
    err := templates.ExecuteTemplate(w, tmpl+".html", p)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
}
Note that the template name is the template file name, so we must append ".html" to the tmpl argument.

Validation
As you may have observed, this program has a serious security flaw: a user can supply an arbitrary path to be read/written on the server. To mitigate this, we can write a function to validate the title with a regular expression.

First, add "regexp" to the import list. Then we can create a global variable to store our validation expression:

var validPath = regexp.MustCompile("^/(edit|save|view)/([a-zA-Z0-9]+)$")
The function regexp.MustCompile will parse and compile the regular expression, and return a regexp.Regexp. MustCompile is distinct from Compile in that it will panic if the expression compilation fails, while Compile returns an error as a second parameter.

Now, let's write a function that uses the validPath expression to validate path and extract the page title:

func getTitle(w http.ResponseWriter, r *http.Request) (string, error) {
    m := validPath.FindStringSubmatch(r.URL.Path)
    if m == nil {
        http.NotFound(w, r)
        return "", errors.New("Invalid Page Title")
    }
    return m[2], nil // The title is the second subexpression.
}
If the title is valid, it will be returned along with a nil error value. If the title is invalid, the function will write a "404 Not Found" error to the HTTP connection, and return an error to the handler. To create a new error, we have to import the errors package.

Let's put a call to getTitle in each of the handlers:

func viewHandler(w http.ResponseWriter, r *http.Request) {
    title, err := getTitle(w, r)
    if err != nil {
        return
    }
    p, err := loadPage(title)
    if err != nil {
        http.Redirect(w, r, "/edit/"+title, http.StatusFound)
        return
    }
    renderTemplate(w, "view", p)
}
func editHandler(w http.ResponseWriter, r *http.Request) {
    title, err := getTitle(w, r)
    if err != nil {
        return
    }
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    renderTemplate(w, "edit", p)
}
func saveHandler(w http.ResponseWriter, r *http.Request) {
    title, err := getTitle(w, r)
    if err != nil {
        return
    }
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    err = p.save()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
Introducing Function Literals and Closures
Catching the error condition in each handler introduces a lot of repeated code. What if we could wrap each of the handlers in a function that does this validation and error checking? Go's function literals provide a powerful means of abstracting functionality that can help us here.

First, we re-write the function definition of each of the handlers to accept a title string:

func viewHandler(w http.ResponseWriter, r *http.Request, title string)
func editHandler(w http.ResponseWriter, r *http.Request, title string)
func saveHandler(w http.ResponseWriter, r *http.Request, title string)
Now let's define a wrapper function that takes a function of the above type, and returns a function of type http.HandlerFunc (suitable to be passed to the function http.HandleFunc):

func makeHandler(fn func (http.ResponseWriter, *http.Request, string)) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// Here we will extract the page title from the Request,
		// and call the provided handler 'fn'
	}
}
The returned function is called a closure because it encloses values defined outside of it. In this case, the variable fn (the single argument to makeHandler) is enclosed by the closure. The variable fn will be one of our save, edit, or view handlers.

Now we can take the code from getTitle and use it here (with some minor modifications):

func makeHandler(fn func(http.ResponseWriter, *http.Request, string)) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        m := validPath.FindStringSubmatch(r.URL.Path)
        if m == nil {
            http.NotFound(w, r)
            return
        }
        fn(w, r, m[2])
    }
}
The closure returned by makeHandler is a function that takes an http.ResponseWriter and http.Request (in other words, an http.HandlerFunc). The closure extracts the title from the request path, and validates it with the TitleValidator regexp. If the title is invalid, an error will be written to the ResponseWriter using the http.NotFound function. If the title is valid, the enclosed handler function fn will be called with the ResponseWriter, Request, and title as arguments.

Now we can wrap the handler functions with makeHandler in main, before they are registered with the http package:

func main() {
    flag.Parse()
    http.HandleFunc("/view/", makeHandler(viewHandler))
    http.HandleFunc("/edit/", makeHandler(editHandler))
    http.HandleFunc("/save/", makeHandler(saveHandler))

    if *addr {
        l, err := net.Listen("tcp", "127.0.0.1:0")
        if err != nil {
            log.Fatal(err)
        }
        err = ioutil.WriteFile("final-port.txt", []byte(l.Addr().String()), 0644)
        if err != nil {
            log.Fatal(err)
        }
        s := &http.Server{}
        s.Serve(l)
        return
    }
    
    http.ListenAndServe(":8080", nil)
}
Finally we remove the calls to getTitle from the handler functions, making them much simpler:

func viewHandler(w http.ResponseWriter, r *http.Request, title string) {
    p, err := loadPage(title)
    if err != nil {
        http.Redirect(w, r, "/edit/"+title, http.StatusFound)
        return
    }
    renderTemplate(w, "view", p)
}
func editHandler(w http.ResponseWriter, r *http.Request, title string) {
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    renderTemplate(w, "edit", p)
}
func saveHandler(w http.ResponseWriter, r *http.Request, title string) {
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    err := p.save()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
Try it out!
Click here to view the final code listing.

Recompile the code, and run the app:

$ go build wiki.go
$ ./wiki
Visiting http://localhost:8080/view/ANewPage should present you with the page edit form. You should then be able to enter some text, click 'Save', and be redirected to the newly created page.

Other tasks
Here are some simple tasks you might want to tackle on your own:

Store templates in tmpl/ and page data in data/.
Add a handler to make the web root redirect to /view/FrontPage.
Spruce up the page templates by making them valid HTML and adding some CSS rules.
Implement inter-page linking by converting instances of [PageName] to 
<a href="/view/PageName">PageName</a>. (hint: you could use regexp.ReplaceAllFunc to do this)
构建版本 devel +f22911f Thu Apr 16 05:55:22 2015 +0000.
除特别注明外， 本页内容均采用知识共享-署名（CC-BY）3.0协议授权，代码采用BSD协议授权。
服务条款 | 隐私政策
```

```