# Adding Backend

> Note: While working with the backend, it is advisable to open the Web Inspector, and check the console & network tab regularly. These two tabs contain all the information relate to how Vue is interacting with the server.

In the previous chapters, we learned basic concepts, now, it is time to add the functionality to talk to the server. As of now, when we run the code in chapter3, we have to start a basic server like that provided by Python to render the pages. When we add a task or category, it gets added during runtime in the JavaScript arrays. But if we refresh the page, it goes away. Also, there is a problem loading tasks related to a category.

It is time to add a server to the app. It'll fetch the tasks when we click Pending/Completed/Deleted or any of the category links.

As per our architecture, we won't be needing `pendingTasks` and `categoryTasks`. Earlier they were required if we had to store the respective tasks in there, but since we will be fetching the tasks from the server, we can do away with them.`

For this book, we will be using  a server written in Go, it is http://github.com/thewhitetulip/Tasks-vue, if you want to write your own server, please feel free to write it, otherwise download and use the example code. If you want to learn writing servers, the author has written another tutorial/book http://github.com/thewhitetulip/web-dev-golang-anti-textbook/.

We will start with fetching tasks from the backend, you'll notice that we currently have everything hardcoded, as we go along in this chapter, we'll be generalizing it. At the end of this chapter, we'll have a fully working app in Vue. 

> Note: This is a introductory tutorial and **not** an advanced tutorial, it is assumed that after reading this tutorial, you'll either practice building apps on your own, or you'll read advanced tutorials.

Since there are already many JS libraries which deal with AJAX, the Vue authors quite correctly decided not to write their own version along with Vue, you are free to choose any of the existing libraries, we'll be using `vue-resource` in this tutorial. **The syntax varies with each library**.

#### Notifications

Notifications, when not overused, tend to make the user experience great. We won't be using any library for notifications, we will roll out our own version of it.

```html <a v-if="notificationVisible" v-on:click="hide"><div id = "noti" class="notification"><span id="message"><p id="actlMsg">${notification}</p></span></div>                           </a>```

We need to define two variables `notificationVisible` and `notification` which control the visibility and the content of the notification. We write a function to notify the users.

    // notify toggles the notification and set the content of the
    // notification with the argument passed in it.
    notify: function (message) {
      this.notificationVisible = true;
      this.notification = message;
    },

For showing a notification, we just need to call, `this.notify("Notify");` Notifications are particularly helpful when the user is not having internet connectivity, otherwise, the user would not understand what happened, if our app doesn't complain, "Unable to perform this task".

