### Building Your First Multi-User Realm Mobile Platform iOS App

This tutorial will guide you through writing an iOS app using Realm Swift to sync with the RealmTasks demo apps.

First, [install the MacOS bundle](get-started/installation/mac) if you haven't yet. This will get you set up with the Realm Mobile Platform including a local copy of the Realm Object Server and let you launch the macOS version of RealmTasks.


## 1. Create a new Xcode project

In this section we will create the basic iOS iPhone application skeleton needed for this tutorial.

1. Launch Xcode 8.
2. Click "Create a new Xcode project".
3. Select "iOS", then "Application", then "Single View Application", then click "Next".
4. Enter "MultiUserRealmTasksTutorial" in the "Product Name" field.
5. Select "Swift" from the "Language" dropdown menu.
6. Select "iPhone" from the "Devices" dropdown menu.
7. Select your team name (log in via Xcode's preferences, if necessary) and enter an organization name.
8. Click "Next", then select a location on your Mac to create this project, then click "Create".

## 2. Setting Up Cocoapods

In this section we set up the Cocoapod dependency manager and add Realm's Swift bindings and a utility module that allows us to create a multi-user application using a preconfigured login panel

1. Quit Xcode
2. Open a Terminal window and change to the directory/folder where you created the Xcode _RealmTasksTutorial_ project
2. If you have not already done so, install the [Cocoapods system](https://cocoapods.org/)
	 - Full details are available via the Cocopods site, but the simplest instructions are to type ``sudo gem install cocoapods`` in a terminal window
3. Initialize a new Cocoapods Podfile with ```pod init```  A new file called `Podfile` will be created.
4. Edit the Podfile find the the comment line that reads:

 ` # Pods for MultiUserRealmTasksTutorial`

And add the following:

```ruby
pod 'RealmSwift'
pod 'RealmLoginKit'
```

5. Save the file
6. At the terminal, type `pod install` - this will cause the Cocoapods system to fetch the RealmSwift and RealmLoginKit modules, as well as create a new Xcode workspace file which enabled these modules to be used in this project.


## 3. Setting Up the Storyboard & Views

In this section we will set up our login and main view controller's storyboard connections.

	1. Reopen Xcode, but rather than open `MultiUserRealmTasksTutorial.xcodeproj` use the newly created `MultiUserRealmTasksTutorial.xcworkspace` file; this was created by the Cocoapods dependency manager and should be used going forward

  2. If you have not already, open the `MultiUserRealmTasksTutorial.xcworkspace` with Xcode.

  3. In the Xcode project navigator select the `main.storyboard` file. The Interface builder (IB) will open and show the default single view layout:

	<center><img src="/Graphics/InterfaceBuilder-start.png"> 	</center>

  3.  Adding the TableViewController - on the lower right of the window is the object browser, type "tableview" to narrow down the possible IB objects. There will be a "TableView Controller" object visible. Drag this onto the canvas. Once you have done this the Storyboard view will resemble this:

	<center> <img src="/Graphics/Adding-theTableViewController.png" /></center>


Once you have added the second view controller, you will need to connect the two controllers by a pair of segues, as well as add class names/storyboard IDs for each controller to prepare for the code will be adding in the next sections:
1. Open the storyboard propery viewer to see the ourline view of the contents of both controllers in the sotoryboard. Then, control-drag from the TasksLoginViewController label to the Table View Controller label and select "show" when the popup menu appears. Select the segue that is created between the two controllers, and set the name of ther segue in the property view on the right side to "loginToTasksViewSegue"

2. Do the same from the `TasksLoginViewController` back to the `TasksLoginViewController`.  Here, again, tap the newly created segue (it will be the diagonal line) and name this segue "tasksViewToLoginControllerSegue"

3. You will need to set the class names for each of the view controller objects. To do this select the controllers one at a time, and for the LoginView Controller, set the class name to `TasksLoginViewController` and to the storyboard id to `loginView`.  For the new TableViewController you added, set the class name to `TasksTableViewController` and here set the storyboard id to `tasksView`. A video summary of these tasks can be seen here:



<center> <img src="/Graphics/MUTasks-StoryBoardSetup.gif" /></center></br>


The final configfuration will look like this:

<center> <img src="/Graphics/final-storyboard-config.png" /></center>


## 3. Configuring the Login View Controller

In this section we will rename and then configure the TasksLoginViewController that will allow you to log in an existing user account, or create a new account


  1. Open the  `view controller` file in the project naivator. Click once on it to enable editing of the file name; change the name to `TasksLoginViewController` and press return to rename the file.
  2. Clicking on the filename should also have opened the newly renamed file in the editor. Here too you should replace all references to `ViewController` in the comments and class name with `TasksLoginViewController`
  3. Next, we are going to update the contents of this view controller and take it from a generic, empty controller to one that can display our Login Panel.

  4. Start by modifying the imports to read as follows:


```swift
import UIKit
import RealmSwift
import RealmLoginKit
```



  5. Modify the top of the class file so the following properties are declared:

```swift
class TasksLoginViewController: UITableViewController {
	var loginViewController: LoginViewController!
	var token: NotificationToken!
	var myIdentity = SyncUser.current?.identity!
	var thePersonRecord: Person?
}
```

  6. Next, modify the empty `viewWillAppear` method to

```swift
	override func viewDidAppear(_ animated: Bool) {
			loginViewController = LoginViewController(style: .lightOpaque)

			loginViewController.isServerURLFieldHidden = true // the user doesn't need to see the server IP in production.
			loginViewController.isRegistering = true
			loginViewController.copyrightLabelText = ""

			if (SyncUser.current != nil) {
					// yup - we've got a stored session, so just go right to the UITabView
					Realm.Configuration.defaultConfiguration = ConstandsAndModels.commonRealmConfig
					performSegue(withIdentifier: ConstandsAndModels.kLoginToMainView, sender: self)
			} else {
					// show the RealmLoginKit controller
					//loginViewController = LoginViewController(style: .lightOpaque)
					if loginViewController!.serverURL == nil {
							loginViewController!.serverURL = ConstandsAndModels.syncAuthURL.absoluteString
					}
					// Set a closure that will be called on successful login
					loginViewController.loginSuccessfulHandler = { user in
							DispatchQueue.main.async {

									Realm.asyncOpen(configuration: ConstandsAndModels.commonRealmConfig) { realm, error in
											if let realm = realm {
													Realm.Configuration.defaultConfiguration = BingoConstants.commonRealmConfig
													self.thePersonRecord = Person.createProfile()   // let's make this person a local profile in /~/BingoPrivate
																																					// then dismiss the login view, and...
													self.loginViewController!.dismiss(animated: true, completion: nil)

													// Do a little stats gathering...
													self.logUser()

													// hop right into the main view for the app (this will be set up by the positioning of the tabs in either app
													self.performSegue(withIdentifier: ConstandsAndModels.kLoginToMainView, sender: nil)

											} else if let error = error {
												print("An error occirred on login: \(error.description)")
													Alertift.alert(title:NSLocalizedString( "Unable to login...", comment:  "Unable to login..."), message: NSLocalizedString("Code: \(error) - please try later", comment: "Code: \(error) - please try later"))
															.action(.cancel("Cancel"))
															.show()
											}
									} // of asyncOpen()

							} // of main queue dispatch
					}// of login controller

					present(loginViewController, animated: true, completion: nil)
			}
	}
	```

Optionally, commit your progress in source control.

Your app should now build and run---although so far it doesn't do much, it will show you to login panel you just configured.

Click the stop button to temrinate the app, and we will continue with the rest of the changes needed to create our Realm Tasks app.


<center> <img src="/Graphics/TaskLoginView.png" /></center>


## 4. XCreate the Models and Constants Class File
In this step we are going to create a few constants to help us manage our Realm as well as the class models our Realm will operate on.

From the Project Navigator, right click and select `New File` and when the file selector apprears select `Swift File` and name the file `ConstantsAndModels` and press preturn.  Xcode will create a new Swift file and open it in the editor.

Our first task will be to create some contants that will make opeing and working with Realms easier, then we will define the Task models.  o do this add the following code to the f


Let's start with the Contants; add the following  to the file:

```swift
import Foundation
import RealmSwift

struct Constants {
    // segue names
    static let      kLoginToMainView                = "loginToMainViewSegue"
    static let      kExitToLoginViewSegue           = "segueToLogin"


    // the host tht will do the synch - if oure using the Mac dev kit you probably want this to be localhost/127.0.0.1
    // if you are using the Professional or Enterprise Editions, then this will be a host on the Internet
    static let defaultSyncHost                      = "127.0.0.1"

    // this is purely for talking to the RMP auth system
    static let syncAuthURL                          = URL(string: "http://\(defaultSyncHost):9080")!

    // The following URLs and URI fragments are about talking to the synchronization service and the Realms
    // it manages on behalf of your application:
    static let syncServerURL                        = URL(string: "realm://\(defaultSyncHost):9080/")

    // Note: When we say Realm file we mean literally the entire collection of models/schemas inside that Realm...
    // So we need to be very clear what models that are represented by a given Realm.  For example:

    // this is a realm where we can store profile info - not covered in the main line of this tutorial
    static let commonRealmURL                       = URL(string: "realm://\(defaultSyncHost):9080/CommonRealm")!
    static let commonRealmConfig                    = Realm.Configuration(syncConfiguration: SyncConfiguration(user: SyncUser.current!, realmURL: commonRealmURL),objectTypes: [Person.self])

    //  this is a task Realm comptible with the fully version of RealmTasks for iOS/Android/C#
    static let tasksRealmURL                       = URL(string: "realm://\(defaultSyncHost):9080/Tasks")!
    static let tasksRealmConfig                    = Realm.Configuration(syncConfiguration: SyncConfiguration(user: SyncUser.current!, realmURL: tasksRealmURL),objectTypes: [TaskList.self, Task.self])

}

```

Next, we'll add the definitions of our models.  Note that there are two kinds of models here: 1) the Task and taskList modes, and 2) a "Person" model.  This last model will allow us to store some meta data (first andm last name) about users of the app. It will be used only by this demo application and is instantiated in such a way the it can cleanly interoperate with other versions of the app that are not multi-user aware.


```swift

// MARK: Model

final class TaskList: Object {
    dynamic var text = ""
    dynamic var id = ""
    let items = List<Task>()

    override static func primaryKey() -> String? {
        return "id"
    }
}

final class Task: Object {
    dynamic var text = ""
    dynamic var completed = false
}
```

At this point, we've imported RealmSwift into the app and defined the data models (`Task` and `TaskList`) that we'll use to represent our data and sync with the RealmTasks apps.

Your app should still build and run.

## 4. Add a title and register a cell class for use with our table view

Add the following to the body of your `ViewController` class:

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    setupUI()
}

func setupUI() {
    title = "My Tasks"
    tableView.register(UITableViewCell.self, forCellReuseIdentifier: "cell")
}
```

## 5. Use a Realm List to display Tasks in the table view

Add the following property to your `ViewController` class, on a new line, right after the class declaration:

```swift
var items = List<Task>()
```

Add the following line to the end of your `viewDidLoad()` function to seed the list with some initial data:

```swift
override func viewDidLoad() {
    // ... existing function ...
    items.append(Task(value: ["text": "My First Task"]))
}
```

Append the following to the end of your `ViewController` class's body:

```swift
// MARK: UITableView

override func tableView(_ tableView: UITableView?, numberOfRowsInSection section: Int) -> Int {
    return items.count
}

override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "cell", for: indexPath)
    let item = items[indexPath.row]
    cell.textLabel?.text = item.text
    cell.textLabel?.alpha = item.completed ? 0.5 : 1
    return cell
}
```

If you then build and run the app, you'll see your one task being displayed in the table. There's also some code in here to show completed items as a little lighter than uncompleted items, but we won't see that in action until later.

## 6. Add support for creating new tasks

Delete the line in your `viewDidLoad()` function that seeded initial data:

```swift
override func viewDidLoad() {
    // -- DELETE THE FOLLOWING LINE --
    items.append(Task(value: ["text": "My First Task"]))
}
```

Add the following function to your `ViewController` class at its end:

```swift
// MARK: Functions

