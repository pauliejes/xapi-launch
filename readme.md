# xAPI-Launch

##### The xAPI-Launch server is a demonstration of the xAPI Launch algorithm. 
xAPI-Launch allows a user to initiate an interaction with some xAPI enabled learning experience. The content, be it an online module, a static HTML file, or an immersive simulation, need not know ahead of time the identity of the learner, the LRS to which the learning data should be submitted, nor the "session" into which the events should be grouped. Content need only implement a minimal HTTP request to become "xAPI-Launch" enabled. 

xAPI Launch exists primarily to enable a learner to track experiences from any learning resource without some out-of-band method to add LRS credentials to the content, and without asking the user to input these credentials into some untrused third-party system. It also ensures that statements which claim to be part of an experience really came from that experience. xAPI Launch differs from other launch algorithms in a few important ways. First, it avoids placing personally indentifying information (PII) in a query string. This helps protect the learner's identity. We also avoid initiating the launch session from an HTTP POST request. The body of a post request is not visible when using a URL to launch a native application. In order to avoid both PII in the query string and an initial POST, we generate a one time use random token that can be exchanged for the name and LRS endpoints of the learner, but can be safely transfered as part of the URL.

#### We support the following user cases
* Launch content that is a **static HTML resource**
* Launch content that is dynamic **HTML content rendered by a server**
* Launch a **desktop application** through a URL Protocol Handler
* Launch a **mobile application** through a URL Protocol Handler
* Launch a **disconnected experience** by allowing the student to manually enter a code

#### The xAPI-Launch algorithm
The launch algorithm is designed to allow the maximum flexibility while still maintaining privacy for the learner, a guarantee that the statements are submitted by the proper content, and that all statements can be grouped by the LMS according to the Launch Attempt

1. The student requests a launch of a given piece of content
2. The launch server generates a unique ID for this launch attempt. The unique ID is called the Launch Token
3. The token is saved along with the associated content record and associated learner identity
4. **Optional:** The launch token is encrypted with the content's registered public key
4. The student delivers the (possibly encrypted) token and the Launch Service address to the content (often as query string parameters in an HTTP Get request)
5. **Optional:** The content decrypts the Launch Token
5. The content issues an HTTP Post request containing the Launch Token to the Launch Service address
6. The Launch Server verifies that the submitted Launch Token
   * Is a valid token
   * Is in the uninitialized state
   * Has not timed out
1. The Launch Server marks the launch as *initialized*, and returns to the content as the response to the Post request
   * The xAPI Actor that should be the subject of the each statement
   * The address of a temporary xAPI endpoint to which the content should submit statements
   * The **Context Activities** that provide the context for the launch event. These activities will include the Launch Token and the Content URL
1.  The Launch Server will set a session cookie as part of the response to the POST request. All incoming XAPI statements must include this cookie.
2.  The Launch Server will enforce that each incoming statement associated with the given launch contains at minimum the Context Activities for the Launch and the Content URL.

#### Static HTML course content
In the case of a static HTML file, the content should use JavaScript to read the query string and post to the Launch Service address. The cookie will be set automatically, and handled natively by the browser. The content may choose to decrypt the Launch Token, but note that this is seldom secure enough to leave to the client. Most often the information should pass as plain text. Content should be delivered via TLS or SSL.

#### Dynamic HTML rendered by a server
The client will deliver to the content server the Launch Token and the Launch Service address via the request query string. This server should establish session for the user, and initiate the xAPI Launch on their behalf. The method that the content server uses to gather performance information from the client's browser is out of scope, but could use xAPI as well. The server should initiate the launch on the user's behalf, and keep the session cookie returned in step **10** above hidden from the learner. The server may choose to persist this information or forward it to other parties. If possible, a public key should be provided to the Launch Server at the time the content is enlisted. If this information is available, then the Launch Token will be delivered to the learner in an encrypted form. The content server should decrypt this information before initializing the launch. Because the learner cannot access the plaintext of the token, the Launch Server can be confident that the incoming xAPI data did in fact originate with the proper content server. 

The content server may optionally terminate the launch attempt immediately. Any business logic is valid for the server to make this decision. We intend that the content server may check a given number of registrations, check the provided actor against a list of authorized users, or initiate some sort of payment process with the learner. The content should use the Launch Service address to identify the Launch Server. 

