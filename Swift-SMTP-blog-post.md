![swift-smtp](https://developer.ibm.com/swift/wp-content/uploads/sites/69/2017/05/swift-smtp-bird.png)

In your application, there may be times where you want to send emails--a user requests a password change, alerting subscribed users that an item has been restocked, sending a newsletter, etc. Introducing **[Swift-SMTP](https://github.com/IBM-Swift/Swift-SMTP)**, a Swift package that can be used for just that. It supports SSL connections, as well as local file, HTML, and data attachments. In this blog post we'll walk through a simple web server that uses **Swift-SMTP** to send emails. Let's get started!

## Getting Started

- Clone the demo project: `git clone https://github.com/quanvo87/Swift-SMTP-Demo`

- `cd` into the new directory, and run `swift package generate-xcodeproj` (this will download dependencies and take a few minutes)

- Open the newly created project: `Swift-SMTP-Demo.xcodeproj`

- Change the scheme to the second one (this has an executable), and hit `⌘+r` to start the server:

![change xcode scheme](https://developer.ibm.com/swift/wp-content/uploads/sites/69/2017/05/Screen-Shot-2017-05-30-at-4.13.30-PM.png)

- Open a browser to `localhost:8080`. You should see the page below:

![demo homepage](https://developer.ibm.com/swift/wp-content/uploads/sites/69/2017/05/swift-smtp-demo-homepage.png)

Before this can work, we need to set up our credentials on the server side. Let's go back to the Xcode project.

## main.swift

Back in the Xcode project, everything happens in `main.swift`. It can be broken into two sections, `Swift-SMTP` and `Kitura`. We'll talk about `Swift-SMTP` first:

### Swift-SMTP

```swift
// The SMTP server you are going to connect to
let hostname = ""

// The email you will be sending from
let email = ""

// The password to the email
let password = ""
```
First we'll set some basic login info we'll need. The `hostname` will be something like `smtp.gmail.com`. A quick Google search should reveal any major SMTP server hostnames. We'll also need our login and password for the server.

```swift
// Init an SSL instance depending on OS
#if os(Linux)
let cert = dir + "/cert.pem"
let key = dir + "/key.pem"
let ssl = SSL(withCACertificateDirectory: nil,
              usingCertificateFile: cert,
              withKeyFile: key)
#else
let cert = dir + "/cert.pfx"
let certPassword = "kitura"
let ssl = SSL(withChainFilePath: cert,
              withPassword: certPassword)
#endif
```
Many email servers require you to connect securely via TLS/SSL, like Gmail. So here we create an `SSL` configuration that we'll use to connect to our SMTP server. Because of current discrepencies between the native security libraries available on macOS and Linux, we must use different methods depending on the OS we are working on. This demo uses self-signed certificates--this [blog post](https://developer.ibm.com/swift/2016/09/22/securing-kitura-part-1-enabling-ssltls-on-your-swift-server/) details how I generated them.

```swift
// The handle to the SMTP server with your login info
let smtp = SMTP(hostname: hostname,
                email: email,
                password: password,
                ssl: ssl)
```
Finally we are able to create our `SMTP` handle. Simply plug in the login info we just created. This is how we'll securely login to our SMTP server and send out emails!

>_If you know your server does not require an SSL connection, you can omit the `ssl` parameter._

>_If you get an error later on in the demo, it could be because **Swift-SMTP** does not work with your SMTP server with default settings. Refer to **Swift-SMTP**'s [documentation](https://ibm-swift.github.io/Swift-SMTP/) for further customization options, such as choosing what port to connect on, or what authentication methods to use. You may also have to enable third party access to your SMTP server._

```swift
// The `User` object that will act as our sender
let sender = User(name: "Swift-SMTP Demo",
                  email: email)
```
One last thing we want to do is create a sender `User`. More on this in the next section.

### Kitura

This demo server is built on the [Kitura](https://github.com/IBM-Swift/Kitura) Swift web framework. Let's jump right in.

```swift
let router = Router()
```
We first create our router.

```swift
// Serve our static index.html
router.all("/", middleware: StaticFileServer())
```
Here we serve our `index.html` at `/`. We'll go over `index.html` in the next section.

```swift
// Send an email on this route
router.get("/send/:email") { req, res, _ in

    // Get the email from the parameters (1)
    let email = req.parameters["email"] ?? ""

    // Create a `User` object that will act as our receiver (2)
    let receiver = User(email: email)

    // Create the `Mail` object that will be sent (3)
    let mail = Mail(from: sender,
                    to: [receiver],
                    subject: "Swift-SMTP Test Email",
                    text: "Hello, world!")

    // Send the mail object (4)
    smtp.send(mail, completion: { (error) in
        var message: String

        // Set the appropriate response message (5)
        if let error = error {
            message = "Send failed: " + String(describing: error)
        } else {
            message = "Email successfully sent! You should see it in your sent box."
        }

        // Send the response (6)
        do {
            try res.send(message).end()
        } catch {
            Log.error("\(error)")
        }
    })
}
```
Where the magic happens. Upon hitting the `/send` route, this code will be executed:

1. Pull the email address out of the URL parameters
2. Create our receiver `User` with this email address
3. Create the actual `Mail` object with the appropriate fields
4. Use this `Mail` object to call `send(_:completion:)` on our `smtp` handle--an asynchronous call
5. Once the function is complete, we check for any errors
6. Then send the appropriate response back to the client

## index.html

Our server is now ready to serve content and send emails. The last piece is our static home page, located in `/public`:

```html
...
  function send() {
	...
    var email = document.getElementById("emailTextBox").value;
    ...
    xhttp.open("GET", "/send/".concat(email), true);
    xhttp.send();
    }
...
```
These are the most important snippets of `index.html`. This is the function that gets called when a user clicks `Send`. Notice we build our URL to include the user's input email.

## Send The Email!

Restart your server now that the proper credentials are in. Refresh `localhost:8080` and try entering an email and clicking `Send`. I recommend sending to yourself, or a public inbox like [Mailinator](https://www.mailinator.com/), so you can confirm that everything worked.

>_Some SMTP servers don't display mails sent like this in the sent box_.

### Debugging

**Swift-SMTP** logs much of its work. This demo utilizes [HeliumLogger](https://github.com/IBM-Swift/HeliumLogger), so you can see this activity in the Xcode console:

![swift-smtp-demo console](https://developer.ibm.com/swift/wp-content/uploads/sites/69/2017/05/swift-smtp-demo-console.png)

## Conclusion

**[Swift-SMTP](https://github.com/IBM-Swift/Swift-SMTP)** is a Swift package that can be used to send emails to an SMTP server. This [demo](https://github.com/quanvo87/Swift-SMTP-Demo) went over how to incorporate it in an application built on the [Kitura](https://github.com/IBM-Swift/Kitura) framework and send a simple email through a secure SSL connection. You can refer to **Swift-SMTP**'s [documentation](https://ibm-swift.github.io/Swift-SMTP/) to learn about the different configuration options available as well as add attachments to your emails.

We are always looking for ways to improve **Swift-SMTP**. If you have any questions or requests, head over to the [repo](https://github.com/IBM-Swift/Swift-SMTP) or our [Slack team](http://swift-at-ibm-slack.mybluemix.net) and let us know.

<script async defer src="https://swift-at-ibm-slack.mybluemix.net/slackin.js?large"></script> 
<a class="github-button" href="https://github.com/IBM-Swift/Kitura" data-style="mega" data-count-href="/IBM-Swift/Kitura/stargazers" data-count-api="/repos/IBM-Swift/Kitura#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star IBM-Swift/Kitura on GitHub">Star</a>
<script async defer src="https://buttons.github.io/buttons.js"></script>