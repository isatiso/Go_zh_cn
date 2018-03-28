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



# 引入 `net/http` 

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

如果你访问 `/view/APageThatDoesntExist` 会发生什么？你会看到一个有 HTML 的页面。这是因为它忽略了 `loadPage` 返回的错误信息，然后继续试图用空的数据渲染模板。所以我们将它修改为，如果请求页面不存在，就将客户端重定向到编辑页面，这样就可以创建文本了：

```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, err := loadPage(title)
    if err != nil {
        http.Redirect(w, r, "/edit/"+title, http.StatusFound)
        return
    }
    renderTemplate(w, "view", p)
}
```

`http.Redirect` 函数添加了一个 HTTP 状态码 `http.StatusFound` （302），并向 HTTP 的响应里面添加了一个 `Location` 头部信息。



# 保存页面

函数 `saveHandler` 会处理编辑页面上的表单提交。在去掉 `main` 中的注释后，下面我们实现这个处理器：

```go
func saveHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/save/"):]
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    p.save()
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
```

页面标题（在 URL 中获得）和表单的唯一字段 `Body` ，被存储为一个新的 `Page` 。之后调用`save()` 方法将数据写入文件，并且客户端会重定向到 `/view/` 页面。

The value returned by `FormValue` is of type `string`. We must convert that value to `[]byte` before it will fit into the `Page` struct. We use `[]byte(body)` to perform the conversion.

`FormValue` 的返回值类型是 `string` 。我们必须将这个值转换为 `[]byte` ，之后才可以存入 `Page` 结构体。我们使用 `[]byte(body)` 来实现这个转换。

# 异常处理

在我们的程序中，还有几处异常被忽略的地方。这其实是很糟糕的做法，尤其是因为当程序中出现一个异常时，会产生意想不到的行为。好一点的解决方案是处理异常，并将错误信息反馈给用户。这样如果一些东西出了问题，程序会按照我们期望的方式运行，并且用户也可以收到通知。

首先，我们来处理 `renderTemplate` 中的异常：

```go
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
```

`http.Error` 函数会发送一个指定的 HTTP 响应码（在这里就是 “服务器错误”）和一个错误信息。现在，将异常处理提取到分离的函数中的作用已经显现出来。

现在，我们来修理 `saveHandler` ：

```go
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
```

任何在运行 `p.save()` 期间发生的异常都会被报告给用户。 

# 模板缓存

目前的代码中还有一个很低效的地方：`renderTemplate` 每次渲染一个模板时都要调用 `ParseFiles` 。一个更好的途径是，在程序初始化时调用一次 `ParseFiles` ，解析所有模板到一个单独的 `*Template` 中。之后我们可以使用 `ExecuteTemplate` 方法来渲染一个指定的模板。

首先我们创建一个全局变量 `templates` ，使用 `ParseFiles` 对它进行初始化。

```go
var templates = template.Must(template.ParseFiles("edit.html", "view.html"))
```

函数 `template.Must` 是一个便捷包装，如果收到一个非零的 `error` 值，则发出 panic，否则返回一个未修改的 `*Template` 。这里触发 panic 是适当的；如果模板不能加载，那么显然能做的事情只有退出程序。

`ParseFiles` 函数接收任意数量的定位模板文件的字符串参数，并且将这些文件解析到以基础文件名命名的模板中。如果我们想要添加更多的模板到程序中，就要将他们的名字添加到 `ParseFiles` 参数中去。

之后修改 `renderTemplate` 函数，通过适当模板的名字调用 `templates.ExecuteTemplate` 方法：

```go
func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
    err := templates.ExecuteTemplate(w, tmpl+".html", p)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
}
```

需要注意的是，模板名字就是模板文件的名字，所以我们必须向 tmpl 参数后面添加 `".html"` 。

# 验证

如果你注意观察，可能会发现这个程序现在有一些安全缺陷：一个用户可以提供一个随意的路径，来读/写服务器。为了避免这一点，我们可以写一个函数来使用正则表达式验证这个路径是否合法。

首先，添加 `"regexp"` 包到加载列表。之后我们可以创建一个全局变量来存储我们的验证表达式：

```go
var validPath = regexp.MustCompile("^/(edit|save|view)/([a-zA-Z0-9]+)$")
```

函数 `regexp.MustCompile` 会解析并编译正则表达式，然后返回一个 `regexp.Regexp. MustCompile` ，它和 `Compile` 不一样的地方在于，如果表达式编译失败，`Compile` 会返回一个 `error` 作为第二个参数，而这会触发一个 panic。

