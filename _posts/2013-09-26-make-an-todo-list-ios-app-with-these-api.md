---
layout: post
title: Make an Todo List iOS app with these API
tags: main
---
Here is the screenshot of the same old Todo List app:

![](/images/todo-lists-ios-app/screensample.png)

First you need to setup a command to generate the iOS/Cocoa SDK code from [Go API defination](https://github.com/hypermusk/todoapp/blob/master/todoapi/api.go)

	hypermusk -pkg=github.com/hypermusk/todoapp/todoapi -lang=objc -outdir=./ios/HyperMuskTodo/HyperMuskTodo/

Then you need to add the generated `Todoapi.h` and `Todoapi.m` to the Xcode project.

![](/images/todo-lists-ios-app/sdkfiles.png)


Then it's ready to use, again, setup the server API request end point first so that the client can make correct request:

	- (void)viewDidLoad
	{
	    [super viewDidLoad];

	    [[Todoapi get] setBaseURL:@"http://localhost:9000/api"];
	    [[Todoapi get] setVerbose:YES];
	    AppService *appService = [AppService alloc];
	    userService = [appService GetUserService:@"admin@example.com" password:@"nimda"];

	    UserServiceGetTodoListsResults *r = [userService GetTodoLists];

	    if (r.Err != NULL) {
	        NSLog(@"GetTodoLists Err: %@", r.Err);
	        return;
	    }

	    if ([r.List count] == 0) {
	        NSLog(@"List size is zero.");
	        return;
	    }
	    // ...
	}


I did it in ViewController, and store that `UserService` object into a property of the controller.

Note that it did the initial `GetTodoLists` call to get the Todo list results from the server and later it use that to assemble it into the view.

Here is the `CreateTodo` call in the iOS app:

	- (IBAction)done:(UIStoryboardSegue *)segue
	{

	    if ([[segue identifier] isEqualToString:@"ReturnInput"]) {
	        AddTodoViewController *addController = [segue sourceViewController];
	        NSString *newTodoContent = [addController todoContent].text;
	        [userService CreateTodo:[self selectedList].Id content:newTodoContent];
	    }
	}


an API call is simple as a local method call, don't need to care about transportation, parsing json, assemble them into Objective C object etc. It all done for you by HyperMusk code generator.


