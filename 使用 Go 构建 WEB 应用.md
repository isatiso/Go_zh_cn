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

```go
$ go build wiki.go
$ ./wiki
This is a sample page.
```

如果你使用 Windows 系統，你需要使用 `wiki` 不带 `./` 来运行程序。

[点击这里查看目前为止所写的代码](https://go-zh.org/doc/articles/wiki/part1.go)

# `net/http` 介绍

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

The main function begins with a call to http.HandleFunc, which tells the http package to handle all requests to the web root ("/") with handler.



It then calls http.ListenAndServe, specifying that it should listen on port 8080 on any interface (":8080"). (Don't worry about its second parameter, nil, for now.) This function will block until the program is terminated.

The function handler is of the type http.HandlerFunc. It takes an http.ResponseWriter and an http.Request as its arguments.

An http.ResponseWriter value assembles the HTTP server's response; by writing to it, we send data to the HTTP client.

An http.Request is a data structure that represents the client HTTP request. r.URL.Path is the path component of the request URL. The trailing [1:] means "create a sub-slice of Path from the 1st character to the end." This drops the leading "/" from the path name.

If you run this program and access the URL:

http://localhost:8080/monkeys
the program would present a page containing:

Hi there, I love monkeys!
Using net/http to serve wiki pages
To use the net/http package, it must be imported:

import (
	"fmt"
	"io/ioutil"
	"net/http"
)
Let's create a handler, viewHandler that will allow users to view a wiki page. It will handle URLs prefixed with "/view/".

func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, _ := loadPage(title)
    fmt.Fprintf(w, "<h1>%s</h1><div>%s</div>", p.Title, p.Body)
}
First, this function extracts the page title from r.URL.Path, the path component of the request URL. The Path is re-sliced with [len("/view/"):] to drop the leading "/view/" component of the request path. This is because the path will invariably begin with "/view/", which is not part of the page's title.

The function then loads the page data, formats the page with a string of simple HTML, and writes it to w, the http.ResponseWriter.

Again, note the use of _ to ignore the error return value from loadPage. This is done here for simplicity and generally considered bad practice. We will attend to this later.

To use this handler, we rewrite our main function to initialize http using the viewHandler to handle any requests under the path /view/.

func main() {
    http.HandleFunc("/view/", viewHandler)
    http.ListenAndServe(":8080", nil)
}
Click here to view the code we've written so far.

Let's create some page data (as test.txt), compile our code, and try serving a wiki page.

Open test.txt file in your editor, and save the string "Hello world" (without quotes) in it.

$ go build wiki.go
$ ./wiki
(If you're using Windows you must type "wiki" without the "./" to run the program.)

With this web server running, a visit to http://localhost:8080/view/test should show a page titled "test" containing the words "Hello world".

Editing Pages
A wiki is not a wiki without the ability to edit pages. Let's create two new handlers: one named editHandler to display an 'edit page' form, and the other named saveHandler to save the data entered via the form.

First, we add them to main():

func main() {
    http.HandleFunc("/view/", viewHandler)
    http.HandleFunc("/edit/", editHandler)
    http.HandleFunc("/save/", saveHandler)
    http.ListenAndServe(":8080", nil)
}
The function editHandler loads the page (or, if it doesn't exist, create an empty Page struct), and displays an HTML form.

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
This function will work fine, but all that hard-coded HTML is ugly. Of course, there is a better way.

The html/template package
The html/template package is part of the Go standard library. We can use html/template to keep the HTML in a separate file, allowing us to change the layout of our edit page without modifying the underlying Go code.

First, we must add html/template to the list of imports. We also won't be using fmt anymore, so we have to remove that.

import (
	"html/template"
	"io/ioutil"
	"net/http"
)
Let's create a template file containing the HTML form. Open a new file named edit.html, and add the following lines:

<h1>Editing {{.Title}}</h1>

<form action="/save/{{.Title}}" method="POST">
<div><textarea name="body" rows="20" cols="80">{{printf "%s" .Body}}</textarea></div>
<div><input type="submit" value="Save"></div>
</form>
Modify editHandler to use the template, instead of the hard-coded HTML:

func editHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/edit/"):]
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    t, _ := template.ParseFiles("edit.html")
    t.Execute(w, p)
}
The function template.ParseFiles will read the contents of edit.html and return a *template.Template.

The method t.Execute executes the template, writing the generated HTML to the http.ResponseWriter. The .Title and .Body dotted identifiers refer to p.Title and p.Body.

Template directives are enclosed in double curly braces. The printf "%s" .Body instruction is a function call that outputs .Body as a string instead of a stream of bytes, the same as a call to fmt.Printf. The html/template package helps guarantee that only safe and correct-looking HTML is generated by template actions. For instance, it automatically escapes any greater than sign (>), replacing it with &gt;, to make sure user data does not corrupt the form HTML.

Since we're working with templates now, let's create a template for our viewHandler called view.html:

<h1>{{.Title}}</h1>

<p>[<a href="/edit/{{.Title}}">edit</a>]</p>

<div>{{printf "%s" .Body}}</div>
Modify viewHandler accordingly:

func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, _ := loadPage(title)
    t, _ := template.ParseFiles("view.html")
    t.Execute(w, p)
}
Notice that we've used almost exactly the same templating code in both handlers. Let's remove this duplication by moving the templating code to its own function:

func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
    t, _ := template.ParseFiles(tmpl + ".html")
    t.Execute(w, p)
}
And modify the handlers to use that function:

func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, _ := loadPage(title)
    renderTemplate(w, "view", p)
}
func editHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/edit/"):]
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    renderTemplate(w, "edit", p)
}
If we comment out the registration of our unimplemented save handler in main, we can once again build and test our program. Click here to view the code we've written so far.

Handling non-existent pages
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