现在，我们来写一个函数，使用 `validPath` 表达式去验证路径并提取页面标题：

```go
func getTitle(w http.ResponseWriter, r *http.Request) (string, error) {
    m := validPath.FindStringSubmatch(r.URL.Path)
    if m == nil {
        http.NotFound(w, r)
        return "", errors.New("Invalid Page Title")
    }
    return m[2], nil // The title is the second subexpression.
}
```

如果标题是有效的，他就会返回一个 `nil` 作为 `error` 值。如果标题是无效的，函数就会向 HTTP 连接写入一个 `"404 Not Found"` 异常，然后返回一个 `error` 给处理器。我们需要加载 `error` 包来创建一个新的 `error` 。

现在我们让每个处理器中调用 `getTitle` ：

```go
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
```

```go
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
```

```go
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
```



# 引入函数字面量和闭包

在每一个处理器中都捕获异常会产生非常多的重复代码。如果我们可以将每一个异常处理包装成一个函数来进行验证和错误检查呢？Go 的 [函数字面量](https://go-zh.org/ref/spec#Function_literals) 提供了一个非常有用的工具来进行函数功能抽象，在这里会非常有用。

首先我们重写每个处理器的函数定义，来接受一个标题字符串：

```go
func viewHandler(w http.ResponseWriter, r *http.Request, title string)
func editHandler(w http.ResponseWriter, r *http.Request, title string)
func saveHandler(w http.ResponseWriter, r *http.Request, title string)
```

现在，我们来定义一个包装函数，接受上面这些函数类型参数，返回一个类型为 `http.HandlerFunc` （也可以传递给函数 `http.HandlerFunc`）的函数：

```go
func makeHandler(fn func (http.ResponseWriter, *http.Request, string)) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// Here we will extract the page title from the Request,
		// and call the provided handler 'fn'
	}
}
```

这个返回的函数被称为闭包，因为他带入了定义在函数外面的值，变量 `fn`（传递给 `makeHandler` 的参数）被带入了闭包中。变量 `fn` 可以是 `save` ，`edit` ，`view` 中的一个处理器。

现在我们可以把 `getTitle` 中的代码拿到这里来（需要做一些很小的修改）：

```go
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
```

`makeHandler` 返回的闭包是一个函数，需要 `http.ResponseWriter` 和 `http.Request` （换言之，就是 `http.HandlerFunc` ）作为参数。闭包从请求路径中提取标题内容，并且用正则表达式 `TitleValidator ` 做验证。如果标题无效，就会通过 `http.NotFound` 函数向 `ResponseWriter` 写入一个异常。如果标题有效，带入的处理器函数 `fn` 就会使用 `ResponseWriter` ，`Request` ，和 `title` 作为参数调用。

现在我们可以在处理器函数被注册到 http 包中之前，使用 `main` 中的 `makeHandler` 对它们进行包装：

```go
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
```

最后，我们移除了对 `getTitle` 的调用，让处理变得更加简洁：

```go
func viewHandler(w http.ResponseWriter, r *http.Request, title string) {
    p, err := loadPage(title)
    if err != nil {
        http.Redirect(w, r, "/edit/"+title, http.StatusFound)
        return
    }
    renderTemplate(w, "view", p)
}
```

```go
func editHandler(w http.ResponseWriter, r *http.Request, title string) {
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    renderTemplate(w, "edit", p)
}
```

```go
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
```



# 试一试吧！

[点击这里查看最终版本的代码](https://go-zh.org/doc/articles/wiki/final.go)。

重新编译代码，运行这个应用：

```bash
$ go build wiki.go
$ ./wiki
```

访问 http://localhost:8080/view/ANewPage 会看到编辑页面表单。之后你就可以输入一些文本，点击 Save 按钮，之后被重定向到刚刚创建的页面。

# 其他任务

下面是一些你现在可以自己解决的简单任务：

- 将模板存储到 `tmpl/` 文件夹，将页面文件存储到 `data/` 文件夹。
- 添加一个处理器将根目录重定向到 `/view/FrontPage`。
- 美化页面模板，添加一些 CSS 规则，并保持它是一个有效的 HTML 文件。
- 将 `[PageName]` 转换为 `<a href="/view/PageName">PageName</a>` 实现一个页面内部链接。(提示：你可以使用 `regexp.ReplaceAllFunc` 来实现。)

