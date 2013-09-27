---
layout: post
title: Todo Lists App
tags: main
---

Here I'll describe the basic requirements of this app.

- You can create a Todo List
- You can create a Todo item inside one of the lists
- Be able to check the Todo item to mark it finish
- Be able to mark the finished item to send it back

From these requirements, I created the Go language api definition with in these directory structure:

 ![](/images/todo-lists-app/api.png)

the `api.go` file will look like this:

	package todoapi

	type TodoList struct {
		Id   string
		Name string
	}

	type TodoItem struct {
		Id      string
		ListId  string
		Content string
		Done    bool
	}

	type AppService interface {
		GetUserService(email string, password string) (service UserService, err error)
	}

	type UserService interface {
		GetTodoLists() (list []*TodoList, err error)
		GetTodoItems(listId string) (list []*TodoItem, err error)
		PutTodoList(name string) (err error)
		CreateTodo(listId string, content string) (err error)
		DoneTodo(todoItemId string) (err error)
		UndoneTodo(todoItemId string) (err error)
	}


Which you can see it's simple Go structs and interfaces. But without logic implementations. Because we want to define our system API interfaces, this file will give the user of the API enough information for what you can do with the API.

Take `GetTodoLists` for example, client side could use `email` and `password` to get a `UserService`, Which is a way of authentication, that invalid email with password combination can't get `UserService`, So there is no way to invoke the `GetTodoLists` api method.

And the `GetTodoLists` describe clearly that it takes no arguments, But could return a list of `TodoList`, the `TodoList` struct includes `Name` and `Id` properties.

If I want to invoke the API with JavaScript, I will imagine It would be nice If I can do:

	userservice.GetTodoLists(function(list, err){
		for(var i=0; i<list.length; i++){
                    console.log(list[i].Name)
                }
	});

Because the `GetTodoLists` definition have two return values, `list` and `err`, So we can see the JavaScript callback that have two arguments. that mapping directly to the Go return values.

The `hypermusk` command that could generate javascript libraries that can do exactly that.

    hypermusk -pkg=github.com/hypermusk/todoapp/todoapi -lang=javascript -outdir=./web

The `-pkg` passed in the Go package of the API definition, `-lang` tells it you want to generate client side library to use to call the API remotely by using JavaScript. and the `-outdir` will tell it where to generate the file.

So If I ask you how to call the method `CreateTodo` in JavaScript?

The answer would be:

    userservice.CreateTodo("1", "new todo", function(err){
        console.log(err);
    })

And to call the API you defined in Go in another language like Objective C, or Java is as easy as do it in JavaScript. But following the language's idioms.

## The server side implementation

But If you really call the API with JavaScript, It won't do anything, Because we only defined the interfaces, didn't do any implementation yet.

The implementation of the server side API must be in Go, and it will import the api package to make sure the implementation is following the definition exactly.

First, we will use the `hypermusk` tool again to generate the glue code to expose those API definition to the outside world through json (at the current stage, we only support expose json through http).

    hypermusk -pkg=github.com/hypermusk/todoapp/todoapi -impl=github.com/hypermusk/todoapp/server -lang=server -outdir=./server

We can see it still uses the `todoapi` package, because that's the specification of the app. But it adds another argument `-impl`, Which will hook up the manually implemented package `github.com/hypermusk/todoapp/server` with the generated glue code. We will write the implementation like this:

	type AppServiceImpl struct {
	}

	type UserServiceImpl struct {
	}

	func (a *AppServiceImpl) GetUserService(email string, password string) (service todoapi.UserService, err error) {
		if email != "admin@example.com" && password != "nimda" {
			err = errors.New("wrong credentials")
			return
		}
		service = new(UserServiceImpl)
		return
	}

	func (u *UserServiceImpl) GetTodoLists() (list []*todoapi.TodoList, err error) {
		return
	}

	func (u *UserServiceImpl) GetTodoItems(listId string) (list []*todoapi.TodoItem, err error) {
		return
	}

	func (u *UserServiceImpl) PutTodoList(name string) (err error) {
		return
	}

	func (u *UserServiceImpl) CreateTodo(listId string, name string) (err error) {
		return
	}

	func (u *UserServiceImpl) DoneTodo(todoId string) (err error) {
		return
	}

	func (u *UserServiceImpl) UndoneTodo(todoId string) (err error) {
		return
	}

	var DefaultAppService = new(AppServiceImpl)

The last variable `DefaultAppService` is the secret to hook up the generated code with our API implementation. Since we defined a `AppService` interface that no other methods returns it. and It has a method called `GetUserService` that returns the other service, So it's the root.

Notice that we write a dummy implementation of `GetUserService` to only return the `UserService` when password is nimda. In there we could validate the user with database and try to get the user's id into `UserServiceImpl`, So that it can be used to query user's own todo lists. But right now let's say the system only works for the user admin@example.com.

Next we implement the `GetTodoLists`:

	func (u *UserServiceImpl) GetTodoLists() (list []*todoapi.TodoList, err error) {
		withdb(func(db *sql.DB) {
		list = make([]*todoapi.TodoList, 0)
		rows, err := db.Query("SELECT id, name FROM todo_lists ORDER BY id ASC")
		panicIf(err)

		for rows.Next() {
			newL := new(todoapi.TodoList)
			var id int
			err = rows.Scan(&id, &newL.Name)
			panicIf(err)
			newL.Id = fmt.Sprintf("%d", id)
			list = append(list, newL)
		}
		})
		return
	}

It first query from the postgresql table `todo_lists`, and then populate them into the struct `todoapi.TodoList`, It's quite straight forward.

Next thing we do is create a http server to serve the API:

	package main

	import (
		"github.com/hypermusk/todoapp/server/todoapihttpimpl"
		"log"
		"net/http"
	)

	func main() {

		todoapihttpimpl.AddToMux("/api", http.DefaultServeMux)
		http.DefaultServeMux.Handle("/web/", http.FileServer(http.Dir("../../")))
		err := http.ListenAndServe(":9000", nil)
		if err != nil {
			log.Fatal("ListenAndServe: ", err)
		}
	}

The `todoapihttpimpl` package is the glue code that `hypermusk` generated. And it has a `AddToMux` method that could expose all those APIs through a url like `/api/UserService/GetTodoLists.json`

After this, you start the server with `go run main.go`, And if there is no compile errors, You should be able to serve the APIs to your javascript, here is the proof.

 ![](/images/todo-lists-app/web_console.png)

And how the http round trip is if you want to know the internal of how it works:

 ![](/images/todo-lists-app/tcpspy.png)


And the full code of this app is at: <a href="https://github.com/hypermusk/todoapp">https://github.com/hypermusk/todoapp</a>


Next: [Make an iOS app with these API](/2013/09/27/make-an-todo-list-ios-app-with-these-api.html)