func add() {
    let alertController = UIAlertController(title: "New Task", message: "Enter Task Name", preferredStyle: .alert)
    var alertTextField: UITextField!
    alertController.addTextField { textField in
        alertTextField = textField
        textField.placeholder = "Task Name"
    }
    alertController.addAction(UIAlertAction(title: "Add", style: .default) { _ in
        guard let text = alertTextField.text , !text.isEmpty else { return }

        self.items.append(Task(value: ["text": text]))
        self.tableView.reloadData()
    })
    present(alertController, animated: true, completion: nil)
}
```

Now add the following line at the end of the `setupUI()` function:

```swift
func setupUI() {
    // ... existing function ...
    navigationItem.rightBarButtonItem = UIBarButtonItem(barButtonSystemItem: .add, target: self, action: #selector(add))
}
```

## 7. Back items by a Realm and integrate sync

Now, add the following properties to your `ViewController` class, just under the `items` property:

```swift
var notificationToken: NotificationToken!
var realm: Realm!
```

The notification token we just added will be needed when we start observing changes from the Realm.

---

Right after the end of the `setupUI()` function, add the following:

```swift
func setupRealm() {
    // Log in existing user with username and password
    let username = "test"  // <--- Update this
    let password = "test"  // <--- Update this
}

deinit {
    notificationToken.stop()
}
```

In the code above, enter the same values for the `username` and `password` variables as you used when registering a user through the RealmTasks app in the "Getting Started" steps.

Then insert the following at the end of the `setupRealm()` function (inside the function body):

```swift
func setupRealm() {
    // ... existing function ...
    SyncUser.logIn(with: .usernamePassword(username: username, password: password, register: false), server: URL(string: "http://127.0.0.1:9080")!) { user, error in
        guard let user = user else {
            fatalError(String(describing: error))
        }

        DispatchQueue.main.async {
            // Open Realm
            let configuration = Realm.Configuration(
                syncConfiguration: SyncConfiguration(user: user, realmURL: URL(string: "realm://127.0.0.1:9080/~/realmtasks")!)
            )
            self.realm = try! Realm(configuration: configuration)

            // Show initial tasks
            func updateList() {
                if self.items.realm == nil, let list = self.realm.objects(TaskList.self).first {
                    self.items = list.items
                }
                self.tableView.reloadData()
            }
            updateList()

            // Notify us when Realm changes
            self.notificationToken = self.realm.addNotificationBlock { _ in
                updateList()
            }
        }
    }
}
```

And, call this setup function at the end of the `viewDidLoad()` function:

```swift
override func viewDidLoad() {
    // ... existing function ...
    setupRealm()
}
```

Now edit the `add()` function to look like this:

```swift
func add() {
    let alertController = UIAlertController(title: "New Task", message: "Enter Task Name", preferredStyle: .alert)
    var alertTextField: UITextField!
    alertController.addTextField { textField in
        alertTextField = textField
        textField.placeholder = "Task Name"
    }
    alertController.addAction(UIAlertAction(title: "Add", style: .default) { _ in
        guard let text = alertTextField.text , !text.isEmpty else { return }

        let items = self.items
        try! items.realm?.write {
            items.insert(Task(value: ["text": text]), at: items.filter("completed = false").count)
        }
    })
    present(alertController, animated: true, completion: nil)
}
```

This deletes two lines in the `guard` block that begin with `self.` and replaces them with a `let` and `try!` block which will actually write a new task to the Realm.

Lastly, we need to allow non-TLS network requests to talk to our local sync server.

Right-click on the `Info.plist` file and select "Open as... Source Code" and paste the following in the `<dict>` section:

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

---

If you build and run the app now, it should connect to the object server and display the tasks that were added in RealmTasks earlier.

If you add new tasks by tapping the "Add" button in your app, you should immediately see them reflected in the RealmTasks app too.

**Congratulations, you've built your first synced Realm app!**

Keep going if you'd like to see how easy it is to add more functionality and finish building your task management app.

## 8. Support moving and deleting tasks

Add the following line to the end of your `setupUI()` function:

```swift
func setupUI() {
    // ... existing function ...
    navigationItem.leftBarButtonItem = editButtonItem
}
```

Now, add these functions to the `ViewController` class body, right after the other `tableView` functions:

```swift
override func tableView(_ tableView: UITableView, moveRowAt sourceIndexPath: IndexPath, to destinationIndexPath: IndexPath) {
    try! items.realm?.write {
        items.move(from: sourceIndexPath.row, to: destinationIndexPath.row)
    }
}

override func tableView(_ tableView: UITableView, commit editingStyle: UITableViewCellEditingStyle, forRowAt indexPath: IndexPath) {
    if editingStyle == .delete {
          try! realm.write {
              let item = items[indexPath.row]
              realm.delete(item)
          }
    }
}
```

## 9. Support toggling the 'completed' state of a task by tapping it

After the last `tableView` function in the `ViewController` class, add the following function override:

```swift
override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    let item = items[indexPath.row]
    try! item.realm?.write {
        item.completed = !item.completed
        let destinationIndexPath: IndexPath
        if item.completed {
            // move cell to bottom
            destinationIndexPath = IndexPath(row: items.count - 1, section: 0)
        } else {
            // move cell just above the first completed item
            let completedCount = items.filter("completed = true").count
            destinationIndexPath = IndexPath(row: items.count - completedCount - 1, section: 0)
        }
        items.move(from: indexPath.row, to: destinationIndexPath.row)
    }
}
```

## 10. You're done!