#### Fetching tasks

    FetchTasks: function () {
      this.$http.get('/task/').then(response => response.json()).then(result => {
        if (result != null) {
          Vue.set(this.$data, 'tasks', result);
        } else {
          this.tasks = [];
        }
      }).catch(err => {
        console.log(err);
        this.notify("Unable to fetch Tasks")
      });


We send a HTTP GET request on URL `/task/` using `vue-resource`. The short hand version of using `vue-resource` is `this.$http`. We can call any http method using `this.$http` like get, put, delete, post, head etc, the syntax is same as `get`, we use `this.$http.put` for PUT and likewise. 

In the first `then`, which gets the output what the http call is going to give, we convert the response to JSON and store it in `result` and use the result to set the `tasks` object with the `result` object **only** if result is not null. If we directly assign the result object it causes problems when there are no pending tasks, since it will set `tasks` as null. Finally, if we have a error (if the webserver returns anything error related using HTTP Status codes for BadRequest or InternalServerError or, if the server is not reachable) then we show a notification that we aren't able to fetch tasks.

> Vue.set takes three arguments, the root element, the name of the variable to set and the value. This is because of a limitation JS has and when we use `Vue.set`, it is reactive, if we directly assign it, Vue is not able to identify if there has been a change in the variable.

We want to fetch tasks as soon as the user opens our webpage. For this, we add it to the `mounted` block. The `mounted` block gets executed when the Vue app is mounted into our HTML page. Later, we will be checking if the user is logged in and if it is true, we'll fetch the tasks. We have added a block of html for `signup` and `login` and wrapped it in the `<template v-show="isLoggedIn">`. We'll be seeing the login functionality at the end of this chapter. It requires change to the HTML. Categories and Tasks should be fetched at the start, since they are core to our app.

#### Categories

    FetchCategories: function () {
      this.$http.get('/categories/').then(response => response.json()).then(result => {
        Vue.set(this.$data, 'categories', result);
      }).catch(err => {
        console.log(err);
        this.notify("Unable to fetch categories");
      });
    },

At this moment, we shall see the app rendering data from our database, for the sake of convenience, we have bundled a sample sqlite database which has tasks and categories and we will be working on this for this chapter, chapter5/final/Tasks-vue/tasks.db. Please follow the instructions given in the readme for how to install and use this app.

#### Trashing a task

When we click on the delete icon displayed in a task, it is supposed to trash the task. We will send HTTP DELETE /task/id where id is the id of the task.

It is nearly same as fetch tasks, with one important difference, we are not fetching anything from the server as of yet, and in the result block, we'll splice the array at `index` by one element to delete the task from our front end. The splicing should only be done in the success block, if we do that in the error block then it is totally wrong.

In this example, we append the taskID to the URL, thus, if we send a DELETE `/task/10`, it will mark that task as trashed.

    // Trashs a task, won't delete from db.
    TrashTask: function (taskIndex, taskID, category) {
      this.$http.delete('/task/' + taskID).then(response => response.json()).then(result => {
        this.tasks.splice(taskIndex, 1);
        this.notify("Task deleted");
        this.UpdateCategoryCount(category, "-", 1);
      }).catch(err => {
        console.log(err);
        this.notify("Unable to trash Tash");
      });
    },

#### Updating task count

The navigation drawer contains the list of all categories and the number of tasks associated with each category which are pending. When we perform operations on any task, the count should change, it doesn't automatically happen and we have to manually change it. We call the `this.UpdateCategoryCount` function to do so.

The first argument is the category name, the second operation can either be + or -, the third operation is the number we want to add/subtract. Currently we can get away with hardcoding to 1, but in the interest of future changes, we keep the value dynamic.

    UpdateCategoryCount: function (name, action, value) {
      categoryIndex = 0;
      for (c in this.categories) {
        if (this.categories[c].categoryName == name) {
          switch (action) {
            case "+":
                this.categories[c].taskCount += value;
            case "-":
                this.categories[c].taskCount -= value;
          }
          break;
        }
      }
    },


#### Completing a task

Same as above, the two differences are the URL it calls and the HTTP method it uses.

    CompleteTask: function (taskIndex, taskID, category) {
      this.$http.get('/complete-task/' + taskID).then(response => response.json()).then(result => {
        this.tasks.splice(taskIndex, 1);
        this.notify("Marked task as complete");
        this.UpdateCategoryCount(category, "-", 1);
      }).catch(err => {
        console.log(err);
        this.notify("Unable to mark as complete");
      });
    },

The next logical portion of our app would be the edit task feature, but it requires that we have worked out the inserting task feature, thus we will be postponing it a bit in our sequence. What we will be looking now, is rendering `completed` and `deleted` tasks.

#### Completed/Pending/Deleted

`FetchTasks` will fetch the tasks from the server, it will be called when we mount the app. But we need a way to show the pending tasks when the user clicks the navigation drawer.

    showCompletedTasks: function (type) {
      this.$http.get('/completed/').then(response => response.json()).then(result => {
        if (result != null) {
                Vue.set(this.$data, 'tasks', result);
        }
        this.selectedTaskTypeName = 'completed';
        this.navigation = 'Completed';
        this.selectedCategoryName = '';
      }).catch(err => {
        console.log(err);
        this.notify("Unable to fetch tasks");
      });
    },
    showPendingTasks: function (type) {
      this.FetchTasks();
      this.selectedTaskTypeName = 'pending';
      this.navigation = 'Pending';
      this.selectedCategoryName = ''
    },
    showDeletedTasks: function (type) {
      this.$http.get('/deleted/').then(response => response.json()).then(result => {
        if (result != null) {
                Vue.set(this.$data, 'tasks', result);
        }
        this.selectedTaskTypeName = 'deleted';
        this.navigation = 'Deleted';
        this.selectedCategoryName = ''
      }).catch(err => {
        console.log(err);
        this.notify("Unable to fetch tasks");
      });
    },


These three functions are nearly the same. Thus, we are having just one paragraph about the three of them.

`navigation`: It is the name displayed at the top left corner of our webpage, it will be pending/delete/completed or category name, depending on what we are seeing.

`selectedCategoryName`: If the tasks we are seeing are related to a category, then this has the name of the category

`selectedTaskTypeName`: If the tasks are pending/completed/deleted, then this has the value of either of the three.

At no given point should both the variables have value.

The value of the last two variables is used to set the active indicator of our navigation drawer, when we click `Pending` link, we want that element to be highlighted, thus we set these two variables and we use it in the html page. `<a v-on:click="showPendingTasks" v-bind:class="{active:(selectedTaskTypeName=='pending')}">`.

#### Fetch task by category

`<a v-on:click="taskByCategory(category.categoryName)" v-bind:class="{active: (selectedCategoryName==category.categoryName)}">`

    taskByCategory: function (category) {
      this.selectedCategoryName = category;
      this.navigation = this.selectedCategoryName;
      this.selectedTaskTypeName = '';
      this.$http.get('/category/' + this.selectedCategoryName).then(response => response.json()).then(result => {
        if (result != null) {
          Vue.set(this.$data, 'tasks', result);
        }
      }).catch(err => {
          console.log(err);
          this.notify("Unable to fetch tasks");
      });
    },    

Here, we fetch tasks depending on the category name selected by the user. The functionality is exactly same as FetchTasks.

#### Adding Tasks.

    AddTask: function (item) {
      this.$http.put('/task/', this.task, {
        emulateJSON: true
      }).then(response => response).then(result => {
        if (this.task.ishidden == false) {
          this.tasks.push(this.task);
        }

        this.UpdateCategoryCount(this.task.category, "+", 1);

        this.task = {
          title: '',
          content: '',
          category: '',
          priority: '',
          comments: [],
          showComment: false
        }
      }).catch(err => {
        console.log(err);
        this.notify("Unable to add Task");
      });
      $('#addNoteModal').modal('hide');
    },

When we want to add a task, we send a form entry back to the server. This is different from the pattern which was forming where we send an http request and got an array back, here, we send form data to the server. For the form value to be recognized by the server, we need to have the `emulateJSON:true` argument as the argument after the object which we want to send. **If we do not have emulateJSON set to true, server won't be able to identify that we sent a form** 

If it is success, and if the `ishidden` value is false, then we push the task into the `tasks` array. Then, we update the category count. `ishidden` is for the feature which we have in our app wherein the user can add a task which isn't visible in the timeline, it is only visible in the category view.

#### HTML changes for editing task

When we see the html, we have only one modal to add a task. The same modal has to be used for the updating of the task, we create a variable in our data element to keep the state of create or edit, if `isEditing` is true, we want to be able to update the task, if it is false, we'll add a new task.

     <template v-if="isEditing">
         <button type="button" class="btn btn-primary" v-on:click="UpdateTask(task)">Update Task</button>
     </template>
     <template v-else>
         <button type="button" class="btn btn-primary" v-on:click="AddTask(task)">Add Task</button>
     </template>

For updating a task, we first need to store the task in our `this.task`, that will be done when the user clicks on the pencil icon on our task, it calls the `edit` method, 

The flow is like this

User clicks the `edit` button on a task => It opens up the modal => User clicks "Update Task"

    // will edit a task
    edit: function (index) {
      this.isEditing = true;
      t = this.tasks[index];
      this.task.title = t.title;
      this.task.id = t.id;
      this.taskIDEdit = t.id;
      this.task.content = t.content;
      this.task.priority = t.priority;
      this.task.category = t.category;
      $('#addNoteModal').modal('show');
    },

After this call is successfully completed, the user will see the `Update Task` button which will call our `UpdateTask` method.

    UpdateTask: function (item) {
      this.$http.post('/task/', this.task, {
        emulateJSON: true
      }).then(response => response).then(result => {
        index = 0;
        for (t in this.tasks) {
          if (t.id == this.taskIDEdit) {
            index = this.tasks.indexOf(t);
          }
        }
        newTask = this.task;

        this.tasks[index].title = newTask.title;
        this.tasks[index].category = newTask.category;
        this.tasks[index].content = newTask.content;
        this.tasks[index].priority = newTask.priority;

        this.notify("Updated task");

        this.task = {
          title: '',
          content: '',
          category: '',
          priority: '',
          comments: [],
          showComment: false
        }
      }).catch(err => {
        console.log(err);
        this.notify("Unable to update Task");
      });
      $('#addNoteModal').modal('hide');
    },

#### Adding categories

     <h5> Categories</h5>
     <span id="categoryForm">
        <form  method="POST" v-on:submit.prevent="OnSubmit">
          <span>
                <input type="text" name="category" width="50px" v-model="category.categoryName">
                <button v-on:click="addCategory" class="btn btn-primary">Add</button>
          </span>
       </form>
     </span>


    addCategory: function () {
      console.log(this.category);
      this.$http.put("/category/", this.category, {
        emulateJSON: true
      }).then(response => response.json()).then(result => {
        this.category.taskCount = 0;
        this.categories.push(this.category);
        this.category = {
          categoryID: '',
          categoryName: '',
          taskCount: ''
        };
        this.notify('Category Added');
      }).catch(err => {
        console.log(err);
        this.notify("Unable to add category");
      });
    },


#### Deleting Category

`<a v-on:click="deleteCategory(selectedCategoryName)">`

    deleteCategory: function (name) {
      this.$http.delete('/category/' + name).then(response => response.json())
        .then(result => {
          console.log('deleting ' + name);
          var index = 0;
          for (category in this.categories) {
            if (this.categories[category].categoryName == name) {
              index = this.categories.indexOf(category);
            }
          }
          this.categories.splice(index, 1);
          this.FetchTasks();
          this.navigation = 'Pending';
          this.selectedTaskTypeName = 'pending'
        }).catch(err => {
          console.log(err);
          this.notify("Unable to delete category");
        });
    },

If you notice, we have only one bar where we show the menu items, we do not want to show `Delete category` button when we are viewing tasks from pending or completed or deleted, hence we wrap it up in `<template v-if="selectedCategoryName">`, that way, we'll see Delete or Edit category only when we are viewing records of a category.

#### Editing category

`<button  class="btn btn-action glyphicon glyphicon-pencil"  type="button" @click="toggleEditCategoryForm()"></button>`

This button will toggle the visibility of

    <template v-if="categoryEdit">
       <form method="POST" id="EditForm" v-on:submit.prevent="onSubmit">
           <input type="text" name="catname" placeholder="new category name" v-model="newCategoryName">
           <button type="submit" v-on:click="updateCategory(selectedCategoryName, newCategoryName)" class="btn btn-default"> Submit </button>
       </form>
    </template>

    updateCategory: function (oldName, newName) {
      category = {
        newCategoryName: this.newCategoryName
      }
      this.$http.post('/category/' + oldName, category, {
        emulateJSON: true
      }).then(response => response.json()).then(result => {

        for (category in this.categories) {
          if (this.categories[category].categoryName == oldName) {
            this.categories[category].categoryName = newName;
            console.log('Updated');
            this.navigation = newName;
            this.toggleEditCategoryForm();
          }
        }
      }).catch(err => {
        console.log(err);
        this.notify("Unable to update Task");
      });
    }


#### Login

For having the html for login and signup forms, we need to wrap our `<span v-id="tasks.length"` span tag by a `<template v-else>`.

	<div id="timeline">
        <template v-if="!isLoggedIn">
            <div class="note">
                <p class="noteHeading ">Login </p>
                <div class="form-group">
                    <form v-on:submit.prevent="onSubmit">
                        <input type="text" name="username" class="loginbutton" placeholder="Username" v-model="userLogin.username" />
                        <input type="password" name="password" class="loginbutton" placeholder="Password" v-model="userLogin.password" />
                        <input type="submit" value="Login" class="btn btn-primary" @click="login" />
                    </form>
                </div>
            </div>
            <div class="note">
                <p class="noteHeading ">Sign up </p>
                <div class="form-group">
                    <form  v-on:submit.prevent="onSubmit">
                        <input type="text" name="username" class="loginbutton" placeholder="Username" v-model="userSignup.username" />
                        <input type="password" name="password" class="loginbutton" placeholder="Password" v-model="userSignup.password" />
                        <input type="email" name="email" class="loginbutton" placeholder="demo@demo.com" v-model="userSignup.email"/>
                        <input type="submit" value="Signup" class="btn btn-primary"  @click="signup()" />
                    </form>
                </div>
            </div>
        </template>
        <template v-else>
	<span v-if="tasks.length">
	....


    login: function () {
      this.$http.post('/login/', this.userLogin, {
        emulateJSON: true
      }).then(response => response.json()).then(result => {
        this.isLoggedIn = true;
        this.FetchCategories();
        this.FetchTasks();
        this.userLogin = {
          username: '',
          password: ''
        }
      }).catch(err => {
        console.log(err);
        this.notify("Unable to login");
      });
    },

We have a login form bound to `userLogin`, we send it to `/login/` and if it is a success, we set the `isLoggedIn` variable to true which will display our tasks instead of the login page, this is a terrible way of logging a user in, but before jumping to using routers, I feel it is necessary to understand barebone components first. Once we are logged in, we fetch tasks and categories.

#### Logout

`<a @click="logout()">`

After login, we have to logout.

    logout: function () {
      this.$http.get('/logout/').then(response => response.json())
        .then(result => {
          this.isLoggedIn = false;
        }).catch(err => {
          console.log(err);
          this.notify("Unable to logout");
        });
    },

#### Signup

    signup: function () {
      this.$http.put('/signup/', this.userSignup, {
        emulateJSON: true
      }).then(response => response.json()).then(result => {
        this.notify("Sign up successful, pls login");
        this.userSignup = {
          username: '',
          password: '',
          email: ''
        }
      }).catch(err => {
        console.log(err);
        this.notify("Unable to signup");
      });
    },

    userSignup : {
        username: '',
        password: '',
        email: ''
    },

At this point in time, you'd notice that if we refresh the page, we are shown the login page. This is because, we can't store persistent objects in memory, for auto login feature, we create another function to check if we are logged in, it'll send a GET to `/login/` and if we get a Status Okay, we will set the `isLoggedIn` variable to true and fetch tasks and categories.

    checklogin: function () {
      this.$http.get('/login/').then(response => response.json()).then(result => {
        this.isLoggedIn = result.loggedin;
        this.FetchCategories();
        this.FetchTasks();
      }).catch(err => {
        console.log(err);
        this.notify("Something went wrong")
      });
    },

Also, in the mounted block, we won't be calling fetch tasks & categories, we will be calling the checkLogin function.

  mounted: function () {
        this.checklogin();
  },

Now, we if refresh the page, it'll show us the task list if we are logged in and a login page otherwise.
