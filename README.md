# Introduction

In this example, I will show you how to use HTMX with GoFiber in a very basic way. The goal is just to get a first idea of HTMX and how it works.

# 1. Start a new Golang project

[Start installing the Go language](https://go.dev/doc/install)

Create a new directory and use the command `go mod init example.com/m/v2` to create a new project.

Once run, you should see a go.mod file in your project directory with the golang version.

# 2. Starting a Fiber server

Once the project is created, install the fiber package with `go get github.com/gofiber/fiber/v2`.

Then create a new file called `main.go` and add the following code:

```go
package main

import (
	"github.com/gofiber/fiber/v2"
)

func main() {
	app := fiber.New()

	app.Get("/", func(c *fiber.Ctx) error {
		return c.SendString("Hello World!")
	})

	app.Listen(":3000")
}
```

Run the server with `go run main.go`.

You should see an output from Fiber with an the url `http://127.0.0.1:3000`.  
And if you open the url in your browser, you should see the message `Hello World!`.

Now that we have our server running, let's add some pages.

# 3. Init the template engine

Fiber comes with multiple template engines out of the box. I like using the Django template because I learned with this one but you can [use any of them](https://docs.gofiber.io/guide/templates/).

Let's first get the Django template engine installed with `go get github.com/gofiber/template/django/v3`.

And add it to our `main.go` file:

```go
package main

import (
	"github.com/gofiber/fiber/v2"
	"github.com/gofiber/template/django/v3"
)

func main() {
	engine := django.New("./views", ".html")

	// Pass the engine to the Views
	app := fiber.New(fiber.Config{
		Views: engine,
	})

	app.Get("/", func(c *fiber.Ctx) error {
		return c.Render("index", fiber.Map{
			"Title": "My title",
			"Text":  "This is a content",
		})
	})

	app.Listen(":3000")
}

```

And create a new directory called `views` with a file called `index.html`.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>{{ Title }}</title>
  </head>
  <body>
    <h1>{{ Text }}</h1>
  </body>
</html>
```

Now if you restart your server and go back to the page, you should see the title and the content.

# Create the base todo list

Ok now that we have set up everything, we can start putting things into the page. For simplicity, I will use Bulma as css framework but feel free to use anything.

For that I add the link in the head of the `index.html` file.

```html
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>{{ Title }}</title>
  <link
    rel="stylesheet"
    href="https://cdn.jsdelivr.net/npm/bulma@1.0.0/css/bulma.min.css"
  />
</head>
```

Now let's add a basic table to body of the page.

```html
<body>
  <table class="table">
    <tbody>
      <tr>
        <th>1</th>
        <td>Todo 1</td>
      </tr>
      <tr>
        <th>2</th>
        <td>Todo 2</td>
      </tr>
    </tbody>
  </table>
</body>
```

Ok, we have your base table. Let's first center it using column before making it responsive. And also some padding to not touch the border of the windows. Just for visual purpose.

```html
<div class="columns is-centered m-4">
  <div class="column is-12-mobile is-8-tablet is-6-desktop">
    <table class="table">
      ...
    </table>
  </div>
</div>
```

Ok looking good, let's change the first column to be a Done button.

```html
<table class="table">
  <tbody>
    <tr>
      <td>
        <button class="button is-success is-small">
          <span>Done</span>
        </button>
      </td>
      <th>Todo 1</th>
    </tr>
    <tr>
      <td>
        <button class="button is-success is-small">
          <span>Done</span>
        </button>
      </td>
      <td>Todo 2</td>
    </tr>
  </tbody>
</table>
```

Now we have a table with two rows and a button for each in the first column. it's time to make it responsive.

# Delete a row using HTMX

HTMX allow you to swap HTML content using what your server send back. So in your case, we want to delete the row. We can remove a row from the table by removing the HTML of the row, or in another term, swaping the HTML of the row by an empty string.

So the first thing to do is to import HTMX in the head of your page.

```html
<script src="https://unpkg.com/htmx.org@2.0.0"></script>
```

Then create a route that return nothing on our server. Note: HTMX also have a special `delete` key word to just remove HTML.

```go
app.Get("/empty", func(c *fiber.Ctx) error {
	return c.SendString("")
})
```

Now we need to call it when the button is press, I recommand you to check the [HTMX documentation](https://htmx.org/docs/), it is very well done as I will not explain HTMX in detail here.

```html
<tr>
  <td>
    <button
      class="button is-success is-small"
      hx-get="/empty"
      hx-target="closest tr"
      x-trigger="click"
      hx-swap="outerHTML"
    >
      <span>Done</span>
    </button>
  </td>
  <th>1</th>
</tr>
```

In this example, when the button is click, it will get what the HTTP Get request of the route `\empty` return and use it to replace the HTML of the parent `tr` element. `tr` being a table row.

# Add a new row

**Note about UX** _In this example, I was thinking having a button `+` button at the end of the list, when you click on it, it is swap for an input field. You could to that in HTMX, and it will be ok when developping as request are very fast. But in a real app, it will take a bit of time. This time is very bad as it build up frustation, it is because the user expect to see a changment instantly. Mostly because you expect to instantly start writting the new TODO when click the + button. If the app freeze, even 0.2s, for this scenario, it will look bad. On the other hand, for the remove, it is ok to have a small delay, because you don't follow with any more actions, it doesn't block you to continue using the app either. It is event a bit rewarding to take 0.2s to remove a row as your have a bit of time to tell yourself "I did that". So keep that in mind, UX is very important, and you need to think about it a bit differently to avoid frustrating the user._

So how do I do that in HTMX? Well I don't, I use JS to do that. It is the perfect use case for JS. If you need something instant, you need JS.

To keep it simple as always, what I like to do is to have hidden part of the UI. So in your example, when the user click on the + button, you hide the button and show an input field. When the user press enter in the input field, you do the opposite.

```html
<button class="button is-success is-small" id="add-todo">
  <span>+</span>
</button>
<input class="is-hidden" type="text" id="todo-input" />
<script>
  const addTodoButton = document.getElementById("add-todo");
  const todoInput = document.getElementById("todo-input");

  addTodoButton.addEventListener("click", () => {
    todoInput.classList.remove("is-hidden");
  });

  todoInput.addEventListener("keyup", (e) => {
    if (e.key === "Enter") {
      todoInput.classList.add("is-hidden");
    }
  });
</script>
```

Ok perfect, now we just need to do a HTMX request when the user press enter in the input field to add a new row.

```html
<input
  class="is-hidden"
  name="input"
  type="text"
  id="todo-input"
  hx-get="/add-todo"
  hx-target="previous tbody"
  hx-trigger="keyup[keyCode==13]"
  hx-swap="beforeend"
  hx-include="[name='input']"
/>
```

This will send a Get request to the `\add-todo` route with the input value in the query string. And then it will qppend the response to the previous `tbody` element, just before `</tbody>`, so inside the table.

Now we need to add the `\add-todo` route to our server.

```go
app.Get("/add-todo", func(c *fiber.Ctx) error {
	input := c.Query("input")
	c.SendString(input)
})
```

But at this point, you just diplay the input field, you don't add a new row to the table. For that let's create a new HTML file named `todo-row.html` in a new directory called `partials` inside the `views` directory.

```html
<tr>
  <th>
    <button
      class="button is-success is-small"
      hx-get="/empty"
      hx-target="closest tr"
      hx-trigger="click"
      hx-swap="outerHTML"
    >
      <span>Done</span>
    </button>
  </th>
  <td>{{ Text }}</td>
</tr>
```

Now we can create a new global variable that will contain the template of this file.

```go
var rowTodoTmpl *pongo2.Template

func main() {
	rowTodoTmpl = pongo2.Must(pongo2.FromFile("views/partials/todo-row.html"))

	// ...
}
```

And then we can use it in our `\add-todo` route.

```go
app.Get("/add-todo", func(c *fiber.Ctx) error {
	input := c.Query("input")

	out, err := rowTodoTmpl.Execute(pongo2.Context{"Text": input})
	if err != nil {
		return c.SendString(err.Error())
	}

	return c.SendString(out)
})
```

And that's it for this example. You saw almost all basics of HTMX and GoFiber. I hope you enjoyed it and I hope you will use it in your own projects.
