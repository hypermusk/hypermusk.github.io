---
layout: post
title: Make the Todo List Web app with these API and Angularjs
tags: main
---

The web app will be using [Angularjs](http://angularjs.org/) to build UI, directly work with the json APIs exposed by HyperMusk.


![](/images/todo-lists-web-app/screensample.png)

The left side is the ToDo lists, the right side is the ToDo in each list. click any list will refresh the ToDos in the right. And input something on the top text field and hit enter, It will add the ToDo to the current selected ToDo list.

First you need to use the `hypermusk` command to generate the javascript file from the [Go API defination](https://github.com/hypermusk/todoapp/blob/master/todoapi/api.go)

	hypermusk -pkg=github.com/hypermusk/todoapp/todoapi -lang=javascript -outdir=./web

It will generate a file called `todoapi.js` in the web directory. inside it includes each methods for calling corresponding APIs from the server, You shouldn't change anything of that file, But only include that into your html file to be ready to use:

		<script src="jquery.js"></script>
		<script src="angular.min.js"></script>
		<script src="todoapi.js"></script>
		<script src="todo.js"></script>


It uses jquery to do the ajax request, So the jquery library is necessary.

Note that the `todo.js` file is a javascript file for Angularjs to build the UI. It first looks like this:

	todoapi.baseurl="http://localhost:9000/api";
	var appservice = new todoapi.AppService();
	var userservice = appservice.GetUserService("admin@example.com", "nimda");


The todoapi is an javascript object as a namespace of all the API methods. So set the `baseurl` property to let it know where is the API server located.

All the rest is generated according to the [Go API defination](https://github.com/hypermusk/todoapp/blob/master/todoapi/api.go). It simulate the best of how to invoke the API in the javascript fashion. For example if you want to call `GetTodoLists` api:

	userservice.GetTodoLists(function(list, err){
		// Do something with the return result list, or deal with err
	});

Note that how many results returned form the API is all defined in [Go API defination](https://github.com/hypermusk/todoapp/blob/master/todoapi/api.go), So both front end developer and backend developer only needs to check that defination in Go language format to know what kind of APIs the system has.


Here is almost all the javascript code the ToDo app has:


	function TodoListCtrl($scope) {
		userservice.GetTodoLists(function(list, err){
			$scope.$apply(function(){
				$scope.todo_lists = list;
				if(list.length > 0) {
					$scope.selectedListId = list[0].Id;
					$scope.refreshTodoItems($scope.selectedListId);
				}
			});

		});

		$scope.refreshTodoItems = function(listId) {
			userservice.GetTodoItems(listId, function(items, err){
				$scope.$apply(function(){
					$scope.current_items = items;
				});
			})
		};

		$scope.chooseList = function(selectedScope) {
			$scope.selectedListId = selectedScope.l.Id;
			$scope.refreshTodoItems($scope.selectedListId);
		};

		$scope.addItem = function(evt) {
			userservice.CreateTodo($scope.selectedListId, $scope.newTodoItemContent, function(err){
				$scope.refreshTodoItems($scope.selectedListId);
				$scope.newTodoItemContent = "";
			})
		};
	}

What they do is call the API and apply them to the view. And all the detail of what attributes the API result has, again is definted clearly in that [Go API defination](https://github.com/hypermusk/todoapp/blob/master/todoapi/api.go) file.

Take the TodoList object for example:

	type TodoList struct {
		Id   string
		Name string
	}

It only includes these two fields `Id`, and `Name`, both are string type. Accordingly the returned JSON will be something like this:

	{"Id":"1","Name":"222"}

And you can use those attributes directly in your Angularjs view.

See on [Github](https://github.com/hypermusk/todoapp/tree/master/web) for the full Web front end of the Todo app.


Next: [Make the Todo List iOS app with these API](/2013/09/26/make-an-todo-list-ios-app-with-these-api.html)