#### Offline systems
The learner may optionally move the Launch Token to the content manually. We imagine that sit-down simulators or desktop applications might prompt the user to enter their xAPI Launch Token. Other than the method used to gather this information from the user, this sort of non-HTML content should behave in the same manor as the content server described above. An offline system could optionally store the launch token and the interactions to be posted later.


## Running the server
1. Clone the repo (https://github.com/adlnet/xapi-launch.git)
2. npm install
3. node app.js
4. enter LRS credentials

## Running the demo content
1. Create a user account
2. Find "Register New Content"
3. Use "http://localhost:3000/static/staticContentDemo/demo.html" as the URL
4. Do not enter a private key.
5. Browse all content
6. Click "Launch"

## API

1. `POST /launch/:key`
    * Post to this endpoint to trade a key for Actor information, and initialize the Launch Session. Key is the Launch Token
    * Returns:
      
      ```
      status:200,
      body:{
        actor:...
        endpoint...
        contextActivities...
      }
      ```
    * Errors
      * "invalid launch key" - the provided key is unknown to the launch server
      * "The launch token has already been used" - this launch token has already been consumed, and you are not the registered consumer
      * "launch initialization timeout" - it has taken too long for the launch token to be used. The launch terminates without ever being initialized
1. `POST /launch/:key/xAPI/statements`
    * Post an XAPI statement to be associated with the Launch. You must first initialize the Launch Session before you can POST. This       endpoint obeys the XAPI interface.
    * In addition to the xAPI error codes, you may get additional errors with a `Status:500`
      * "This launch was closed automatically..." - the launch session has timed out 
      * "invalid launch key" - the key you provided is not known to the server
      * "Launch is not in the open state" - the launch session is not in a state where it can accept statements. It could be closed or uninitialized
      * "You are not the registered consumer for this launch" - The submission did not include the established launch consumer session cookie
      * "The content associated with this launch has been removed" - The content associated with this launch has been removed from the Launcher database
1. `POST /launch/:key/terminate`
    * Terminate the session. The Launch Session token will be invalidated and can no longer be used.
    * Errors
      * All the same errors as the XAPI endpoint as well as 
      * "key is locked to prevent double launch" - you already have a request to close this session in progress
1. URL Query String form for Static Content and Dynamic HTML. Data is URL encoded
  * `xAPILaunchKey` - the Launch Token to use in the above API
  * `xAPILaunchService` - the fully qualified URL of the above API root
  * Example:
    
    ```
    http://www.myCourseWare.com/course1?xAPILaunchService=http%3A%2F%2Flaunch.adlnet.gov%2F&xAPILaunchKey=892cdfbe-1549-4554-b321-96a0562f4eb5
    ```


### Using the xAPI-Launch library
The xAPI-Launch sample implementation includes a client side JavaScript library that you can use to make your content xAPI-Launch enabled. 

1.  Make sure that your project already contains the [xAPIWrapper JavaScript file](https://github.com/adlnet/xAPIWrapper)
2.  Include the [xAPI-launch-client.js](https://github.com/adlnet/xapi-launch/blob/master/public/xAPI-launch-client.js) file from this repository with a script tag.
3.  Call the global function `xAPILaunch(callback, terminate_on_unload)`
    1.  `callback` should be a function with the form `function(err,launchdata,xAPIWrapper)`
    2.  If the algorithm succeeds, launchdata will contain information about the user, and xAPIWrapper will be an instance of the xAPIWRapper tool configured to talk to the appropriate LRS endpoint
    3.  Set `terminate_on_unload` to automatically terminate the launch session when the user leaves the page
4.  If the algorithm fails (because of a misconfiguration or an invalid launch) you can still setup the xAPIWrapper manually, and use it as if Launch has never occured. (however you did it before.)
5.  You can link together multiple static pages by including the `courselink="true"` attribute on HTML links. The library will edit the page to include the launch token and endpoint in the address of these links. If you are navigating the browser via JavaScript, and you want the next page in a set of static pages to be included in the launch, you will need to manually append the launch token and endpoint to the next address.

##License
[Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0.txt)
